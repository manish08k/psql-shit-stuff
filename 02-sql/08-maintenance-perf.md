# 08 — Maintenance & Performance

## VACUUM
```sql
VACUUM;
VACUUM t;
VACUUM (VERBOSE, ANALYZE) t;
VACUUM (FULL) t;                                -- rewrites table, ACCESS EXCLUSIVE lock
VACUUM (FREEZE) t;
VACUUM (PARALLEL 4) t;
VACUUM (INDEX_CLEANUP off, TRUNCATE off) t;
VACUUM (SKIP_LOCKED, DISABLE_PAGE_SKIPPING) t;
VACUUM (ONLY_DATABASE_STATS);
VACUUM (BUFFER_USAGE_LIMIT '256kB') t;
VACUUM (PROCESS_MAIN false, PROCESS_TOAST true) t;
VACUUM (ANALYZE) t (col1, col2);
```

## ANALYZE
```sql
ANALYZE;
ANALYZE VERBOSE t;
ANALYZE t (col1, col2);
ANALYZE (SKIP_LOCKED) t;
ANALYZE ONLY partitioned_t;
```

## REINDEX
```sql
REINDEX INDEX idx;
REINDEX TABLE t;
REINDEX SCHEMA s;
REINDEX DATABASE db;
REINDEX SYSTEM db;
REINDEX (VERBOSE) TABLE CONCURRENTLY t;
REINDEX INDEX CONCURRENTLY idx;
REINDEX (TABLESPACE ts) TABLE t;
```

## CLUSTER
```sql
CLUSTER t USING idx;
CLUSTER t;                                      -- reuse previous index
CLUSTER VERBOSE;
```

## CHECKPOINT
```sql
CHECKPOINT;
```

## REFRESH MATERIALIZED VIEW
```sql
REFRESH MATERIALIZED VIEW mv;
REFRESH MATERIALIZED VIEW CONCURRENTLY mv;      -- needs unique index
REFRESH MATERIALIZED VIEW mv WITH NO DATA;
```

## EXPLAIN
```sql
EXPLAIN SELECT ...;
EXPLAIN ANALYZE SELECT ...;                     -- executes the query!
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, COSTS, TIMING, SETTINGS, WAL, SUMMARY) SELECT ...;
EXPLAIN (ANALYZE, FORMAT JSON|YAML|XML|TEXT) SELECT ...;
EXPLAIN (GENERIC_PLAN) SELECT * FROM t WHERE id = $1;
EXPLAIN (ANALYZE, MEMORY) SELECT ...;
EXPLAIN (ANALYZE, SERIALIZE) SELECT ...;
-- safe way to analyze writes:
BEGIN; EXPLAIN ANALYZE UPDATE ...; ROLLBACK;
```

## Planner switches (for diagnosis)
```sql
SET enable_seqscan = off;     SET enable_indexscan = off;     SET enable_bitmapscan = off;
SET enable_hashjoin = off;    SET enable_mergejoin = off;     SET enable_nestloop = off;
SET random_page_cost = 1.1;   SET effective_cache_size = '12GB';   SET default_statistics_target = 500;
SET max_parallel_workers_per_gather = 4;      SET jit = off;
```

## Extended statistics
```sql
CREATE STATISTICS s (dependencies, ndistinct, mcv) ON a, b FROM t;
ANALYZE t;
SELECT * FROM pg_stats_ext;
```

## Table / index health
```sql
SELECT relname, n_live_tup, n_dead_tup, last_vacuum, last_autovacuum, last_analyze FROM pg_stat_user_tables ORDER BY n_dead_tup DESC;
SELECT relname, seq_scan, idx_scan FROM pg_stat_user_tables;
SELECT indexrelname, idx_scan FROM pg_stat_user_indexes WHERE idx_scan = 0;
SELECT pg_size_pretty(pg_total_relation_size('t')), pg_size_pretty(pg_relation_size('t')), pg_size_pretty(pg_indexes_size('t'));
SELECT * FROM pg_stat_progress_vacuum;   SELECT * FROM pg_stat_progress_create_index;
-- extensions: CREATE EXTENSION pg_stat_statements; pgstattuple; pg_buffercache; amcheck;
SELECT query, calls, total_exec_time, mean_exec_time, rows FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 20;
SELECT pg_stat_statements_reset();
```

## Autovacuum per-table tuning
```sql
ALTER TABLE t SET (autovacuum_vacuum_scale_factor = 0.02, autovacuum_analyze_scale_factor = 0.01, autovacuum_vacuum_cost_limit = 1000);
```
