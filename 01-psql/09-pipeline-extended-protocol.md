# 09 — Extended Query Protocol & Pipeline Mode (psql 16+/18)

## Parameterised queries without PREPARE
```sql
SELECT $1 + $2 \bind 3 4 \g                                -- 7
SELECT * FROM users WHERE id = $1 AND status = $2 \bind 42 active \g
SELECT * FROM t WHERE name = $1 \bind 'O''Brien' \g
INSERT INTO t(a, b) VALUES ($1, $2) \bind 1 x \gx
SELECT $1::int[] \bind '{1,2,3}' \g
```
Note: parameters are sent as text, so `\bind NULL` sends the string "NULL", not a SQL NULL. To test NULL handling, write `NULLIF($1, 'NULL')` in the query.
`\bind` sends the query via the extended protocol (parameters sent separately → no quoting problems, SQL-injection-safe test of app queries).
Works with `\g`, `\gx`, `\gset`, `\gexec`, `\watch` as terminators.

## Named prepared statements
```sql
SELECT * FROM users WHERE id = $1 \parse by_id          -- create statement "by_id" from buffer
\bind_named by_id 42 \g                                  -- run it
\bind_named by_id 43 \gx
\close_prepared by_id                                     -- drop it
```
(Compare SQL-level: `PREPARE q AS ...; EXECUTE q(1); DEALLOCATE q;`)

## Pipeline mode
Send many queries without waiting for each reply (reduces round-trips).
```
\startpipeline          begin pipeline
\sendpipeline           append the query buffer to the pipeline (query must use \bind / \bind_named)
\syncpipeline           send a Sync message (commit point) without ending pipeline
\flushrequest           ask server to flush results so far
\flush                  push unsent client data to server manually
\getresults [n]         read pending results (n = how many; 0/omitted = all) without ending pipeline
\endpipeline            end pipeline (sync + collect everything)
```
Example:
```
\startpipeline
SELECT * FROM pg_class;
SELECT 1 \bind \sendpipeline
INSERT INTO log(msg) VALUES ($1) \bind 'hello' \sendpipeline
\flushrequest
\getresults
\endpipeline
```
Notes:
- `COPY` is not supported in pipeline mode.
- Prompt escape `%P` shows pipeline state (`off` / `on` / `abort`).
- Counters: variables `PIPELINE_COMMAND_COUNT`, `PIPELINE_RESULT_COUNT`, `PIPELINE_SYNC_COUNT`.
- An error aborts the pipeline until the next Sync (`\syncpipeline` / `\endpipeline`).

## Multi-statement in one request (simple protocol)
```
SELECT 1 \; SELECT 2 \; SELECT 3;           -- sent as ONE request (implicit single transaction)
\set SHOW_ALL_RESULTS off                    -- show only the last result
```
Without `\;`, each `;` sends a separate request.

## Restricted mode (security for untrusted scripts)
```
\restrict mykey          -- from now only \unrestrict mykey is allowed (blocks \!, \i, \copy program, etc.)
... untrusted SQL ...
\unrestrict mykey
```
pg_dump / pg_dumpall / pg_restore plain output includes these markers; supply a key on restore as needed.
