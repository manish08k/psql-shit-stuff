# 09 — Cursors, Prepared Statements, LISTEN/NOTIFY

## Cursors
```sql
BEGIN;
DECLARE cur [BINARY] [INSENSITIVE] [NO] SCROLL CURSOR [WITH|WITHOUT HOLD] FOR SELECT ...;
FETCH NEXT FROM cur;
FETCH 10 FROM cur;       FETCH FORWARD 10 FROM cur;      FETCH ALL FROM cur;
FETCH PRIOR FROM cur;    FETCH FIRST FROM cur;           FETCH LAST FROM cur;
FETCH ABSOLUTE 5 FROM cur;                               FETCH RELATIVE -2 FROM cur;
FETCH BACKWARD 3 FROM cur;
MOVE FORWARD 5 IN cur;   MOVE BACKWARD ALL IN cur;       -- same directions as FETCH, no rows returned
CLOSE cur;               CLOSE ALL;
COMMIT;
SELECT * FROM pg_cursors;
```

## Prepared statements
```sql
PREPARE q (int, text) AS SELECT * FROM t WHERE id = $1 AND name = $2;
EXECUTE q(1, 'x');
DEALLOCATE q;           DEALLOCATE ALL;
SELECT * FROM pg_prepared_statements;
SET plan_cache_mode = auto | force_generic_plan | force_custom_plan;
```

## LISTEN / NOTIFY
```sql
LISTEN channel;
NOTIFY channel, 'payload';
SELECT pg_notify('channel', 'payload');
UNLISTEN channel;       UNLISTEN *;
SELECT pg_listening_channels();
```

## DO (anonymous block)
```sql
DO $$ BEGIN RAISE NOTICE 'hi'; END $$;
DO LANGUAGE plpgsql $$ DECLARE r record; BEGIN FOR r IN SELECT ... LOOP ... END LOOP; END $$;
```

## CALL
```sql
CALL proc(1, 'x');
CALL proc(a => 1, b => 'x');
```
