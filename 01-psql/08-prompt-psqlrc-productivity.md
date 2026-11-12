# 08 — Prompts, psqlrc, History, Editing, Productivity

## Prompt escapes (PROMPT1, PROMPT2, PROMPT3)
Defaults: `PROMPT1 = PROMPT2 = '%/%R%x%# '` · `PROMPT3 = '>> '`
| Escape | Meaning |
|---|---|
| `%M` | full host name (or `[local]`, `[local:/dir]`) |
| `%m` | host name up to first dot (or `[local]`) |
| `%>` | port |
| `%n` | session user |
| `%s` | service name |
| `%/` | current database |
| `%~` | like `%/` but `~` for default database |
| `%#` | `#` superuser, `>` normal |
| `%p` | backend PID |
| `%P` | pipeline status: `off` `on` `abort` |
| `%R` | PROMPT1: `=` normal · `@` inactive `\if` branch · `^` single-line mode · `!` disconnected. PROMPT2: `-` continue · `*` in comment · `'` `"` `$` in quote · `(` open paren |
| `%x` | tx status: empty (none) · `*` in tx · `!` failed tx · `?` unknown |
| `%l` | line number within statement |
| `%NNN` | char by octal code |
| `%:name:` | value of psql variable |
| `` %`cmd` `` | output of shell command |
| `%[ … %]` | wrap non-printing (colour) sequences |
| `%w` | whitespace same width as PROMPT1 (align PROMPT2) |
| `%%` | literal % |

```
\set PROMPT1 '%n@%m:%> %/%R%x%# '
\set PROMPT1 '%[%033[1;32m%]%n@%/%[%033[0m%]%R%x%# '                         -- coloured
\set PROMPT1 '%[%033[1;31m%]%/%[%033[0m%]%R%x%# '                              -- red db name
\set PROMPT2 '%w%R '
\set PROMPT1 '(%`date +%H:%M`) %n@%/%R%x%# '
\set PROMPT1 '%M:%> %n@%/%R%x%# '                                              -- full host:port
```
Tip: coloured prod prompt in `~/.psqlrc` → `\if :{?...}` per-host rc via `~/.psqlrc-18`.

## psqlrc files (read after connecting unless `-X`)
Order: system `psqlrc` (dir from `pg_config --sysconfdir` or `$PGSYSCONFDIR`) → `~/.psqlrc` (Windows: `%APPDATA%\postgresql\psqlrc.conf`).
Version-specific files override unversioned: `~/.psqlrc-18`, `~/.psqlrc-18.6`.
Override with `PSQLRC=/path/rc`.

### Recommended `~/.psqlrc`
```
\set QUIET 1
\pset null '∅'
\pset linestyle unicode
\pset border 2
\pset unicode_border_linestyle single
\x auto
\timing on
\set COMP_KEYWORD_CASE upper
\set HISTFILE ~/.psql_history-:DBNAME
\set HISTCONTROL ignoredups
\set HISTSIZE 10000
\set ON_ERROR_ROLLBACK interactive
\set VERBOSITY verbose
\set PROMPT1 '%[%033[1;36m%]%n@%/%[%033[0m%]%R%x%# '
\set PROMPT2 '%R%x%# '
\setenv PAGER 'less -S -X'
\pset pager_min_lines 30
-- handy saved queries (run with :name)
\set activity 'SELECT pid, usename, state, now()-query_start AS runtime, left(query,80) FROM pg_stat_activity WHERE state <> ''idle'' ORDER BY runtime DESC;'
\set locks    'SELECT pid, pg_blocking_pids(pid) AS blocked_by, left(query,80) FROM pg_stat_activity WHERE cardinality(pg_blocking_pids(pid))>0;'
\set sizes    'SELECT relname, pg_size_pretty(pg_total_relation_size(oid)) FROM pg_class WHERE relkind=''r'' AND relnamespace=''public''::regnamespace ORDER BY pg_total_relation_size(oid) DESC LIMIT 20;'
\set dbsize   'SELECT datname, pg_size_pretty(pg_database_size(datname)) FROM pg_database ORDER BY pg_database_size(datname) DESC;'
\set uptime   'SELECT now()-pg_postmaster_start_time() AS uptime;'
\set version  'SELECT version();'
\unset QUIET
```
Use: type `:activity` and press Enter; it expands to the saved query text and runs it.

## History
```
\s                      show history          \s file    save history to file
HISTFILE  HISTSIZE  HISTCONTROL  (see 05)    PSQL_HISTORY=path
~/.psql_history         default file (Windows: %APPDATA%\postgresql\psql_history)
```
Disable history for a session: `\set HISTFILE /dev/null`

## Line editing (Readline / libedit)
```
↑ / Ctrl-P  previous          ↓ / Ctrl-N  next          Ctrl-R  reverse search
Ctrl-A / Ctrl-E  line start/end         Ctrl-W  delete word        Ctrl-U  delete to start        Ctrl-K  delete to end
Ctrl-L  clear screen          Ctrl-C  cancel input/query          Ctrl-D  quit (empty line)
Tab  complete keywords, tables, columns, schemas, functions, \commands, variables
```
- Tab completion queries the server (can interfere with e.g. `SET TRANSACTION` right after `BEGIN`).
- Disable completion permanently in `~/.inputrc`:
```
$if psql
set disable-completion on
$endif
```
- `psql -n` disables readline for one run (use when pasting text containing tabs).
- Multi-line: statements run at `;` (or `\g`); `-S` makes newline end a statement.

## Editor integration
```
\e                      edit current buffer (or last query)
\e file.sql [line]      edit file
\ef func [line]         edit function         \ev view [line]
PSQL_EDITOR / EDITOR / VISUAL         (default vi; Windows notepad.exe)
PSQL_EDITOR_LINENUMBER_ARG='--line '  (how to pass line number; default '+')
```
After saving & quitting the editor, the text is executed (end with `;`).

## Handy aliases (shell)
```bash
alias pgp='psql -X -h prod -U ro appdb'
alias pgq='psql -XAtq'
alias pgcsv='psql -X --csv'
export PSQL_PAGER='pspg -X -b'      # nicer pager (install pspg)
export PG_COLOR=auto
```

## Colour / readability
```
\pset linestyle unicode      \pset border 2      \pset null '∅'      \x auto
PG_COLOR=always (for pg_* tools)   · pspg pager adds colour themes & column freeze
```
