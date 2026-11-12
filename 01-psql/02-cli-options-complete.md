# 02 — psql Command-Line Options (complete, PostgreSQL 18)

Verified against https://www.postgresql.org/docs/18/app-psql.html

## Connection
| Option | Meaning |
|---|---|
| `-h host`, `--host=host` | server host; leading `/` = Unix-socket directory |
| `-p port`, `--port=port` | TCP port or socket file extension |
| `-U user`, `--username=user` | connect as this user |
| `-d dbname`, `--dbname=dbname` | database; may be a full connection string |
| `-w`, `--no-password` | never prompt for password |
| `-W`, `--password` | force password prompt |

## What to run
| Option | Meaning |
|---|---|
| `-c cmd`, `--command=cmd` | run one SQL command or ONE backslash command; repeatable |
| `-f file`, `--file=file` | read commands from file (`-` = stdin); repeatable |
| `-1`, `--single-transaction` | wrap all `-c`/`-f` in one BEGIN/COMMIT |
| `-l`, `--list` | list databases and exit |
| `-v name=val`, `--set=`, `--variable=` | set a psql variable (omit `=` to unset) |
| `-X`, `--no-psqlrc` | skip system and user psqlrc |
| `-V`, `--version` | print version and exit |
| `-?`, `--help[=topic]` | help; topic = `options` (default), `commands`, `variables` |

## Echo / logging / interaction
| Option | Meaning |
|---|---|
| `-a`, `--echo-all` | print all non-empty input lines as read (not interactive) |
| `-b`, `--echo-errors` | print failed SQL commands to stderr |
| `-e`, `--echo-queries` | echo SQL sent to server |
| `-E`, `--echo-hidden` | show queries generated internally by `\d` etc. |
| `-q`, `--quiet` | no welcome / informational messages |
| `-L file`, `--log-file=file` | also write all query output to file |
| `-o file`, `--output=file` | write all query output to file |
| `-s`, `--single-step` | confirm each command before sending |
| `-S`, `--single-line` | newline ends an SQL command (like `;`) |
| `-n`, `--no-readline` | disable line editing/history (useful when pasting text with tabs) |

## Output format
| Option | Meaning |
|---|---|
| `-A`, `--no-align` | unaligned output |
| `-t`, `--tuples-only` | no headers, no row-count footer |
| `-x`, `--expanded` | expanded (vertical) output |
| `--csv` | CSV output |
| `-H`, `--html` | HTML tables |
| `-T attrs`, `--table-attr=attrs` | attributes for HTML `<table>` |
| `-P name=value`, `--pset=` | set a `\pset` option (e.g. `-P format=latex`, `-P null='(null)'`) |
| `-F sep`, `--field-separator=sep` | field separator for unaligned (default `\|`) |
| `-R sep`, `--record-separator=sep` | record separator for unaligned (default newline) |
| `-z`, `--field-separator-zero` | NUL byte as field separator |
| `-0`, `--record-separator-zero` | NUL byte as record separator |

## Handy combos
```bash
psql -XAtq                       # scripting: no rc, unaligned, tuples-only, quiet
psql -XAtqF, -c "SELECT a,b FROM t"         # a,b per line
psql -X --csv -c "SELECT ..." > out.csv
psql -x -c "SELECT * FROM t LIMIT 1"        # one row vertically
psql -E mydb                                # see how \d works under the hood
psql -b -f script.sql                       # show failed commands only
psql -a -f script.sql                       # echo whole script as it runs
psql -e -f script.sql                       # echo each statement sent
psql -s mydb                                # step through statements
psql -P pager=off -P border=2 mydb
psql -v ON_ERROR_STOP=1 -v schema=app -f x.sql
psql -L session.log mydb
psql -o results.txt -c "SELECT ..." mydb
```
