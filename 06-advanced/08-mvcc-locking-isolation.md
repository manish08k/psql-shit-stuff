# 08 — MVCC, Isolation, Locking, Concurrency Patterns

## Isolation levels
| Level | Dirty read | Non-repeatable | Phantom | Serialization anomaly |
|---|---|---|---|---|
| READ UNCOMMITTED (= READ COMMITTED in PG) | no | possible | possible | possible |
| **READ COMMITTED** (default) | no | possible | possible | possible |
| REPEATABLE READ (snapshot) | no | no | no | possible |
| SERIALIZABLE (SSI) | no | no | no | no (may raise 40001) |
```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SET default_transaction_isolation = 'serializable';
-- retry on SQLSTATE 40001 (serialization_failure) and 40P01 (deadlock_detected)
```

## Row locks
```sql
SELECT * FROM t WHERE id = 1 FOR UPDATE;                 -- exclusive row lock
SELECT * FROM t WHERE id = 1 FOR NO KEY UPDATE;          -- allows FK inserts
SELECT * FROM t WHERE id = 1 FOR SHARE;
SELECT * FROM t WHERE id = 1 FOR KEY SHARE;
... NOWAIT;      ... SKIP LOCKED;
-- job queue
SELECT * FROM jobs WHERE status='new' ORDER BY id LIMIT 1 FOR UPDATE SKIP LOCKED;
WITH j AS (SELECT id FROM jobs WHERE status='new' ORDER BY id LIMIT 10 FOR UPDATE SKIP LOCKED)
UPDATE jobs SET status='run' FROM j WHERE jobs.id = j.id RETURNING jobs.*;
```

## Table lock modes (conflicts rise toward ACCESS EXCLUSIVE)
```
ACCESS SHARE            ← SELECT
ROW SHARE               ← SELECT FOR UPDATE/SHARE
ROW EXCLUSIVE           ← INSERT/UPDATE/DELETE/MERGE
SHARE UPDATE EXCLUSIVE  ← VACUUM, ANALYZE, CREATE INDEX CONCURRENTLY, ALTER TABLE ... VALIDATE
SHARE                   ← CREATE INDEX
SHARE ROW EXCLUSIVE     ← CREATE TRIGGER, some ALTER TABLE
EXCLUSIVE               ← REFRESH MATVIEW CONCURRENTLY
ACCESS EXCLUSIVE        ← DROP, TRUNCATE, VACUUM FULL, most ALTER TABLE, LOCK default
```

## Advisory locks (app-level)
```sql
SELECT pg_advisory_lock(42);              SELECT pg_advisory_unlock(42);
SELECT pg_try_advisory_lock(42);          -- non-blocking
SELECT pg_advisory_xact_lock(42);         -- auto release at tx end
SELECT pg_advisory_xact_lock(hashtext('job:abc'));
SELECT pg_advisory_lock_shared(42);
SELECT * FROM pg_locks WHERE locktype='advisory';
```

## Concurrency-safe patterns
```sql
-- atomic upsert
INSERT INTO t (k, v) VALUES ($1, $2) ON CONFLICT (k) DO UPDATE SET v = EXCLUDED.v;
-- optimistic locking
UPDATE t SET v = $1, version = version + 1 WHERE id = $2 AND version = $3;      -- check rowcount
-- atomic counter
UPDATE counters SET n = n + 1 WHERE id = 1 RETURNING n;
-- get-or-create
WITH ins AS (INSERT INTO t (k) VALUES ($1) ON CONFLICT (k) DO NOTHING RETURNING *)
SELECT * FROM ins UNION ALL SELECT * FROM t WHERE k = $1 LIMIT 1;
-- transfer (consistent lock order to avoid deadlock)
SELECT * FROM acct WHERE id IN (1,2) ORDER BY id FOR UPDATE;
-- timeouts
SET lock_timeout = '2s';   SET statement_timeout = '30s';   SET idle_in_transaction_session_timeout = '60s';   SET deadlock_timeout = '1s';
```

## Observe blocking
```sql
SELECT a.pid, a.query, pg_blocking_pids(a.pid) AS blocked_by FROM pg_stat_activity a WHERE cardinality(pg_blocking_pids(a.pid)) > 0;
SELECT * FROM pg_locks WHERE NOT granted;
SELECT pg_terminate_backend(<blocker_pid>);
SET log_lock_waits = on;
```

## MVCC internals / bloat
```sql
SELECT xmin, xmax, ctid, * FROM t LIMIT 5;
SELECT txid_current(), pg_current_xact_id(), pg_snapshot_xmin(pg_current_snapshot());
SELECT pg_export_snapshot();                         -- share a snapshot across sessions
-- HOT updates need fillfactor headroom
ALTER TABLE t SET (fillfactor = 85);
SELECT n_tup_upd, n_tup_hot_upd FROM pg_stat_user_tables WHERE relname='t';
-- dead tuple removal
VACUUM (VERBOSE) t;       -- plain: reuse space;  VACUUM FULL: rewrite (exclusive lock)
-- online repack: pg_repack extension  →  pg_repack -d db -t t
```
