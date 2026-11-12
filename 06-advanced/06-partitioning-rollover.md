# 06 — Partitioning & ROLLOVER (partitions, logs, WAL, sequences, XID)

"Rollover" in PostgreSQL = 5 different things. All covered below:
1. Partition rollover (create next partition, retire old)
2. Log file rotation
3. WAL segment switch / archive rotation
4. Sequence / identity exhaustion (integer overflow, wrap/cycle)
5. Transaction-ID wraparound

---
## 1. Partition rollover

### Setup (range by time)
```sql
CREATE TABLE events (
  id bigint GENERATED ALWAYS AS IDENTITY,
  ts timestamptz NOT NULL,
  payload jsonb,
  PRIMARY KEY (id, ts)                       -- PK must include partition key
) PARTITION BY RANGE (ts);

CREATE TABLE events_default PARTITION OF events DEFAULT;   -- safety net
CREATE TABLE events_2026_11 PARTITION OF events FOR VALUES FROM ('2026-11-01') TO ('2026-12-01');
CREATE INDEX ON events (ts);                                -- cascades to all partitions
```

### Roll forward: create next partition
```sql
CREATE TABLE events_2026_12 PARTITION OF events FOR VALUES FROM ('2026-12-01') TO ('2027-01-01');
-- zero-lock method: build standalone, then attach
CREATE TABLE events_2027_01 (LIKE events INCLUDING ALL);
ALTER TABLE events_2027_01 ADD CONSTRAINT chk CHECK (ts >= '2027-01-01' AND ts < '2027-02-01');
ALTER TABLE events ATTACH PARTITION events_2027_01 FOR VALUES FROM ('2027-01-01') TO ('2027-02-01');
ALTER TABLE events_2027_01 DROP CONSTRAINT chk;
```

### Roll off: retire old partition
```sql
ALTER TABLE events DETACH PARTITION events_2026_01 CONCURRENTLY;     -- no long lock
-- then archive or drop
pg_dump -Fc -t events_2026_01 db > events_2026_01.dump
DROP TABLE events_2026_01;
-- or move to cheaper storage
ALTER TABLE events_2026_01 SET TABLESPACE slow_ts;
```
Dropping a partition is instant vs `DELETE ... WHERE ts < x` (slow + bloat).

### Automated rollover function
```sql
CREATE OR REPLACE PROCEDURE roll_partitions(parent text, months_ahead int DEFAULT 3, keep_months int DEFAULT 12)
LANGUAGE plpgsql AS $$
DECLARE d date; nm text;
BEGIN
  FOR i IN 0..months_ahead LOOP
    d := date_trunc('month', now() + make_interval(months => i))::date;
    nm := format('%s_%s', parent, to_char(d, 'YYYY_MM'));
    EXECUTE format('CREATE TABLE IF NOT EXISTS %I PARTITION OF %I FOR VALUES FROM (%L) TO (%L)',
                   nm, parent, d, (d + interval '1 month')::date);
  END LOOP;
  FOR nm IN
    SELECT c.relname FROM pg_inherits i JOIN pg_class c ON c.oid = i.inhrelid JOIN pg_class p ON p.oid = i.inhparent
    WHERE p.relname = parent AND c.relname < format('%s_%s', parent, to_char(now() - make_interval(months => keep_months), 'YYYY_MM'))
  LOOP
    EXECUTE format('DROP TABLE %I', nm);
  END LOOP;
END $$;
CALL roll_partitions('events');
```

### Schedule it
```sql
-- pg_cron
CREATE EXTENSION pg_cron;
SELECT cron.schedule('roll-partitions', '0 1 * * *', $$CALL roll_partitions('events')$$);
SELECT * FROM cron.job;   SELECT cron.unschedule('roll-partitions');
SELECT * FROM cron.job_run_details ORDER BY start_time DESC LIMIT 10;
-- pg_partman
CREATE EXTENSION pg_partman;
SELECT partman.create_parent('public.events', 'ts', '1 month');
UPDATE partman.part_config SET retention = '12 months', retention_keep_table = false WHERE parent_table = 'public.events';
CALL partman.run_maintenance_proc();
```
OS cron alternative: `0 1 * * * psql -d db -c "CALL roll_partitions('events')"`

### Other partition types
```sql
PARTITION BY LIST (region)            FOR VALUES IN ('eu','uk')
PARTITION BY HASH (id)                FOR VALUES WITH (MODULUS 8, REMAINDER 0)
-- sub-partitioning
CREATE TABLE m_2026 PARTITION OF m FOR VALUES FROM ('2026-01-01') TO ('2027-01-01') PARTITION BY HASH (id);
```

### Inspect
```sql
SELECT * FROM pg_partition_tree('events');
SELECT pg_get_partkeydef('events'::regclass);
SELECT inhrelid::regclass, pg_get_expr(c.relpartbound, c.oid) FROM pg_inherits i JOIN pg_class c ON c.oid = i.inhrelid WHERE inhparent = 'events'::regclass;
\d+ events
EXPLAIN SELECT * FROM events WHERE ts >= '2026-11-01' AND ts < '2026-12-01';    -- check pruning
SHOW enable_partition_pruning;
```

