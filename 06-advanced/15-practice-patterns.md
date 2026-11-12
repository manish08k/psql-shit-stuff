# 15 — Practice Patterns (interview + real-world)

## Analytics
```sql
-- 2nd highest salary
SELECT DISTINCT salary FROM emp ORDER BY salary DESC OFFSET 1 LIMIT 1;
SELECT salary FROM (SELECT salary, dense_rank() OVER (ORDER BY salary DESC) r FROM emp) s WHERE r = 2 LIMIT 1;

-- Nth per group
SELECT * FROM (SELECT *, row_number() OVER (PARTITION BY dept ORDER BY salary DESC) rn FROM emp) s WHERE rn = 2;

-- Running total / moving avg / YoY
SELECT d, sum(v) OVER (ORDER BY d) FROM s;
SELECT d, avg(v) OVER (ORDER BY d ROWS 6 PRECEDING) FROM s;
SELECT yr, v, v - lag(v) OVER (ORDER BY yr) AS yoy FROM yearly;

-- Cumulative distinct (first-seen)
SELECT d, count(*) FILTER (WHERE rn = 1) OVER (ORDER BY d) AS cum_users FROM (SELECT d, user_id, row_number() OVER (PARTITION BY user_id ORDER BY d) rn FROM ev) s;

-- Retention / cohort
SELECT date_trunc('month', first_seen) AS cohort, count(*) FROM (SELECT user_id, min(ts) first_seen FROM ev GROUP BY 1) s GROUP BY 1;

-- Median
SELECT percentile_cont(0.5) WITHIN GROUP (ORDER BY v) FROM t;

-- Percent of total per group
SELECT g, v, round(100.0 * v / sum(v) OVER (PARTITION BY g), 2) FROM t;

-- Pivot
SELECT id, sum(v) FILTER (WHERE m = 'jan') jan, sum(v) FILTER (WHERE m = 'feb') feb FROM t GROUP BY id;

-- Unpivot
SELECT id, k, v FROM t CROSS JOIN LATERAL (VALUES ('a', a), ('b', b)) x(k, v);

-- Missing dates (fill gaps)
SELECT d::date, coalesce(sum(t.v), 0) FROM generate_series('2026-01-01'::date, '2026-01-31', '1 day') d LEFT JOIN t ON t.dt = d::date GROUP BY 1 ORDER BY 1;

-- Consecutive streaks
SELECT user_id, min(d), max(d), count(*) FROM (SELECT user_id, d, d - (row_number() OVER (PARTITION BY user_id ORDER BY d))::int AS g FROM logins) s GROUP BY user_id, g HAVING count(*) >= 3;

-- Overlapping intervals
SELECT a.id, b.id FROM bookings a JOIN bookings b ON a.id < b.id AND a.room = b.room AND tstzrange(a.s, a.e) && tstzrange(b.s, b.e);

-- Hierarchy rollup
WITH RECURSIVE h AS (SELECT id, parent_id, amt FROM node UNION ALL SELECT n.id, n.parent_id, h.amt FROM node n JOIN h ON h.parent_id = n.id) SELECT id, sum(amt) FROM h GROUP BY id;
```

## Data cleaning
```sql
-- find duplicates
SELECT email, count(*) FROM u GROUP BY email HAVING count(*) > 1;
-- delete duplicates keep lowest id
DELETE FROM u a USING u b WHERE a.email = b.email AND a.id > b.id;
-- trim + normalize
UPDATE u SET email = lower(trim(email));
-- null-safe compare
WHERE a IS NOT DISTINCT FROM b
-- replace empty strings
UPDATE t SET c = NULLIF(trim(c), '');
-- bulk update from CSV staging
UPDATE t SET v = s.v FROM stage s WHERE t.id = s.id AND t.v IS DISTINCT FROM s.v;
-- insert missing only
INSERT INTO t SELECT s.* FROM stage s WHERE NOT EXISTS (SELECT 1 FROM t WHERE t.id = s.id);
-- sync (upsert + delete)
MERGE INTO t USING stage s ON t.id = s.id WHEN MATCHED THEN UPDATE SET v = s.v WHEN NOT MATCHED THEN INSERT VALUES (s.id, s.v) WHEN NOT MATCHED BY SOURCE THEN DELETE;
```

## Schema evolution (safe)
```sql
ALTER TABLE t ADD COLUMN c int;                                                  -- instant
ALTER TABLE t ADD COLUMN c int NOT NULL DEFAULT 0;                               -- instant (11+)
-- add NOT NULL on existing col without long lock:
ALTER TABLE t ADD CONSTRAINT c_nn CHECK (c IS NOT NULL) NOT VALID;
ALTER TABLE t VALIDATE CONSTRAINT c_nn;
ALTER TABLE t ALTER COLUMN c SET NOT NULL;                                       -- fast once valid CHECK exists
ALTER TABLE t DROP CONSTRAINT c_nn;
-- change type w/o rewrite when compatible (e.g. varchar(n) → text), else:
ALTER TABLE t ADD COLUMN c_new bigint;  -- backfill in batches → swap → drop old
-- rename safely: add new + dual-write + backfill + switch + drop
-- always: SET lock_timeout = '3s'; retry on failure
```

## Backfill in batches
```sql
DO $$
DECLARE n int;
BEGIN
  LOOP
    UPDATE t SET c_new = c WHERE id IN (SELECT id FROM t WHERE c_new IS NULL LIMIT 10000);
    GET DIAGNOSTICS n = ROW_COUNT;
    EXIT WHEN n = 0;
    COMMIT;
  END LOOP;
END $$;
```

## Soft delete + unique
```sql
CREATE UNIQUE INDEX ON t (email) WHERE deleted_at IS NULL;
```

## Audit / history
```sql
CREATE TABLE t_hist (LIKE t INCLUDING DEFAULTS, valid tstzrange, op text);
-- or trigger → audit_log(jsonb) (see 09)
```

## Queue
```sql
SELECT id FROM jobs WHERE status='new' ORDER BY id FOR UPDATE SKIP LOCKED LIMIT 1;
LISTEN new_job;   NOTIFY new_job, 'id';
```

## Idempotent keys / dedupe on insert
```sql
INSERT INTO events (idem_key, payload) VALUES ($1, $2) ON CONFLICT (idem_key) DO NOTHING RETURNING id;
```

## Pagination
```sql
-- keyset (fast)
SELECT * FROM t WHERE (created_at, id) < ($1, $2) ORDER BY created_at DESC, id DESC LIMIT 50;
```

## Full DB documentation dump
```sql
SELECT table_schema, table_name, column_name, data_type, is_nullable, column_default FROM information_schema.columns WHERE table_schema NOT IN ('pg_catalog','information_schema') ORDER BY 1,2,ordinal_position;
```
```bash
pg_dump -s db > schema.sql
```

## Learn-path (suggested order)
0. 01-psql/ first (00-quick-card → 01 → 03 → 04 → 05 → 06); you will use psql for everything below
1. 02-sql/05 (SELECT/DML) → 06 (roles) → 02/03/04 (DDL)
2. 06-advanced/01 (windows) → 02 (CTE/joins) → 03 (functions) → 04 (JSON/arrays)
3. 06-advanced/07 (indexes) → 08 (MVCC/locks) → 10 (tuning)
4. 06-advanced/06 (partitioning/rollover) → 09 (PL/pgSQL)
5. 01-psql/07-10 + 03/04 apps → 06-advanced/11 (HA/backup) → 12 (security)
6. 05-cheatsheets for daily use
