# psql — One-Page Quick Card

## Connect
```bash
psql -h host -p 5432 -U user -d db
psql "postgresql://user@host:5432/db?sslmode=require"
psql -l                                     # list databases
```
Inside psql: `\c db [user]`  `\conninfo`  `\q`

## Look around
```
\l+            databases            \dn+           schemas
\dt+           tables               \d+ t          describe table
\di+  \dv+  \dm+  \ds+              indexes / views / matviews / sequences
\df+ name      functions            \sf func       show function source
\du            roles                \dx            extensions
\dp t          privileges           \dconfig       non-default settings
\dt *.*        every table          \dt sales.*    schema's tables
```

## Run things
```
\i file.sql        run file               \ir rel.sql     run relative to current script
\e                 edit buffer in $EDITOR \ef func        edit function
\g                 run buffer             \gx             run, expanded
\watch 2           repeat every 2s        \timing         time each query
\! ls              shell                  \o file / \o    redirect output / stop
```

## Output
```
\x auto            expanded when wide     \pset null '∅'
\pset format csv   csv output             \pset pager off
\t                 rows only (no header)  \a              aligned/unaligned toggle
```

## Data in/out (client side)
```
\copy t FROM 'f.csv' CSV HEADER
\copy (SELECT * FROM t) TO 'out.csv' CSV HEADER
```

## Variables & scripting
```
\set v 5           SELECT :v;   :'v' (literal)   :"v" (identifier)
\gset              result → variables             \gexec   run each result cell as SQL
\if :{?v} ... \else ... \endif
\set ON_ERROR_STOP on
```

## Help
```
\?   \? variables   \? options       \h CREATE TABLE      psql --help
```

## Exit codes
`0` ok · `1` fatal psql error · `2` connection failed (non-interactive) · `3` script error with ON_ERROR_STOP
