# 10 — psql Recipes (daily DBA + developer work)

## Where am I?
```
\conninfo
SELECT current_database(), current_user, inet_server_addr(), version();
\dconfig *dir*
SHOW data_directory;
```

## Explore a database fast
```
\l+            \dn+            \dt+ *.*            \d+ table
\dx            \du+            \dconfig            \df+ schema.*
\dt+ public.*  \dv+ \dm+       \di+ table*
```

## Find stuff
```
\dt *order*                 tables with 'order' in name
\d *customer*               any relation
\df *tax*                   functions
\dn+ *                      all schemas
SELECT table_schema, table_name FROM information_schema.columns WHERE column_name = 'email';
```

## Run + inspect plans
```
\timing on
EXPLAIN (ANALYZE, BUFFERS) SELECT ... ;
\x on
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT ... \g explain.json
```

## Live monitoring with \watch
```sql
SELECT pid, state, wait_event, now()-query_start AS run, left(query,60) FROM pg_stat_activity WHERE state<>'idle' ORDER BY run DESC \watch 2
SELECT count(*), state FROM pg_stat_activity GROUP BY state \watch 1
SELECT * FROM pg_stat_progress_vacuum \watch 5
SELECT * FROM pg_stat_replication \watch 2
SELECT pg_size_pretty(pg_database_size(current_database())) \watch 10
```

## Cancel / kill
```sql
SELECT pg_cancel_backend(pid)   FROM pg_stat_activity WHERE state='active' AND now()-query_start > interval '10 min';
SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname='db' AND pid <> pg_backend_pid();
```

## Generate & run bulk DDL (`\gexec`)
```sql
-- vacuum bloated tables
SELECT format('VACUUM (ANALYZE) %I.%I', schemaname, relname) FROM pg_stat_user_tables WHERE n_dead_tup > 50000 \gexec
-- reindex all indexes of a schema concurrently
SELECT format('REINDEX INDEX CONCURRENTLY %I.%I', schemaname, indexname) FROM pg_indexes WHERE schemaname='app' \gexec
-- change owner of everything in a schema
SELECT format('ALTER %s %I.%I OWNER TO app_owner', CASE relkind WHEN 'r' THEN 'TABLE' WHEN 'v' THEN 'VIEW' WHEN 'S' THEN 'SEQUENCE' WHEN 'm' THEN 'MATERIALIZED VIEW' END, n.nspname, c.relname)
FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace WHERE n.nspname='app' AND relkind IN ('r','v','S','m') \gexec
-- grant on all tables
SELECT format('GRANT SELECT ON %I.%I TO ro', schemaname, tablename) FROM pg_tables WHERE schemaname='app' \gexec
-- sync sequences to max(id)
SELECT format('SELECT setval(%L, coalesce(max(%I),1)) FROM %I.%I', pg_get_serial_sequence(format('%I.%I', schemaname, tablename), column_name), column_name, schemaname, tablename)
FROM information_schema.columns c JOIN pg_tables t ON t.tablename=c.table_name AND t.schemaname=c.table_schema
WHERE pg_get_serial_sequence(format('%I.%I', schemaname, tablename), column_name) IS NOT NULL \gexec
-- drop all tables with prefix
SELECT format('DROP TABLE IF EXISTS %I CASCADE', tablename) FROM pg_tables WHERE schemaname='public' AND tablename LIKE 'tmp\_%' \gexec
-- count rows in every table
SELECT format('SELECT %L AS tbl, count(*) FROM %I.%I', tablename, schemaname, tablename) FROM pg_tables WHERE schemaname='public' \gexec
```

## Capture a value and reuse
```sql
SELECT max(id) AS last_id FROM orders \gset
DELETE FROM order_items WHERE order_id < :last_id - 1000;
SELECT oid AS tbl_oid FROM pg_class WHERE relname='orders' \gset
```

