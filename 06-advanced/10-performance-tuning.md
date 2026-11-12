# 10 — Performance Tuning

## Workflow
1. Find slow queries (`pg_stat_statements`, `log_min_duration_statement`)
2. `EXPLAIN (ANALYZE, BUFFERS)` the worst
3. Fix: index / rewrite / stats / config
4. Re-measure

## pg_stat_statements
```
shared_preload_libraries = 'pg_stat_statements'      # restart
pg_stat_statements.track = all
```
```sql
CREATE EXTENSION pg_stat_statements;
SELECT calls, round(total_exec_time::numeric,1) AS total_ms, round(mean_exec_time::numeric,2) AS mean_ms, rows, shared_blks_read, left(query,80)
FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 20;
SELECT ... ORDER BY mean_exec_time DESC;      -- slowest per call
SELECT ... ORDER BY calls DESC;                -- most frequent
SELECT ... ORDER BY shared_blks_read DESC;     -- I/O heavy
SELECT pg_stat_statements_reset();
```

## Reading EXPLAIN
```
Seq Scan           full table read                      → add index / check selectivity
Index Scan         index + heap fetch
Index Only Scan    no heap (needs visibility map)       → VACUUM
Bitmap Heap/Index  many matches via bitmap
Nested Loop        good for small outer / indexed inner
Hash Join          big unsorted sets, equality
Merge Join         pre-sorted inputs
Sort / Incremental Sort / HashAggregate / GroupAggregate
Gather / Gather Merge  parallel
Look for: rows estimated vs actual (bad stats), "Rows Removed by Filter", "Heap Fetches", loops × time, "Sort Method: external merge Disk" (raise work_mem), "Buffers: read" (cache misses)
```

## Query rewrite tips
```sql
-- avoid function on indexed column
WHERE created_at >= '2026-01-01' AND created_at < '2026-02-01'        -- not date_trunc/extract on column
-- avoid leading wildcard w/o trigram index
-- NOT IN → NOT EXISTS
-- OFFSET large → keyset pagination (WHERE id > $last ORDER BY id LIMIT n)
-- SELECT only needed columns (index-only scans)
-- COUNT(*) on huge tables → estimate: SELECT reltuples::bigint FROM pg_class WHERE relname='t';
-- batch: INSERT ... VALUES (..),(..),(..)  or COPY  instead of row-by-row
-- UNION ALL instead of UNION when dupes impossible
-- EXISTS instead of IN (subquery) / COUNT(*) > 0
-- avoid SELECT DISTINCT as a bug-fix for bad joins
-- parameterized/prepared statements for repeated queries
```

## Bulk load fast
```sql
COPY t FROM '/f.csv' CSV;
-- or: drop indexes/FKs → load → recreate; ALTER TABLE t SET UNLOGGED; (then SET LOGGED)
SET maintenance_work_mem = '2GB';   SET synchronous_commit = off;   SET max_wal_size = '16GB';
ANALYZE t;
```

## Key memory/CPU settings (starting points for dedicated server, RAM = R)
```
shared_buffers            = 25% R
effective_cache_size      = 50–75% R
work_mem                  = 16–64MB   (per sort/hash node per query!)
maintenance_work_mem      = 512MB–2GB
wal_buffers               = 16MB
max_wal_size              = 4–16GB
checkpoint_timeout        = 15min
checkpoint_completion_target = 0.9
random_page_cost          = 1.1 (SSD)        effective_io_concurrency = 200 (SSD)
default_statistics_target = 100–500
max_connections           = 100–200 (use a pooler)
max_worker_processes      = #cores     max_parallel_workers = #cores     max_parallel_workers_per_gather = 2–4
huge_pages                = try         jit = off (OLTP)
io_method                 = worker | io_uring | sync    (18)
autovacuum_vacuum_scale_factor = 0.05 (or per-table 0.01)   autovacuum_vacuum_cost_limit = 1000–2000
```
Generate with PGTune / `pgtune`; apply via `ALTER SYSTEM SET ...; SELECT pg_reload_conf();`

## Connection pooling (pgbouncer)
```ini
; pgbouncer.ini
[databases]
appdb = host=127.0.0.1 port=5432 dbname=appdb
[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction            ; session | transaction | statement
max_client_conn = 2000
default_pool_size = 20
```
Admin: `psql -p 6432 pgbouncer -c "SHOW POOLS;"` · `SHOW STATS;` · `RELOAD;` · `PAUSE;` · `RESUME;`
(transaction mode: no session-level SET/advisory locks/LISTEN; prepared statements need protocol support)

## Caching / buffers
```sql
CREATE EXTENSION pg_buffercache;
SELECT c.relname, count(*) AS buffers FROM pg_buffercache b JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid) GROUP BY 1 ORDER BY 2 DESC LIMIT 10;
CREATE EXTENSION pg_prewarm;   SELECT pg_prewarm('t');
SELECT sum(heap_blks_hit)*100.0/nullif(sum(heap_blks_hit+heap_blks_read),0) AS hit_pct FROM pg_statio_user_tables;
```

## auto_explain
```
shared_preload_libraries = 'auto_explain'
auto_explain.log_min_duration = '500ms'
auto_explain.log_analyze = on
auto_explain.log_buffers = on
auto_explain.log_nested_statements = on
```

## Parallelism
```sql
SET max_parallel_workers_per_gather = 4;
SET parallel_setup_cost = 100;   SET parallel_tuple_cost = 0.05;   SET min_parallel_table_scan_size = '8MB';
ALTER TABLE t SET (parallel_workers = 8);
-- functions must be PARALLEL SAFE
```

## Statistics
```sql
ANALYZE t;
ALTER TABLE t ALTER COLUMN c SET STATISTICS 1000;
CREATE STATISTICS s (dependencies, ndistinct, mcv) ON a, b FROM t;
SELECT * FROM pg_stats WHERE tablename='t' AND attname='c';
```

## Materialized views for heavy aggregates
```sql
CREATE MATERIALIZED VIEW daily AS SELECT date_trunc('day', ts) d, count(*) n FROM e GROUP BY 1;
CREATE UNIQUE INDEX ON daily (d);
REFRESH MATERIALIZED VIEW CONCURRENTLY daily;     -- schedule via pg_cron
```

## Table design
```
Use narrow types (int vs bigint, timestamptz, numeric only when needed), order columns big → small to reduce padding,
text instead of varchar(n), avoid EAV, avoid wide rows with TOAST churn, set fillfactor for hot-update tables,
partition huge tables, use partial/covering indexes, FK columns indexed.
```
