# Client Apps — Backup & Restore

## pg_dump (one database)
```bash
pg_dump db > db.sql                               # plain SQL
pg_dump -Fc db > db.dump                          # custom format (compressed, pg_restore)
pg_dump -Fd -j 4 -f dumpdir db                    # directory format, parallel
pg_dump -Ft db > db.tar                           # tar format
```
| Flag | Meaning |
|---|---|
| `-F p\|c\|d\|t` | format: plain, custom, directory, tar |
| `-f file` | output file/dir |
| `-j N` | parallel jobs (directory format only) |
| `-Z 0-9` / `-Z zstd:5` / `-Z lz4` | compression |
| `-s, --schema-only` | schema only |
| `-a, --data-only` | data only |
| `-t table` / `-T table` | include / exclude table (patterns) |
| `-n schema` / `-N schema` | include / exclude schema |
| `--exclude-table-data=t` | skip data of table |
| `-C, --create` | include CREATE DATABASE |
| `-c, --clean` / `--if-exists` | emit DROP before CREATE |
| `-O, --no-owner` / `-x, --no-privileges` | skip ownership / ACL |
| `--inserts` / `--column-inserts` | INSERT instead of COPY |
| `--on-conflict-do-nothing` | with --inserts |
| `-b, --large-objects` / `--no-large-objects` | blobs |
| `--no-comments` `--no-publications` `--no-subscriptions` `--no-tablespaces` | skip items |
| `--section=pre-data\|data\|post-data` | partial |
| `--serializable-deferrable` | consistent snapshot, no anomalies |
| `--snapshot=id` | use exported snapshot |
| `--role=r` | SET ROLE before dump |
| `-v` | verbose |
| `--statistics-only` / `--no-statistics` | planner stats (18) |

```bash
pg_dump -U postgres -h host -p 5432 -Fc -f db.dump db
pg_dump -s db > schema.sql
pg_dump -t 'public.users' -t 'public.orders' db > two_tables.sql
pg_dump -n app --no-owner --no-privileges db > app.sql
pg_dump -Fc db | gzip > db.dump.gz
```

## pg_dumpall (whole cluster)
```bash
pg_dumpall > cluster.sql
pg_dumpall -g > globals.sql                       # roles + tablespaces only
pg_dumpall -r > roles.sql
pg_dumpall -t > tablespaces.sql
pg_dumpall --clean --if-exists > cluster.sql
pg_dumpall --no-role-passwords > cluster.sql
pg_dumpall -U postgres -h host -f cluster.sql
```

## pg_restore (custom/directory/tar archives)
```bash
pg_restore -d db db.dump
pg_restore -C -d postgres db.dump                 # create db from archive, then connect
pg_restore -j 4 -d db db.dump                     # parallel
pg_restore -c --if-exists -d db db.dump           # clean first
pg_restore -s -d db db.dump                       # schema only
pg_restore -a -d db db.dump                       # data only
pg_restore -t users -d db db.dump                 # one table
pg_restore -n app -d db db.dump                   # one schema
pg_restore -l db.dump > toc.list                  # list contents
pg_restore -L toc.list -d db db.dump              # restore selected TOC
pg_restore -O -x -d db db.dump                    # no owner / no privileges
pg_restore --single-transaction -e -d db db.dump  # all-or-nothing, exit on error
pg_restore -f out.sql db.dump                     # convert to plain SQL
pg_restore --no-data-for-failed-tables -d db db.dump
pg_restore --role=r -d db db.dump
```
Plain-SQL dumps restore with psql: `psql -d db -f db.sql` / `psql -v ON_ERROR_STOP=1 -1 -f db.sql db`

## pg_basebackup (physical base backup)
```bash
pg_basebackup -D /backup/base -Fp -Xs -P -v -h primary -U repl
pg_basebackup -D /backup -Ft -z -Xs -P                    # tar + gzip
pg_basebackup -D /backup -Ft --compress=server-zstd:5
pg_basebackup -D /replica -R -Xs -P -h primary -U repl    # -R writes standby.signal + primary_conninfo
pg_basebackup -D /backup -C -S slot1 -Xs                  # create replication slot
pg_basebackup -D /backup --incremental=/prev/backup_manifest   # incremental (17+)
pg_basebackup -D /backup -c fast -l "label" --checkpoint=fast
pg_basebackup -D /backup -T /old/ts=/new/ts               # tablespace remap
pg_basebackup -D /backup --max-rate=50M --no-manifest --manifest-checksums=sha256
```
| Flag | Meaning |
|---|---|
| `-D dir` | target dir |
| `-F p\|t` | plain / tar |
| `-X none\|fetch\|stream` (`-Xs`) | include WAL |
| `-R` | write recovery config |
| `-P` | progress |
| `-z` `-Z` | compression |
| `-S slot` `-C` | slot name / create |
| `-c fast\|spread` | checkpoint mode |
| `-r rate` | throttle |
| `-i manifest` | incremental |

## pg_combinebackup (incremental → full)
```bash
pg_combinebackup /backup/full /backup/incr1 /backup/incr2 -o /restore/dir
pg_combinebackup --copy-file-range -o /out full incr1
pg_combinebackup -T /old/ts=/new/ts -o /out full incr1
```

## pg_verifybackup
```bash
pg_verifybackup /backup/base
pg_verifybackup -n /backup/base                  # skip WAL parsing
pg_verifybackup -m /path/backup_manifest -w /wal /backup/base
pg_verifybackup -s /backup/base                  # skip checksums
```

## Quick backup scripts
```bash
# daily compressed logical backup
pg_dump -Fc -Z 6 -f /backups/db_$(date +%F).dump db
# restore into a fresh DB
createdb newdb && pg_restore -j 4 -d newdb /backups/db_2026-11-11.dump
# copy database over network
pg_dump -Fc -h src db | pg_restore -h dst -d db --clean --if-exists
```
