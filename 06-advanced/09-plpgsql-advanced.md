# 09 — PL/pgSQL, Procedures, Triggers, Dynamic SQL (advanced)

## Function attributes
```sql
CREATE FUNCTION f(a int, OUT b int, INOUT c int, VARIADIC d int[]) ...
RETURNS int | void | TABLE(...) | SETOF t | record | trigger | event_trigger
LANGUAGE sql | plpgsql | plpython3u | plperl | pltcl
IMMUTABLE | STABLE | VOLATILE
STRICT | CALLED ON NULL INPUT
PARALLEL SAFE | RESTRICTED | UNSAFE
SECURITY DEFINER | SECURITY INVOKER
LEAKPROOF   COST 100   ROWS 1000
SET search_path = public, pg_temp
AS $$ ... $$ | AS 'obj_file', 'symbol'
SUPPORT support_fn
```
SQL-standard body (inlined/tracked deps): `CREATE FUNCTION f(a int) RETURNS int LANGUAGE sql IMMUTABLE RETURN a + 1;`
`BEGIN ATOMIC SELECT ...; END;`

## PL/pgSQL structure
```sql
DO $$
DECLARE
  n int := 0;  r record;  t tbl%ROWTYPE;  c tbl.col%TYPE;  arr int[] := ARRAY[1,2,3];  cur refcursor;
  CONSTANT pi numeric := 3.14;
BEGIN
  -- assignment / queries
  n := n + 1;
  SELECT count(*) INTO n FROM t;
  SELECT * INTO STRICT r FROM t WHERE id = 1;             -- errors if 0 or >1 rows
  INSERT INTO t VALUES (...) RETURNING id INTO n;
  PERFORM pg_sleep(1);                                    -- run query, discard result
  GET DIAGNOSTICS n = ROW_COUNT;      IF FOUND THEN ... END IF;
  GET STACKED DIAGNOSTICS msg = MESSAGE_TEXT, code = RETURNED_SQLSTATE, detail = PG_EXCEPTION_DETAIL, ctx = PG_EXCEPTION_CONTEXT;

  -- control
  IF a THEN ... ELSIF b THEN ... ELSE ... END IF;
  CASE x WHEN 1 THEN ... WHEN 2 THEN ... ELSE ... END CASE;
  LOOP ... EXIT WHEN cond; END LOOP;
  WHILE cond LOOP ... END LOOP;
  FOR i IN 1..10 BY 2 LOOP ... END LOOP;        FOR i IN REVERSE 10..1 LOOP ... END LOOP;
  FOR r IN SELECT * FROM t LOOP ... END LOOP;
  FOR r IN EXECUTE format('SELECT * FROM %I', tbl) LOOP ... END LOOP;
  FOREACH x IN ARRAY arr LOOP ... END LOOP;
  <<outer>> LOOP ... EXIT outer; CONTINUE outer; END LOOP;

  -- output / errors
  RAISE NOTICE 'x=%', n;  RAISE WARNING ...;  RAISE INFO ...;  RAISE LOG ...;
  RAISE EXCEPTION 'bad value %', n USING ERRCODE = 'P0001', HINT = 'fix it', DETAIL = '...';
  RAISE EXCEPTION USING ERRCODE = 'unique_violation';
  ASSERT n > 0, 'n must be positive';

  -- exceptions
  BEGIN
    INSERT ...;
  EXCEPTION
    WHEN unique_violation THEN RAISE NOTICE 'dup';
    WHEN foreign_key_violation OR check_violation THEN ...;
    WHEN SQLSTATE '23505' THEN ...;
    WHEN OTHERS THEN RAISE;                              -- re-raise
  END;                                                   -- sub-block = savepoint
END $$;
```

## Returning rows
```sql
CREATE FUNCTION f() RETURNS TABLE (id int, name text) LANGUAGE plpgsql AS $$
BEGIN
  RETURN QUERY SELECT t.id, t.name FROM t;
  RETURN QUERY EXECUTE format('SELECT ... FROM %I', tbl);
  FOR r IN SELECT ... LOOP id := r.id; name := r.name; RETURN NEXT; END LOOP;
  RETURN;
END $$;
SELECT * FROM f();
```

## Dynamic SQL (safe)
```sql
EXECUTE format('INSERT INTO %I.%I (a, b) VALUES (%L, %L)', schema_name, tbl, a, b);
EXECUTE 'SELECT count(*) FROM ' || quote_ident(tbl) INTO n;
EXECUTE 'SELECT * FROM t WHERE id = $1 AND x = $2' INTO r USING p1, p2;
-- %I identifier  %L literal  %s plain string  %% percent
```

