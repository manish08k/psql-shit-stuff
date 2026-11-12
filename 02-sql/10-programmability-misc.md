# 10 — PL/pgSQL, Triggers, Comments, FDW, Replication, Extensions

## PL/pgSQL skeleton
```sql
CREATE FUNCTION f(p int) RETURNS int LANGUAGE plpgsql AS $$
DECLARE
  v int := 0;
  r record;
BEGIN
  IF p > 0 THEN v := 1; ELSIF p = 0 THEN v := 0; ELSE v := -1; END IF;
  CASE WHEN p = 1 THEN ... ELSE ... END CASE;
  FOR i IN 1..10 LOOP ... END LOOP;
  FOR r IN SELECT * FROM t LOOP ... END LOOP;
  WHILE cond LOOP ... EXIT WHEN x; CONTINUE; END LOOP;
  FOREACH x IN ARRAY arr LOOP ... END LOOP;
  SELECT a INTO v FROM t WHERE id = p;
  PERFORM some_function();
  EXECUTE format('SELECT %I FROM %I WHERE id = $1', col, tbl) INTO v USING p;
  RETURN QUERY SELECT ...;   RETURN NEXT;   RETURN v;
  RAISE NOTICE 'val=%', v;   RAISE EXCEPTION 'bad %', v USING ERRCODE = '22000';
  GET DIAGNOSTICS n = ROW_COUNT;
  EXCEPTION WHEN unique_violation THEN ... WHEN OTHERS THEN RAISE;
END $$;
```

## Trigger function
```sql
CREATE FUNCTION trg_fn() RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
  NEW.updated_at := now();      -- TG_OP, TG_TABLE_NAME, OLD, NEW available
  RETURN NEW;
END $$;
CREATE TRIGGER trg BEFORE UPDATE ON t FOR EACH ROW EXECUTE FUNCTION trg_fn();
```

## COMMENT
```sql
COMMENT ON TABLE t IS 'text';
COMMENT ON COLUMN t.c IS 'text';
COMMENT ON {DATABASE|SCHEMA|INDEX|VIEW|FUNCTION f(int)|ROLE|EXTENSION|...} name IS 'text';
COMMENT ON TABLE t IS NULL;                     -- remove
```

## Extensions
```sql
CREATE EXTENSION pg_stat_statements;   postgres_fdw;   pgcrypto;   "uuid-ossp";   pg_trgm;   hstore;   citext;
CREATE EXTENSION btree_gin;   btree_gist;   ltree;   tablefunc;   pg_buffercache;   pgstattuple;   amcheck;   file_fdw;   dblink;
SELECT * FROM pg_available_extensions;   SELECT * FROM pg_extension;
\dx
ALTER EXTENSION e UPDATE;   DROP EXTENSION e;
```

## Foreign data (postgres_fdw)
```sql
CREATE EXTENSION postgres_fdw;
CREATE SERVER r FOREIGN DATA WRAPPER postgres_fdw OPTIONS (host 'h', port '5432', dbname 'd');
CREATE USER MAPPING FOR CURRENT_USER SERVER r OPTIONS (user 'u', password 'p');
IMPORT FOREIGN SCHEMA public FROM SERVER r INTO local_s;
CREATE FOREIGN TABLE ft (...) SERVER r OPTIONS (schema_name 'public', table_name 't');
```

## Logical replication
```sql
-- publisher (wal_level = logical)
CREATE PUBLICATION pub FOR ALL TABLES;
CREATE PUBLICATION pub FOR TABLE t (col1, col2) WHERE (active);
-- subscriber
CREATE SUBSCRIPTION sub CONNECTION 'host=pub dbname=d user=repl password=x' PUBLICATION pub;
ALTER SUBSCRIPTION sub REFRESH PUBLICATION;
ALTER SUBSCRIPTION sub DISABLE;   DROP SUBSCRIPTION sub;
SELECT * FROM pg_stat_subscription;   SELECT * FROM pg_publication_tables;
SELECT * FROM pg_replication_slots;
SELECT pg_create_logical_replication_slot('s', 'pgoutput');
SELECT pg_drop_replication_slot('s');
```

## Physical replication helpers
```sql
SELECT * FROM pg_stat_replication;
SELECT pg_is_in_recovery();
SELECT pg_current_wal_lsn(), pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn();
SELECT pg_wal_lsn_diff(a, b);
SELECT pg_promote();
SELECT pg_wal_replay_pause();   SELECT pg_wal_replay_resume();
SELECT pg_create_physical_replication_slot('slot1');
SELECT pg_switch_wal();
SELECT pg_backup_start('label');   SELECT * FROM pg_backup_stop();
```

## Large objects
```sql
SELECT lo_import('/path/f');   SELECT lo_export(oid, '/path/f');   SELECT lo_unlink(oid);
\lo_import f   \lo_export oid f   \lo_list   \lo_unlink oid
```

## Event trigger example
```sql
CREATE FUNCTION log_ddl() RETURNS event_trigger LANGUAGE plpgsql AS $$
BEGIN RAISE NOTICE '% on %', tg_tag, tg_event; END $$;
CREATE EVENT TRIGGER ddl_log ON ddl_command_end EXECUTE FUNCTION log_ddl();
```