### Move rows out of DEFAULT before adding a partition
```sql
BEGIN;
ALTER TABLE events DETACH PARTITION events_default;
CREATE TABLE events_2027_02 PARTITION OF events FOR VALUES FROM ('2027-02-01') TO ('2027-03-01');
INSERT INTO events SELECT * FROM events_default WHERE ts >= '2027-02-01' AND ts < '2027-03-01';
DELETE FROM events_default WHERE ts >= '2027-02-01' AND ts < '2027-03-01';
ALTER TABLE events ATTACH PARTITION events_default DEFAULT;
COMMIT;
```

---
## 2. Log rotation
```
# postgresql.conf
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_age = 1d              # 0 disables time-based
log_rotation_size = 100MB          # 0 disables size-based
log_truncate_on_rotation = on      # reuse names, e.g. 'postgresql-%a.log' = 7-day cycle
log_file_mode = 0600
```
```bash
pg_ctl logrotate -D $PGDATA              # rotate now
```
```sql
SELECT pg_rotate_logfile();             -- rotate now (SQL)
SELECT pg_current_logfile();
SELECT * FROM pg_ls_logdir();           -- if you have privilege
```
logrotate (syslog/stderr redirect) example `/etc/logrotate.d/postgresql`:
```
/var/log/postgresql/*.log { daily rotate 14 compress delaycompress missingok notifempty copytruncate }
```

---
## 3. WAL switch / rotation
```sql
SELECT pg_switch_wal();                         -- close current WAL segment, start next
SELECT pg_current_wal_lsn(), pg_walfile_name(pg_current_wal_lsn());
SELECT * FROM pg_ls_waldir() ORDER BY modification DESC LIMIT 5;
CHECKPOINT;
SELECT * FROM pg_stat_archiver;                 -- archive success/failure
```
```
archive_mode = on
archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'
archive_timeout = 300              # force switch every 5 min
wal_keep_size = 2GB                # retain WAL for standbys
max_wal_size = 4GB   min_wal_size = 1GB
max_slot_wal_keep_size = 10GB      # protect disk from stuck slots
wal_segment_size set at initdb (--wal-segsize)
```
Clean archive: `pg_archivecleanup /archive <oldest_needed_wal_file>`
Find stuck slots: `SELECT slot_name, active, wal_status, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) FROM pg_replication_slots;`

---
## 4. Sequence rollover / integer overflow
```sql
-- how close to limit
SELECT schemaname, sequencename, last_value, max_value, round(100.0*last_value/max_value,2) AS pct_used FROM pg_sequences ORDER BY pct_used DESC NULLS LAST;
-- identity / serial types: serial=int4 max 2,147,483,647 ; bigserial=int8
-- convert int → bigint (rewrite!)
ALTER TABLE t ALTER COLUMN id TYPE bigint;
ALTER SEQUENCE t_id_seq AS bigint MAXVALUE 9223372036854775807;
-- cycle (wraps back to MINVALUE — can cause dup errors)
ALTER SEQUENCE seq CYCLE;   ALTER SEQUENCE seq NO CYCLE;
-- reset
ALTER SEQUENCE seq RESTART WITH 1;
SELECT setval('seq', (SELECT coalesce(max(id),0)+1 FROM t), false);
-- per-period counter that restarts (e.g. yearly invoice no.)
CREATE TABLE counters (yr int PRIMARY KEY, n int NOT NULL);
INSERT INTO counters VALUES (2026, 1) ON CONFLICT (yr) DO UPDATE SET n = counters.n + 1 RETURNING n;
```

---
## 5. Transaction-ID wraparound (the "rollover" that stops the DB)
```sql
SELECT datname, age(datfrozenxid), 2147483647 - age(datfrozenxid) AS left_before_wrap FROM pg_database ORDER BY 2 DESC;
SELECT relname, age(relfrozenxid), mxid_age(relminmxid) FROM pg_class WHERE relkind IN ('r','m','t') ORDER BY 2 DESC LIMIT 20;
```
```
autovacuum_freeze_max_age = 200000000        # forces anti-wraparound autovacuum
vacuum_freeze_min_age = 50000000
vacuum_freeze_table_age = 150000000
vacuum_failsafe_age = 1600000000
```
Fix: `VACUUM (FREEZE, VERBOSE) big_table;` · `vacuumdb -a -F -j 4` · kill old transactions/prepared xacts/stale replication slots:
```sql
SELECT pid, xact_start FROM pg_stat_activity WHERE backend_xid IS NOT NULL OR backend_xmin IS NOT NULL ORDER BY xact_start;
SELECT * FROM pg_prepared_xacts;
SELECT slot_name, xmin, catalog_xmin FROM pg_replication_slots;
```

---
## Retention / archival patterns
```sql
-- batch delete without huge lock
DELETE FROM t WHERE id IN (SELECT id FROM t WHERE ts < now() - interval '90 days' LIMIT 10000);
-- table swap rollover
BEGIN;
ALTER TABLE logs RENAME TO logs_old;
CREATE TABLE logs (LIKE logs_old INCLUDING ALL);
COMMIT;
-- move old data to archive table
WITH moved AS (DELETE FROM logs WHERE ts < now() - interval '30 days' RETURNING *) INSERT INTO logs_archive SELECT * FROM moved;
```