## Procedures with transaction control
```sql
CREATE PROCEDURE batch_job() LANGUAGE plpgsql AS $$
DECLARE r record;
BEGIN
  FOR r IN SELECT id FROM big LOOP
    UPDATE big SET done = true WHERE id = r.id;
    IF r.id % 1000 = 0 THEN COMMIT; END IF;       -- ROLLBACK also allowed
  END LOOP;
END $$;
CALL batch_job();        -- must NOT be inside an explicit transaction block
```

## Cursors in PL/pgSQL
```sql
DECLARE cur CURSOR FOR SELECT ...;     cur2 CURSOR (p int) FOR SELECT ... WHERE x = p;
OPEN cur;  FETCH cur INTO r;  MOVE ...;  CLOSE cur;
OPEN refcur FOR EXECUTE ...;   RETURN refcur;
```

## Triggers
```sql
CREATE FUNCTION audit() RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
  -- TG_OP: INSERT|UPDATE|DELETE|TRUNCATE; TG_TABLE_NAME; TG_TABLE_SCHEMA; TG_WHEN; TG_LEVEL; TG_ARGV[]; OLD; NEW
  IF TG_OP = 'DELETE' THEN INSERT INTO audit_log(tbl, op, old_row) VALUES (TG_TABLE_NAME, TG_OP, to_jsonb(OLD)); RETURN OLD;
  ELSIF TG_OP = 'UPDATE' THEN INSERT INTO audit_log(tbl, op, old_row, new_row) VALUES (TG_TABLE_NAME, TG_OP, to_jsonb(OLD), to_jsonb(NEW)); RETURN NEW;
  ELSE INSERT INTO audit_log(tbl, op, new_row) VALUES (TG_TABLE_NAME, TG_OP, to_jsonb(NEW)); RETURN NEW; END IF;
END $$;
CREATE TRIGGER trg_audit AFTER INSERT OR UPDATE OR DELETE ON t FOR EACH ROW EXECUTE FUNCTION audit();
CREATE TRIGGER trg BEFORE UPDATE ON t FOR EACH ROW WHEN (OLD.* IS DISTINCT FROM NEW.*) EXECUTE FUNCTION f();
CREATE TRIGGER trg AFTER UPDATE OF col ON t FOR EACH ROW ...;
CREATE TRIGGER trg AFTER INSERT ON t REFERENCING NEW TABLE AS new_rows FOR EACH STATEMENT EXECUTE FUNCTION f();
CREATE TRIGGER trg INSTEAD OF INSERT ON view_v FOR EACH ROW EXECUTE FUNCTION f();
CREATE CONSTRAINT TRIGGER trg AFTER INSERT ON t DEFERRABLE INITIALLY DEFERRED FOR EACH ROW EXECUTE FUNCTION f();
ALTER TABLE t DISABLE TRIGGER trg;     ALTER TABLE t DISABLE TRIGGER USER;
SET session_replication_role = replica;      -- skip non-ALWAYS triggers & FK (superuser)
```
BEFORE ROW: return `NEW` (modified) / `NULL` to skip row. AFTER: return value ignored.

## Notify from trigger
```sql
PERFORM pg_notify('chan', json_build_object('id', NEW.id, 'op', TG_OP)::text);
```

## Handy generated-column & default patterns
```sql
created_at timestamptz NOT NULL DEFAULT now()
id uuid PRIMARY KEY DEFAULT gen_random_uuid()           -- or uuidv7() [18+]
total numeric GENERATED ALWAYS AS (qty * price) STORED
slug text GENERATED ALWAYS AS (lower(regexp_replace(name,'\s+','-','g'))) STORED
```

## Other languages
```sql
CREATE EXTENSION plpython3u;  CREATE FUNCTION py(x int) RETURNS int AS $$ return x*2 $$ LANGUAGE plpython3u;
CREATE EXTENSION plperl;      CREATE EXTENSION pltcl;
```

## Event triggers
```sql
CREATE FUNCTION no_drop() RETURNS event_trigger LANGUAGE plpgsql AS $$
BEGIN IF tg_tag = 'DROP TABLE' THEN RAISE EXCEPTION 'blocked'; END IF; END $$;
CREATE EVENT TRIGGER block_drop ON ddl_command_start WHEN TAG IN ('DROP TABLE') EXECUTE FUNCTION no_drop();
SELECT * FROM pg_event_trigger_ddl_commands();    -- inside ddl_command_end
SELECT * FROM pg_event_trigger_dropped_objects();  -- inside sql_drop
```
