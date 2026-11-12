# Cheatsheet — Admin & Monitoring

## Sessions & queries
```sql
SELECT pid, usename, datname, state, wait_event_type, wait_event, now()-query_start AS runtime, left(query,100)
FROM pg_stat_activity WHERE state <> 'idle' ORDER BY runtime DESC;
SELECT pid, now()-xact_start AS tx_age, state, query FROM pg_stat_activity WHERE xact_start IS NOT NULL ORDER BY tx_age DESC;
SELECT count(*), state FROM pg_stat_activity GROUP BY state;
SELECT pg_cancel_backend(pid);                    -- cancel current query
SELECT pg_terminate_backend(pid);                 -- kill session
-- kill all sessions of a db
SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname='db' AND pid <> pg_backend_pid();
-- idle in transaction
SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state='idle in transaction' AND now()-state_change > interval '10 min';
```

## Locks / blocking
```sql
SELECT pid, pg_blocking_pids(pid) AS blocked_by, left(query,80) FROM pg_stat_activity WHERE cardinality(pg_blocking_pids(pid)) > 0;
SELECT l.pid, l.locktype, l.mode, l.granted, a.query FROM pg_locks l JOIN pg_stat_activity a USING (pid) WHERE NOT l.granted;
SELECT * FROM pg_locks WHERE relation = 't'::regclass;
```

## Connections
```sql
SHOW max_connections;
SELECT count(*) FROM pg_stat_activity;
SELECT usename, count(*) FROM pg_stat_activity GROUP BY usename ORDER BY 2 DESC;
SELECT rolname, rolconnlimit FROM pg_roles WHERE rolconnlimit <> -1;
```

## Stats
```sql
SELECT * FROM pg_stat_database;
SELECT * FROM pg_stat_bgwriter;      SELECT * FROM pg_stat_checkpointer;      SELECT * FROM pg_stat_io;      SELECT * FROM pg_stat_wal;
SELECT * FROM pg_stat_user_tables;   SELECT * FROM pg_statio_user_tables;
SELECT sum(blks_hit)*100.0/nullif(sum(blks_hit)+sum(blks_read),0) AS cache_hit_pct FROM pg_stat_database;
SELECT pg_stat_reset();
SELECT * FROM pg_stat_progress_vacuum;   pg_stat_progress_cluster;   pg_stat_progress_create_index;   pg_stat_progress_basebackup;   pg_stat_progress_copy;
SELECT * FROM pg_stat_replication;   SELECT * FROM pg_stat_wal_receiver;   SELECT * FROM pg_replication_slots;
SELECT * FROM pg_stat_archiver;
SELECT * FROM pg_stat_ssl;           SELECT * FROM pg_stat_gssapi;
```

## Transaction ID wraparound watch
```sql
SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY 2 DESC;
SELECT relname, age(relfrozenxid) FROM pg_class WHERE relkind='r' ORDER BY 2 DESC LIMIT 10;
```

## Replication lag
```sql
-- primary
SELECT client_addr, state, sent_lsn, replay_lsn, pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes, replay_lag FROM pg_stat_replication;
-- standby
SELECT now() - pg_last_xact_replay_timestamp() AS lag;
SELECT slot_name, active, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) FROM pg_replication_slots;
```

## Config
```sql
SHOW data_directory;  SHOW config_file;  SHOW hba_file;  SHOW log_directory;
SELECT pg_conf_load_time();     SELECT pg_postmaster_start_time();
SELECT name, setting FROM pg_settings WHERE source <> 'default';
SELECT * FROM pg_hba_file_rules;     SELECT * FROM pg_ident_file_mappings;
```

## Logging parameters
```
log_min_duration_statement = 500       log_statement = 'ddl'|'mod'|'all'
log_connections = on                   log_disconnections = on
log_lock_waits = on                    log_checkpoints = on
log_autovacuum_min_duration = 0        log_line_prefix = '%m [%p] %u@%d '
shared_preload_libraries = 'pg_stat_statements,auto_explain'
```

## Key tuning parameters
```
shared_buffers  effective_cache_size  work_mem  maintenance_work_mem  max_wal_size  min_wal_size
checkpoint_timeout  checkpoint_completion_target  wal_buffers  random_page_cost  effective_io_concurrency
max_connections  max_worker_processes  max_parallel_workers  max_parallel_workers_per_gather
autovacuum_max_workers  autovacuum_naptime  default_statistics_target  io_method (18: sync|worker|io_uring)
```

## OS-level
```bash
ps aux | grep postgres
ss -ltnp | grep 5432
tail -f /var/log/postgresql/postgresql-18-main.log
journalctl -u postgresql -f
du -sh $PGDATA/base/*
```
