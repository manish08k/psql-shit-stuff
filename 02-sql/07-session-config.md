# 07 — Session & Configuration

## SET / SHOW / RESET
```sql
SET param = value;                      -- session
SET LOCAL param = value;                -- current transaction only
SET param TO DEFAULT;
SET search_path TO s1, public;
SET statement_timeout = '30s';
SET lock_timeout = '5s';
SET idle_in_transaction_session_timeout = '1min';
SET work_mem = '64MB';
SET TIME ZONE 'Asia/Kolkata';
SET client_encoding = 'UTF8';
SET application_name = 'myapp';
SET role r;                             -- see SET ROLE
SET enable_seqscan = off;
SET default_transaction_read_only = on;
SELECT set_config('app.user_id', '42', false);   -- 3rd arg: true = local
SELECT current_setting('app.user_id');

SHOW param;
SHOW ALL;
SHOW search_path;   SHOW server_version;   SHOW data_directory;   SHOW config_file;   SHOW hba_file;
SHOW max_connections;   SHOW shared_buffers;   SHOW TRANSACTION ISOLATION LEVEL;

RESET param;
RESET ALL;
```

## Persist config
```sql
ALTER SYSTEM SET shared_buffers = '4GB';        -- postgresql.auto.conf (restart needed for some)
ALTER SYSTEM RESET shared_buffers;
ALTER DATABASE db SET work_mem = '32MB';
ALTER ROLE r SET statement_timeout = '10s';
ALTER ROLE r IN DATABASE db SET search_path = s;
SELECT pg_reload_conf();
SELECT name, setting, unit, context, source, pending_restart FROM pg_settings WHERE name LIKE '%mem%';
SELECT * FROM pg_file_settings WHERE error IS NOT NULL;
```
Parameter `context`: `internal | postmaster (restart) | sighup (reload) | superuser | user`

## DISCARD
```sql
DISCARD ALL;        DISCARD PLANS;      DISCARD SEQUENCES;      DISCARD TEMP;
```

## LOAD
```sql
LOAD 'auto_explain';
LOAD '$libdir/plugin_name';
```

## Session info
```sql
SELECT current_database(), current_schema(), current_schemas(true), current_user, version();
SELECT pg_backend_pid(), inet_client_addr(), inet_server_addr();
SELECT now(), clock_timestamp(), transaction_timestamp(), statement_timestamp();
```
