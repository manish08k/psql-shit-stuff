# 01 — Starting psql & Connecting

## Invocation
```bash
psql [option...] [dbname [username]]
psql                                      # uses PGHOST/PGUSER/PGDATABASE or OS user defaults
psql mydb
psql mydb alice
psql -h 10.0.0.5 -p 5433 -U alice -d mydb
psql -h /var/run/postgresql mydb           # leading slash = Unix socket directory
```

## Connection string forms (`-d` accepts either)
```bash
psql "postgresql://alice:secret@host:5432/mydb?sslmode=require&application_name=psql"
psql "postgres://alice@host/mydb"
psql "host=host port=5432 dbname=mydb user=alice sslmode=verify-full sslrootcert=root.crt"
psql "service=prod"                        # from ~/.pg_service.conf
psql "postgresql://h1:5432,h2:5432/db?target_session_attrs=read-write"   # multi-host failover
psql "postgresql://host/db?options=-c%20search_path%3Dapp"
```
Useful conninfo keys: `host hostaddr port dbname user password passfile connect_timeout application_name sslmode sslcert sslkey sslrootcert sslcrl target_session_attrs options service channel_binding gssencmode keepalives keepalives_idle`
`sslmode`: `disable allow prefer require verify-ca verify-full`

## Passwords (don't put them on the command line)
```bash
# 1) ~/.pgpass  (chmod 0600)  — one entry per line, * allowed
hostname:port:database:username:password
localhost:5432:*:alice:s3cret
# override file: PGPASSFILE=/path/to/pgpass
# 2) env var (visible to other processes; avoid on shared hosts)
PGPASSWORD=s3cret psql -h host -U alice mydb
# 3) prompt
psql -W -U alice mydb            # force prompt     |     -w never prompt (fail instead)
```

## Service file `~/.pg_service.conf`
```ini
[prod]
host=db.example.com
port=5432
dbname=app
user=alice
sslmode=verify-full
```
`psql service=prod`  ·  env: `PGSERVICE=prod`  ·  file override: `PGSERVICEFILE`

## Environment variables read by psql / libpq
```
PGHOST  PGPORT  PGDATABASE  PGUSER  PGPASSWORD  PGPASSFILE  PGSERVICE  PGSERVICEFILE
PGSSLMODE  PGSSLROOTCERT  PGSSLCERT  PGSSLKEY  PGCONNECT_TIMEOUT  PGOPTIONS  PGAPPNAME  PGCLIENTENCODING  PGTZ
PSQLRC  PSQL_HISTORY  PSQL_PAGER / PAGER  PSQL_WATCH_PAGER  PSQL_EDITOR / EDITOR / VISUAL
PSQL_EDITOR_LINENUMBER_ARG  COLUMNS  PG_COLOR (always|auto|never)  SHELL  TMPDIR
```
Example: `PGOPTIONS='-c statement_timeout=5s -c search_path=app' psql mydb`

## Switching connections inside psql
```
\c                                  reconnect to same
\c otherdb                          change database
\c otherdb bob                      change db + user
\c otherdb bob host 5433            full
\c -reuse-previous=on otherdb       reuse previous params (e.g. host/port/sslmode)
\c "host=h dbname=d user=u"         conninfo string
\conninfo                           show current connection details
\password [user]                    change password safely (hashed client-side)
\encoding [name]                    show / set client encoding
```

## Ways to run SQL
```bash
psql -c "SELECT now();"                                  # one command (SQL or ONE backslash command)
psql -c "SELECT 1" -c "SELECT 2"                         # repeatable; each runs separately
psql -c '\dt'                                            # backslash via -c  (can't mix with SQL in one -c)
psql -f script.sql                                       # file (repeatable; "-f -" = stdin)
psql < script.sql                                        # stdin
echo "SELECT 1;" | psql mydb
psql mydb <<'SQL'                                        # heredoc (quote 'SQL' to stop shell expansion)
  CREATE TABLE t(id int);
  INSERT INTO t VALUES (1);
SQL
psql -1 -f migrate.sql                                   # single transaction (-1 / --single-transaction)
psql -v ON_ERROR_STOP=1 -f migrate.sql                   # stop at first error
psql -X -q -At -c "SELECT count(*) FROM t"               # script-friendly: no rc, quiet, unaligned, tuples-only
psql -f a.sql -f b.sql                                   # several files in order
```
Notes:
- Multiple `-c`/`-f` mixed are run in the order given.
- With `-1`, a failed command aborts the whole transaction (`ON_ERROR_STOP` recommended).
- Before 9.6, `-c` implied `-X` (skip psqlrc). That is no longer true, so add `-X` yourself for reproducible scripts.

## Exit status
| Code | Meaning |
|---|---|
| 0 | finished normally |
| 1 | fatal psql error (out of memory, file not found, …) |
| 2 | connection to server failed and session was not interactive |
| 3 | error in a script **and** `ON_ERROR_STOP` was set |
```bash
psql -v ON_ERROR_STOP=1 -f x.sql || echo "failed: $?"
```

## Script-friendly one-liners
```bash
psql -XAtqc "SELECT datname FROM pg_database WHERE NOT datistemplate"    # list, one per line
psql -XAtqc "SELECT count(*) FROM users" mydb                             # scalar to shell var
N=$(psql -XAtqc "SELECT count(*) FROM users" mydb)
psql -X --csv -c "SELECT * FROM t" > out.csv
psql -X -F $'\t' -At -c "SELECT a,b FROM t" > out.tsv
psql -X -c "\copy t TO 'out.csv' CSV HEADER"
for db in $(psql -XAtqc "SELECT datname FROM pg_database WHERE NOT datistemplate"); do psql -X -d "$db" -c "VACUUM ANALYZE"; done
```

## Interactive basics
```
mydb=>  normal user prompt      mydb=#  superuser prompt
mydb-#  statement continues     mydb(#  open parenthesis      mydb'#  open quote
;  or  \g  ends statement       Ctrl-C cancels query / clears buffer      Ctrl-D quits (empty line)
\q  or  \quit  quits
```
