# Cheatsheet — Catalog & Introspection Queries

## Lists
```sql
SELECT datname FROM pg_database WHERE NOT datistemplate;
SELECT schema_name FROM information_schema.schemata;
SELECT table_schema, table_name FROM information_schema.tables WHERE table_schema NOT IN ('pg_catalog','information_schema');
SELECT * FROM information_schema.columns WHERE table_name = 't';
SELECT column_name, data_type, is_nullable, column_default FROM information_schema.columns WHERE table_schema='public' AND table_name='t' ORDER BY ordinal_position;
SELECT * FROM pg_indexes WHERE tablename = 't';
SELECT conname, contype, pg_get_constraintdef(oid) FROM pg_constraint WHERE conrelid = 't'::regclass;
SELECT * FROM pg_views WHERE schemaname='public';
SELECT * FROM pg_matviews;
SELECT * FROM pg_sequences;
SELECT proname, pg_get_functiondef(oid) FROM pg_proc WHERE pronamespace = 'public'::regnamespace;
SELECT tgname, pg_get_triggerdef(oid) FROM pg_trigger WHERE NOT tgisinternal;
SELECT * FROM pg_policies;
SELECT * FROM pg_extension;
SELECT * FROM pg_tablespace;
SELECT rolname, rolsuper, rolcreatedb, rolcanlogin FROM pg_roles;
SELECT * FROM pg_user;   SELECT * FROM pg_stat_activity;
SELECT * FROM pg_settings;   SELECT * FROM pg_available_extensions;
SELECT enumlabel FROM pg_enum WHERE enumtypid = 'mood'::regtype ORDER BY enumsortorder;
SELECT pg_get_viewdef('v'::regclass, true);
SELECT pg_get_indexdef('idx'::regclass);
SELECT pg_get_constraintdef(oid) FROM pg_constraint WHERE conname = 'c';
```

## Foreign keys
```sql
SELECT conrelid::regclass AS tbl, conname, confrelid::regclass AS ref, pg_get_constraintdef(oid)
FROM pg_constraint WHERE contype = 'f';
```

## Table sizes
```sql
SELECT relname, pg_size_pretty(pg_total_relation_size(oid)) AS total, pg_size_pretty(pg_relation_size(oid)) AS table, pg_size_pretty(pg_indexes_size(oid)) AS idx
FROM pg_class WHERE relkind = 'r' AND relnamespace = 'public'::regnamespace ORDER BY pg_total_relation_size(oid) DESC LIMIT 20;
SELECT pg_size_pretty(pg_database_size(current_database()));
SELECT datname, pg_size_pretty(pg_database_size(datname)) FROM pg_database ORDER BY pg_database_size(datname) DESC;
SELECT spcname, pg_size_pretty(pg_tablespace_size(spcname)) FROM pg_tablespace;
```

## Columns of a table with types (compact)
```sql
SELECT a.attname, format_type(a.atttypid, a.atttypmod), a.attnotnull
FROM pg_attribute a WHERE a.attrelid = 't'::regclass AND a.attnum > 0 AND NOT a.attisdropped;
```

## Bloat / unused indexes / missing indexes
```sql
SELECT schemaname, relname, indexrelname, idx_scan, pg_size_pretty(pg_relation_size(indexrelid)) FROM pg_stat_user_indexes WHERE idx_scan = 0 ORDER BY pg_relation_size(indexrelid) DESC;
SELECT relname, seq_scan, seq_tup_read, idx_scan FROM pg_stat_user_tables WHERE seq_scan > idx_scan ORDER BY seq_tup_read DESC;
SELECT relname, n_dead_tup, n_live_tup, round(100.0*n_dead_tup/nullif(n_live_tup+n_dead_tup,0),1) AS dead_pct FROM pg_stat_user_tables ORDER BY n_dead_tup DESC;
```

## Duplicate rows
```sql
SELECT col, count(*) FROM t GROUP BY col HAVING count(*) > 1;
DELETE FROM t a USING t b WHERE a.ctid < b.ctid AND a.col = b.col;
```

## Generate DDL helpers (use with \gexec in psql)
```sql
SELECT format('GRANT SELECT ON %I.%I TO readonly', schemaname, tablename) FROM pg_tables WHERE schemaname='public' \gexec
SELECT format('ALTER TABLE %I OWNER TO app', tablename) FROM pg_tables WHERE schemaname='public' \gexec
```

## Common JSON
```sql
SELECT jsonb_pretty(data);   SELECT jsonb_each(data);   SELECT jsonb_array_elements(data->'items');
UPDATE t SET data = jsonb_set(data, '{a,b}', '1');   UPDATE t SET data = data || '{"k":1}';   UPDATE t SET data = data - 'k';
```

## Data type quick list
```
smallint integer bigint numeric(p,s) real double precision serial bigserial
text varchar(n) char(n) boolean bytea uuid json jsonb xml
date time timetz timestamp timestamptz interval
inet cidr macaddr point line box circle polygon
int4range tsrange daterange tsvector tsquery money oid
type[] (arrays)   enum   composite   domain
```
