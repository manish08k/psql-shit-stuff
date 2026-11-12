# 07 — Import / Export with psql

## `\copy` (client-side: file on YOUR machine, runs with your OS permissions)
```
\copy table [(col, ...)] FROM 'file' | PROGRAM 'cmd' | STDIN | PSTDIN  [[WITH] (option [, ...])]  [WHERE condition]
\copy { table [(col, ...)] | (query) } TO 'file' | PROGRAM 'cmd' | STDOUT | PSTDOUT  [[WITH] (option [, ...])]
```
- Whole command must be on ONE line (no multi-line, no variable interpolation inside the quoted parts except via `:'var'` outside).
- `STDIN`/`STDOUT` = the COPY data stream within a script; `PSTDIN`/`PSTDOUT` = psql's own stdin/stdout.
- `WHERE` filter on FROM is available in 18.

### Import
```
\copy users FROM 'users.csv' WITH (FORMAT csv, HEADER true)
\copy users (id, name, email) FROM 'users.csv' CSV HEADER
\copy users FROM 'users.tsv'                                    -- default text format, tab-delimited
\copy users FROM 'u.csv' WITH (FORMAT csv, HEADER, DELIMITER ';', NULL 'NA', QUOTE '"', ENCODING 'LATIN1')
\copy users FROM program 'gunzip -c users.csv.gz' WITH (FORMAT csv, HEADER)
\copy users FROM 'u.csv' WITH (FORMAT csv, HEADER, FORCE_NULL (email), ON_ERROR ignore, LOG_VERBOSITY verbose)
\copy users FROM 'u.csv' WITH (FORMAT csv, HEADER) WHERE active
\copy t FROM STDIN WITH (FORMAT csv)
1,alice
2,bob
\.
```

### Export
```
\copy users TO 'users.csv' WITH (FORMAT csv, HEADER)
\copy (SELECT id, name FROM users WHERE active) TO 'active.csv' CSV HEADER
\copy (SELECT * FROM t) TO program 'gzip > t.csv.gz' CSV HEADER
\copy t TO STDOUT WITH (FORMAT csv, HEADER)
\copy (SELECT ...) TO 'f.tsv'
\copy t TO 'f.bin' WITH (FORMAT binary)
```

### COPY options
`FORMAT text|csv|binary` · `FREEZE` · `DELIMITER 'c'` · `NULL 'str'` · `DEFAULT 'str'` · `HEADER true|false|MATCH` · `QUOTE 'c'` · `ESCAPE 'c'` · `FORCE_QUOTE (cols)|*` · `FORCE_NOT_NULL (cols)|*` · `FORCE_NULL (cols)|*` · `ON_ERROR stop|ignore` · `REJECT_LIMIT n` · `LOG_VERBOSITY default|verbose|silent` · `ENCODING 'enc'`

## Server-side `COPY` (file on the DB SERVER; needs superuser or pg_read_server_files / pg_write_server_files)
```sql
COPY t FROM '/var/lib/postgresql/data/f.csv' WITH (FORMAT csv, HEADER);
COPY t TO '/tmp/out.csv' WITH (FORMAT csv, HEADER);
COPY t FROM PROGRAM 'curl -s https://host/data.csv' WITH (FORMAT csv, HEADER);   -- needs pg_execute_server_program
```

## Fast loading recipe
```sql
BEGIN;
CREATE TABLE stage (LIKE target INCLUDING DEFAULTS);        -- no indexes/constraints
\copy stage FROM 'big.csv' CSV HEADER
INSERT INTO target SELECT * FROM stage ON CONFLICT (id) DO NOTHING;
DROP TABLE stage;
COMMIT;
ANALYZE target;
```
Bulk-load speedups: load into empty table in same tx as CREATE/TRUNCATE (+ `FREEZE`), drop indexes then recreate, `SET maintenance_work_mem='2GB'`, `ALTER TABLE t SET UNLOGGED` (then `SET LOGGED`).

## Load a CSV with unknown columns (all text → cast later)
```sql
CREATE TABLE raw (a text, b text, c text);
\copy raw FROM 'f.csv' CSV HEADER
```

## Copy between databases / servers (no temp file)
```bash
psql -h src -d db -c "\copy (SELECT * FROM t) TO STDOUT" | psql -h dst -d db -c "\copy t FROM STDIN"
psql -h src -d db -c "COPY t TO STDOUT WITH (FORMAT binary)" | psql -h dst -d db -c "COPY t FROM STDIN WITH (FORMAT binary)"
```

## psql-only export without `\copy`
```bash
psql --csv -c "SELECT ..." > out.csv
psql -At -F, -c "SELECT ..." > out.txt
psql -c "SELECT json_agg(t) FROM t" -At > out.json
```
```
\o out.csv   \pset format csv   SELECT ...;   \o
```

## Large objects
```
\lo_import '/path/photo.jpg' 'comment'       →  lo_import 16409   (note the OID)
\lo_export 16409 '/path/copy.jpg'
\lo_list    \lo_list+   \dl
\lo_unlink 16409
\gset after: SELECT lo_import('/server/path/f') AS oid \gset  (server-side function, server file)
```
Prefer `bytea` columns for small files; large objects for streaming/huge files.

## Import text file contents into a variable / column
```sql
\set content `cat note.txt`
INSERT INTO docs (body) VALUES (:'content');
-- bytea from file via server function (superuser): pg_read_binary_file('/path')
```

## Run a SQL dump / restore plain files
```bash
psql -d db -f dump.sql
psql -v ON_ERROR_STOP=1 -1 -d db -f dump.sql          # all-or-nothing
psql -q -d db -f dump.sql 2> errors.log
```
(Plain pg_dump output may start with `\restrict <key>` and end with `\unrestrict <key>` — normal in 18.)
