# 05 — Variables, Special Variables, Scripting

## Setting & using variables
```
\set name value             \set name           (empty string)        \set          (list all)
\unset name                 \set name 'two words'                     \set a :b     (copy)
psql -v name=value          psql --set=name=value
\set content `cat file.txt`                    -- backticks: shell output
\getenv home HOME                               -- env var → psql var
\setenv PGOPTIONS '-c statement_timeout=5s'    -- psql var → env (for next \c)
\prompt 'Enter table: ' tbl                     -- ask user
```
Interpolation (not performed inside quoted SQL strings):
| Form | Result |
|---|---|
| `:name` | raw text |
| `:'name'` | SQL literal, properly quoted |
| `:"name"` | SQL identifier, properly quoted |
| `:{?name}` | `TRUE` / `FALSE` — is it set? |
```sql
\set tbl users
\set id 42
SELECT * FROM :"tbl" WHERE id = :id;
\set name O'Brien
INSERT INTO t(name) VALUES (:'name');
```
Variable names: letters, digits, underscore. Unset names are not replaced.

## Special variables (control psql behavior)
| Variable | Values | Meaning |
|---|---|---|
| `AUTOCOMMIT` | `on` (default) / `off` | off = implicit BEGIN; must COMMIT/ROLLBACK |
| `COMP_KEYWORD_CASE` | `lower` `upper` `preserve-lower` `preserve-upper`(default) | tab-complete keyword case |
| `ECHO` | `none`(default) `errors` `queries` `all` | echo input (-b / -e / -a) |
| `ECHO_HIDDEN` | `on` `off` `noexec` | show queries behind `\d` |
| `FETCH_COUNT` | integer (0 = off) | fetch results in batches via cursor (100–1000 typical) for huge results |
| `HIDE_TABLEAM` | `true/false` | hide table access method in `\d` |
| `HIDE_TOAST_COMPRESSION` | `true/false` | hide TOAST compression in `\d+` |
| `HISTCONTROL` | `none`(default) `ignorespace` `ignoredups` `ignoreboth` | history filtering |
| `HISTFILE` | path | history file (`\set HISTFILE ~/.psql_history-:DBNAME` for per-db) |
| `HISTSIZE` | int (default 500, negative = unlimited) | history length |
| `IGNOREEOF` | int (default 0) | Ctrl-D presses needed to quit |
| `ON_ERROR_ROLLBACK` | `on` `interactive` `off`(default) | savepoint around each statement so errors don't abort tx |
| `ON_ERROR_STOP` | `on` / `off` | stop script on first error (exit 3) |
| `PROMPT1` `PROMPT2` `PROMPT3` | string | prompts (see 08) |
| `QUIET` | `on/off` | same as -q |
| `SHOW_ALL_RESULTS` | `on`(default) / `off` | show results of every `\;` statement vs only last |
| `SHOW_CONTEXT` | `never` `errors`(default) `always` | CONTEXT lines |
| `SINGLELINE` / `SINGLESTEP` | `on/off` | -S / -s |
| `VERBOSITY` | `default` `verbose` `terse` `sqlstate` | error detail |
| `WATCH_INTERVAL` | seconds (default 2) | default `\watch` interval |

## Read-only / state variables
| Variable | Contains |
|---|---|
| `DBNAME` `USER` `HOST` `PORT` `SERVICE` `ENCODING` | current connection (DBNAME/ENCODING writable via \c / \encoding) |
| `ERROR` | `true`/`false` – did last query fail |
| `SQLSTATE` | last query's SQLSTATE (`00000` ok) |
| `LAST_ERROR_MESSAGE` `LAST_ERROR_SQLSTATE` | last error in session |
| `ROW_COUNT` | rows affected/returned by last query |
| `SHELL_ERROR` `SHELL_EXIT_CODE` | outcome of last shell command (`\!`, backticks, `\g |cmd`, `\copy program`) |
| `SERVER_VERSION_NAME` `SERVER_VERSION_NUM` | e.g. `18.6` / `180006` |
| `VERSION` `VERSION_NAME` `VERSION_NUM` | psql's own version |
| `LASTOID` | always 0 on servers ≥ 12 |
| `PIPELINE_COMMAND_COUNT` `PIPELINE_RESULT_COUNT` `PIPELINE_SYNC_COUNT` | pipeline counters |