## Migration runner pattern
```bash
for f in migrations/*.sql; do
  echo "== $f"
  psql -X -v ON_ERROR_STOP=1 -1 -d "$DB" -f "$f" || { echo "FAILED $f"; exit 1; }
done
```
Within a migration file:
```sql
\set ON_ERROR_STOP on
SET lock_timeout = '3s';
BEGIN;
ALTER TABLE t ADD COLUMN c int;
COMMIT;
```

## Backup / restore with psql in the loop
```bash
pg_dump -Fc db > db.dump
pg_restore -l db.dump | head
psql -d postgres -c "CREATE DATABASE db_copy TEMPLATE db"
psql -d db -f plain.sql
psql -XAtc "SELECT pg_size_pretty(pg_database_size('db'))"
```

## Compare quickly
```bash
diff <(psql -XAtc "\d+ t" db1) <(psql -XAtc "\d+ t" db2)
diff <(pg_dump -s db1) <(pg_dump -s db2)
```

## Dump query results in many shapes
```bash
psql --csv -c "SELECT * FROM t" db > t.csv
psql -At -c "SELECT json_agg(t) FROM t" db > t.json
psql -x -c "SELECT * FROM t WHERE id=1" db
psql -P format=html -c "SELECT * FROM t" db > t.html
psql -At -F $'\t' -c "SELECT * FROM t" db > t.tsv
```

## Interactive "what is slow" session
```sql
\x auto
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
SELECT calls, round(total_exec_time::numeric,0) AS total_ms, round(mean_exec_time::numeric,1) AS mean_ms, left(query,100) FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;
\set ECHO_HIDDEN off
```

## Safe production habits
```
\set ON_ERROR_ROLLBACK interactive        -- a typo won't abort your open tx
\set AUTOCOMMIT off                       -- every statement needs explicit COMMIT (prod safety)
BEGIN; ... check with SELECT ...; COMMIT;   -- or ROLLBACK;
SET statement_timeout='30s'; SET lock_timeout='3s';
\set PROMPT1 '%[%033[1;41;37m%] PROD %[%033[0m%] %n@%/%R%x%# '     -- loud prod prompt
psql -X -h prod -U readonly app             -- readonly role for exploration
```

## Test connectivity & auth problems
```bash
pg_isready -h host -p 5432
psql "host=host dbname=postgres user=u connect_timeout=3" -c "select 1"
psql -w ...                                    # fail instead of prompting (scripts)
PGPASSFILE=./pgpass psql ...
psql -d "..." -c "SELECT * FROM pg_hba_file_rules" # (as superuser)
```
| Error | Likely cause / fix |
|---|---|
| `connection refused` | server down / wrong port / `listen_addresses` / firewall |
| `could not connect to server: No such file (socket)` | wrong socket dir → `-h /var/run/postgresql` or use `-h localhost` |
| `FATAL: Peer authentication failed` | local socket uses `peer`; connect as OS user or use `-h localhost` + password |
| `FATAL: password authentication failed` | wrong password / `pg_hba.conf` method / role has no password |
| `FATAL: no pg_hba.conf entry for host ...` | add a matching line, `SELECT pg_reload_conf();` |
| `FATAL: database "x" does not exist` | `psql -l`; connect to `postgres` db first |
| `FATAL: too many connections` | raise `max_connections` / use pgbouncer / kill idle sessions |
| `SSL error / SSL required` | set `sslmode`, provide `sslrootcert` |
| `permission denied for table` | `GRANT`; check `\dp table` |
| `current transaction is aborted, commands ignored until end of transaction block` | `ROLLBACK;` (or use `ON_ERROR_ROLLBACK`) |
| `\copy: parse error` | `\copy` must be one line; no semicolon needed |
| `could not open file ... for reading: Permission denied` (COPY) | server-side COPY: file must be readable by the postgres OS user — use `\copy` instead |
| `psql: invalid command \xyz` | typo/unsupported meta-command for this psql version → `\?` |

## Version compatibility
psql works best with the same or older server major version. `\d` commands work back to 9.2. psql 18 against older servers: some `\d` output columns/features may be absent.
