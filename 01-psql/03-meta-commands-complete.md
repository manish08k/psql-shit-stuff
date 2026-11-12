# 03 — psql Meta-Commands (complete, PostgreSQL 18)

Verified against https://www.postgresql.org/docs/18/app-psql.html
Rules: a meta-command is a line starting with `\`. Several can't share a line with SQL except those that end a query (`\g`, `\gx`, `\gset`, `\gexec`, `\watch`, `\crosstabview`, `\bind`, …). Arguments are separated by whitespace. `\\` separates multiple meta-commands on one line. Arguments in `'single quotes'` are interpreted C-style; in `` `backticks` `` run a shell command and substitute output.

## Help / info
```
\?  [commands|options|variables]    psql help
\h  [SQL command]                    SQL syntax help (\h * = all)       \help
\copyright                           copyright
\errverbose                          repeat last error at max verbosity
\conninfo                            current connection info
```

## Connection / session
```
\c | \connect [-reuse-previous=on|off] [dbname [user] [host] [port] | conninfo]
\q | \quit
\password [user]
\encoding [enc]
\cd [dir]
\setenv name [value]                 set/unset environment variable
\getenv psql_var ENV_VAR             copy env var into psql var
\timing [on|off]                     time statements (ms; >1s also min:sec)
\restrict key  /  \unrestrict key    restricted mode (only \unrestrict allowed); used in pg_dump plain output
\! [command]                         run shell command (or subshell)
```

## Query buffer
```
\p | \print                          show buffer
\r | \reset                          clear buffer
\e | \edit [file [line]]             edit buffer/file in $PSQL_EDITOR/$EDITOR/$VISUAL
\ef [function [line]]                edit function definition
\ev [view [line]]                    edit view definition
\w | \write file                     write buffer to file     \w |command   pipe to shell
\;                                   add a literal ; to buffer without sending (multi-statement request)
```

## Execute
```
\g [(opt=val ...)] [file | |command]      run buffer (output to file/pipe)
\gx [(opt=val ...)] [file | |command]     run buffer with expanded output
\gdesc                                    describe result columns (names + types) without running
\gexec                                    run each cell of the result as SQL
\gset [prefix]                            store 1-row result into variables
\watch [i[nterval]=sec] [c[ount]=n] [m[in_rows]=n] [sec]    re-run until interrupted / count / min_rows not met
\crosstabview [colV [colH [colD [sortcolH]]]]               pivot grid output
\i | \include file                        run file
\ir | \include_relative file              run file relative to current script
```
`\g` per-query options: `\g (format=csv fieldsep=';' tuples_only=on) out.csv` (any `\pset` option)
`\watch` default interval 2s (variable `WATCH_INTERVAL`). Pager for watch: `PSQL_WATCH_PAGER` (e.g. `pspg --stream`).

## Conditionals
```
\if expression  /  \elif expression  /  \else  /  \endif
```
Expression: `true|false|on|off|yes|no|1|0` (case-insens.), or interpolated `:var`, `:{?var}`.

## Variables & input
```
\set [name [value ...]]               set; no args lists all
\unset name                           delete (control variables reset to default)
\prompt [text] name                   ask user
```
Interpolation: `:name` raw · `:'name'` quoted literal · `:"name"` quoted identifier · `:{?name}` TRUE/FALSE if set.

## Output control
```
\o | \out [file | |command]           redirect query output (no arg = back to stdout)
\echo [-n] text                       print to stdout
\qecho text                           print to \o target
\warn text                            print to stderr
\pset [option [value]]                print options (see 06-output-formatting.md)
\a                                    aligned ⇄ unaligned
\C [title]                            = \pset title
\f [sep]                              = \pset fieldsep
\H | \html                            HTML on/off
\t [on|off]                           = \pset tuples_only
\T attrs                              = \pset tableattr
\x [on|off|auto]                      = \pset expanded
\s [file]                             history to file/stdout
```

## COPY & large objects
```
\copy table [(cols)] FROM 'file'|PROGRAM 'cmd'|STDIN|PSTDIN [[WITH] (opt, ...)] [WHERE cond]
\copy {table [(cols)] | (query)} TO 'file'|PROGRAM 'cmd'|STDOUT|PSTDOUT [[WITH] (opt, ...)]
\lo_import file [comment]      \lo_export loid file      \lo_list[x+]  (alias \dl)      \lo_unlink loid
```

## Function / view source
```
\sf[+] function_desc          show function definition  (+ = line numbers)
\sv[+] view                   show view definition
\ef  \ev                      edit them
```

## Extended query protocol / prepared statements
```
\parse name                   prepare buffer as named statement
\bind [param ...]             bind params for next query:  SELECT $1+$2 \bind 1 2 \g
\bind_named name [param ...]  execute named statement
\close_prepared name          close prepared statement
```

## Pipeline mode
```
\startpipeline   \sendpipeline   \syncpipeline   \endpipeline   \flushrequest   \flush   \getresults [n]
```
(see 09-pipeline-extended-protocol.md)

## Describe family → see 04-describe-commands.md
`\d \da \dA \dAc \dAf \dAo \dAp \db \dc \dconfig \dC \dd \dD \ddp \dE \des \det \deu \dew \df \dF \dFd \dFp \dFt \dg \di \dl \dL \dm \dn \do \dO \dp \dP \drds \drg \ds \dRp \dRs \dt \dT \du \dv \dx \dX \dy \l \z`