Control variables reject invalid values; `on`/`off` accept `true`/`false`; `\set VAR` with no value sets `on`.

## Error handling in scripts
```sql
\set ON_ERROR_STOP on                       -- stop at first error (exit code 3)
\set ON_ERROR_ROLLBACK interactive          -- in interactive sessions, don't lose tx on typos
\set VERBOSITY verbose
SELECT 1/0;
\echo :ERROR :SQLSTATE :LAST_ERROR_MESSAGE
\if :ERROR
  \echo 'it failed'
  \q
\endif
```
Prefer shell: `psql -v ON_ERROR_STOP=1 -f x.sql; echo $?`

## Conditionals
```
\if :{?env}
  \echo env is set
\elif :is_prod
  \echo prod
\else
  \echo none
\endif
\set is_prod true
\if :is_prod ... \endif
```
In an inactive branch, commands aren't executed (prompt shows `@` for `%R`). Valid booleans: `true false on off yes no 1 0`.

## `\gset` — query result → variables
```sql
SELECT count(*) AS n, max(id) AS maxid FROM users \gset
\echo :n :maxid
SELECT current_database() AS db \gset cfg_       -- prefix
\echo :cfg_db
SELECT 1 AS x \gset
\if :x ... \endif
```
Must return exactly 1 row (error on 0 or >1). NULL → variable unset.

## `\gexec` — generate SQL then run it
```sql
SELECT format('VACUUM ANALYZE %I.%I', schemaname, relname) FROM pg_stat_user_tables WHERE n_dead_tup > 10000 \gexec
SELECT format('GRANT SELECT ON %I.%I TO readonly', schemaname, tablename) FROM pg_tables WHERE schemaname='public' \gexec
SELECT format('ALTER TABLE %I OWNER TO app', tablename) FROM pg_tables WHERE schemaname='app' \gexec
SELECT format('CREATE INDEX CONCURRENTLY ON %I (%I)', 't', column_name) FROM information_schema.columns WHERE table_name='t' AND column_name LIKE '%_id' \gexec
SELECT format('DROP TABLE %I', tablename) FROM pg_tables WHERE tablename LIKE 'tmp\_%' \gexec
```
Each cell of each row runs as its own statement; empty/NULL cells are skipped. Use `\echo`/`\g` first to preview: run the SELECT with `;` before switching to `\gexec`.

## `\gdesc`
```sql
SELECT id, name, created FROM users \gdesc        -- column names + types, query not executed
```

## `\watch`
```sql
SELECT now(), count(*) FROM pg_stat_activity \watch 1
SELECT * FROM jobs WHERE status='run' \watch interval=5 count=12
SELECT * FROM queue \watch i=1 m=1                 -- stop when fewer than 1 row returned
```

## `\g` options and redirects
```sql
SELECT * FROM t \g out.txt                        -- to file
SELECT * FROM t \g |less -S                       -- to pipe
SELECT * FROM t \g (format=csv) out.csv
SELECT * FROM t \gx
SELECT * FROM t \g (tuples_only=on fieldsep='|' format=unaligned) out.psv
```

## Shell integration
```
\! ls -l
\! sh
\cd /tmp
\set x `date +%F`
\o | gzip > out.gz
\w | pbcopy              -- write buffer to clipboard command
\i /abs/path/file.sql    \ir relative.sql       (inside nested includes use \ir)
```
`SHELL_ERROR` / `SHELL_EXIT_CODE` hold result of the last shell command.

## Script template
```sql
\set ON_ERROR_STOP on
\set QUIET on
\timing off
\pset pager off
\if :{?target_schema} \else \set target_schema public \endif
\echo Running against :DBNAME as :USER on :HOST
BEGIN;
SET LOCAL lock_timeout = '3s';
SET LOCAL statement_timeout = '5min';
SET LOCAL search_path = :"target_schema", pg_catalog;
-- ... work ...
COMMIT;
\echo done
```
Run: `psql -X -d mydb -v target_schema=app -f script.sql`
