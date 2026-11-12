# 05 — Queries & Data Manipulation

## SELECT
```sql
[WITH [RECURSIVE] cte AS [NOT] [MATERIALIZED] (query) [, ...]]
SELECT [ALL | DISTINCT | DISTINCT ON (expr)] expr [AS alias], ...
FROM t [AS a] [TABLESAMPLE BERNOULLI|SYSTEM (10)] [LATERAL (subquery)]
  [INNER] JOIN t2 ON cond | USING (col)
  LEFT|RIGHT|FULL [OUTER] JOIN t2 ON cond
  CROSS JOIN t2 | NATURAL JOIN t2
WHERE cond
GROUP BY col | ROLLUP (a,b) | CUBE (a,b) | GROUPING SETS ((a),(b),())
HAVING cond
WINDOW w AS (PARTITION BY a ORDER BY b)
UNION|INTERSECT|EXCEPT [ALL] SELECT ...
ORDER BY col [ASC|DESC] [NULLS FIRST|LAST]
LIMIT n OFFSET m                         -- or: OFFSET m ROWS FETCH FIRST n ROWS ONLY | WITH TIES
FOR UPDATE|NO KEY UPDATE|SHARE|KEY SHARE [OF t] [NOWAIT|SKIP LOCKED];
```

### Frequently used forms
```sql
SELECT * FROM t;
SELECT count(*), sum(x), avg(x), min(x), max(x) FROM t GROUP BY g HAVING count(*) > 1;
SELECT DISTINCT ON (user_id) * FROM e ORDER BY user_id, created DESC;
SELECT row_number() OVER (PARTITION BY g ORDER BY d), rank() OVER w, lag(x) OVER w FROM t WINDOW w AS (PARTITION BY g ORDER BY d);
SELECT sum(x) OVER (ORDER BY d ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) FROM t;
SELECT a FROM t WHERE x IN (SELECT ...) AND EXISTS (SELECT 1 FROM u WHERE u.id = t.id);
SELECT a FROM t WHERE x = ANY (ARRAY[1,2,3]);
SELECT CASE WHEN x > 0 THEN 'pos' ELSE 'neg' END, COALESCE(a, 0), NULLIF(a, 0), GREATEST(a,b), LEAST(a,b);
SELECT x::int, CAST(x AS int), x::text;
SELECT * FROM generate_series(1, 10);
SELECT * FROM unnest(ARRAY[1,2,3]) WITH ORDINALITY;
SELECT string_agg(name, ', ' ORDER BY name), array_agg(id), jsonb_agg(t), jsonb_object_agg(k, v) FROM t;
SELECT count(*) FILTER (WHERE x > 1) FROM t;
SELECT * FROM t WHERE name ILIKE '%abc%' OR name ~* '^a.*z$' OR name SIMILAR TO '%(a|b)%';
SELECT data->'a'->>'b', data #>> '{a,b}', data @> '{"k":1}', data ? 'k' FROM t;
SELECT to_tsvector('english', body) @@ to_tsquery('english', 'foo & bar') FROM t;
WITH RECURSIVE r(n) AS (SELECT 1 UNION ALL SELECT n+1 FROM r WHERE n < 10) SELECT * FROM r;
TABLE t;                                  -- = SELECT * FROM t
```

## VALUES
```sql
VALUES (1,'a'), (2,'b');
SELECT * FROM (VALUES (1,'a'), (2,'b')) AS v(id, name);
```

## INSERT
```sql
INSERT INTO t (a, b) VALUES (1, 'x'), (2, 'y');
INSERT INTO t DEFAULT VALUES;
INSERT INTO t (a, b) SELECT a, b FROM src;
INSERT INTO t (...) OVERRIDING SYSTEM VALUE VALUES (...);
INSERT INTO t (a, b) VALUES (1, 'x') RETURNING *;
INSERT INTO t (id, v) VALUES (1, 'x')
  ON CONFLICT (id) DO UPDATE SET v = EXCLUDED.v, updated = now();   -- upsert
INSERT INTO t (id, v) VALUES (1, 'x') ON CONFLICT DO NOTHING;
INSERT INTO t ... ON CONFLICT ON CONSTRAINT uq DO NOTHING;
WITH ins AS (INSERT INTO t ... RETURNING id) INSERT INTO u SELECT id FROM ins;
```

## UPDATE
```sql
UPDATE t SET a = 1, b = b + 1 WHERE id = 5;
UPDATE t SET (a, b) = (1, 2) WHERE id = 5;
UPDATE t SET a = s.a FROM src s WHERE t.id = s.id;
UPDATE t SET a = 1 WHERE CURRENT OF cur;
UPDATE t SET a = 1 WHERE id = 5 RETURNING *;
UPDATE ONLY t SET ...;
UPDATE t SET a = DEFAULT;
```

## DELETE
```sql
DELETE FROM t WHERE id = 5;
DELETE FROM t USING u WHERE t.uid = u.id AND u.bad;
DELETE FROM t WHERE id = 5 RETURNING *;
DELETE FROM ONLY t;
DELETE FROM t WHERE CURRENT OF cur;
```

## MERGE
```sql
MERGE INTO target t USING source s ON t.id = s.id
WHEN MATCHED AND s.del THEN DELETE
WHEN MATCHED THEN UPDATE SET v = s.v
WHEN NOT MATCHED THEN INSERT (id, v) VALUES (s.id, s.v)
WHEN NOT MATCHED BY SOURCE THEN DELETE        -- 17+
RETURNING merge_action(), *;                  -- 17+
```

## RETURNING OLD / NEW (18+)
```sql
UPDATE t SET v = v + 1 WHERE id = 1 RETURNING old.v AS before, new.v AS after;
```

## TRUNCATE
```sql
TRUNCATE t;
TRUNCATE t1, t2 RESTART IDENTITY CASCADE;
TRUNCATE ONLY t CONTINUE IDENTITY RESTRICT;
```

## COPY
```sql
COPY t [(cols)] FROM '/path/file.csv' WITH (FORMAT csv, HEADER true, DELIMITER ',', NULL '', QUOTE '"', ENCODING 'UTF8');
COPY t TO '/path/file.csv' WITH (FORMAT csv, HEADER);
COPY (SELECT ...) TO STDOUT WITH (FORMAT csv, HEADER);
COPY t FROM STDIN;
COPY t FROM PROGRAM 'gunzip -c /f.gz';
COPY t TO PROGRAM 'gzip > /f.gz';
COPY t FROM '/f.csv' WITH (FORMAT csv, FORCE_NULL (a), FORCE_NOT_NULL (b), ON_ERROR ignore, LOG_VERBOSITY verbose);
COPY t FROM '/f.bin' WITH (FORMAT binary);
```
Client-side (psql): `\copy t FROM 'file.csv' CSV HEADER`
