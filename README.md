# PostgreSQL 18 — Command Reference (psql first)

Source: https://www.postgresql.org/docs/current/ (PostgreSQL 18.6)
Format: syntax + one-line comment. Optional parts in `[ ]`, choices with `|`.

Verification status:
- `01-psql/` options, meta-commands, `\pset`, special variables, prompt escapes, env vars, files, exit codes: checked against https://www.postgresql.org/docs/18/app-psql.html
- Everything else (SQL, client/server apps, advanced): written from knowledge, not checked line by line. Version tags like `[17+]` / `[18+]` are from memory; verify on the command's doc page if it matters.

```
pg-commands/
├── 01-psql/                       MAIN — everything about psql
│   ├── 00-quick-card.md           one-page cheat card
│   ├── 01-start-connect.md        invocation, conninfo, .pgpass, service file, env vars, run modes, exit codes
│   ├── 02-cli-options-complete.md every command-line flag
│   ├── 03-meta-commands-complete.md every backslash command
│   ├── 04-describe-commands.md    the \d family, patterns, modifiers
│   ├── 05-variables-and-scripting.md  \set, interpolation, all special variables, \if, \gset, \gexec, \watch
│   ├── 06-output-formatting.md    every \pset option, formats, export, pager, \crosstabview
│   ├── 07-copy-import-export.md   \copy, COPY, large objects, fast load, db-to-db copy
│   ├── 08-prompt-psqlrc-productivity.md  prompt escapes, ~/.psqlrc, history, editing
│   ├── 09-pipeline-extended-protocol.md  \bind, \parse, pipeline mode, \restrict
│   └── 10-psql-recipes.md         monitoring, bulk DDL with \gexec, migrations, error table
├── 02-sql/                        SQL statements (CREATE/ALTER/DROP, queries, DML, roles, VACUUM, ...)
├── 03-client-apps/                createdb, pg_dump, pg_restore, pg_basebackup, vacuumdb, pgbench ...
├── 04-server-apps/                initdb, pg_ctl, pg_upgrade, pg_rewind, pg_waldump ...
├── 05-cheatsheets/                catalog queries, monitoring, common workflows
└── 06-advanced/                   windows, CTEs, JSON, FTS, partitioning + rollover, indexes, MVCC,
                                   PL/pgSQL, tuning, HA/PITR, security, extensions, patterns
```

Suggested order: 01-psql/00 → 01 → 03 → 04 → 05 → 06 → 07, then 02-sql, then 06-advanced.
