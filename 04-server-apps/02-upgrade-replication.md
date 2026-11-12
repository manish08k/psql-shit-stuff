# Server Apps — Upgrade & Replication

## pg_upgrade
```bash
pg_upgrade \
  -b /usr/lib/postgresql/17/bin -B /usr/lib/postgresql/18/bin \
  -d /var/lib/postgresql/17/main -D /var/lib/postgresql/18/main \
  [-U postgres] [-p 50432 -P 50433] [-j 4] [--check] [--link|--clone|--copy|--swap] [-v]
```
| Flag | Meaning |
|---|---|
| `-b/-B` | old / new bin dir |
| `-d/-D` | old / new data dir |
| `-c, --check` | dry-run compatibility check |
| `-k, --link` | hard links (fast, old cluster unusable after start of new) |
| `--clone` | reflink copy (fast, safe on supporting FS) |
| `--copy` | full copy (default) |
| `--swap` | swap dirs (18) |
| `-j N` | parallel jobs |
| `-r` | retain logs |
| `--no-statistics` | skip carrying planner stats |
| `--set-char-signedness=signed\|unsigned` | char signedness (18) |

Steps:
```bash
initdb -D new_data
pg_upgrade --check ...
pg_ctl stop -D old_data
pg_upgrade ...
pg_ctl start -D new_data
vacuumdb --all --analyze-in-stages --missing-stats-only
./delete_old_cluster.sh
```

## pg_rewind (resync a diverged old primary)
```bash
pg_rewind --target-pgdata=/old/primary/data --source-server='host=newprimary user=postgres dbname=postgres'
pg_rewind -D $PGDATA --source-pgdata=/other/data
pg_rewind -D $PGDATA --source-server=... -R -P --dry-run
pg_rewind -D $PGDATA --source-server=... --write-recovery-conf --restore-target-wal --config-file=postgresql.conf
```
Requires `wal_log_hints=on` or data checksums.

## pg_createsubscriber (physical → logical replica)
```bash
pg_createsubscriber -D /standby/data -P 'host=primary port=5432 user=repl' -d db1 -d db2 [-p 5433] [-U user] [--dry-run] [-v]
pg_createsubscriber ... --publication=pub1 --subscription=sub1 --replication-slot=slot1 --enable-two-phase --clean=publications
```

## Standby setup recipe
```bash
pg_basebackup -h primary -U repl -D $PGDATA -R -Xs -P
# -R creates standby.signal + primary_conninfo in postgresql.auto.conf
pg_ctl start -D $PGDATA
pg_ctl promote -D $PGDATA
```
Primary settings: `wal_level=replica  max_wal_senders=10  max_replication_slots=10  hot_standby=on  synchronous_standby_names='...'`
