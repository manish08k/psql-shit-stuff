# 11 — High Availability, Replication, Backup Strategy, PITR

## Streaming (physical) replication
Primary `postgresql.conf`:
```
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
wal_keep_size = 1GB
hot_standby = on
synchronous_commit = on                      # on | remote_apply | remote_write | local | off
synchronous_standby_names = 'ANY 1 (s1, s2)' # or 'FIRST 1 (s1, s2)'
```
`pg_hba.conf`: `host replication repl 10.0.0.0/24 scram-sha-256`
```sql
CREATE ROLE repl REPLICATION LOGIN PASSWORD 'pw';
SELECT pg_create_physical_replication_slot('s1');
```
Standby:
```bash
pg_basebackup -h primary -U repl -D $PGDATA -R -Xs -P -S s1
# creates standby.signal + primary_conninfo / primary_slot_name
pg_ctl start -D $PGDATA
```
```
primary_conninfo = 'host=primary user=repl password=pw application_name=s1'
primary_slot_name = 's1'
hot_standby_feedback = on
max_standby_streaming_delay = 30s
recovery_min_apply_delay = 0
```
Monitor:
```sql
-- primary
SELECT application_name, state, sync_state, sent_lsn, replay_lsn, write_lag, flush_lag, replay_lag FROM pg_stat_replication;
-- standby
SELECT pg_is_in_recovery(), pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn(), now() - pg_last_xact_replay_timestamp() AS lag;
SELECT * FROM pg_stat_wal_receiver;
```
Failover / switchover:
```bash
pg_ctl promote -D $PGDATA                       # or: SELECT pg_promote();
pg_rewind --target-pgdata=$OLD --source-server='host=newprimary user=postgres'   # re-attach old primary
```
Cascading: point a standby's `primary_conninfo` at another standby.

## Logical replication
Publisher: `wal_level = logical`
```sql
CREATE PUBLICATION p FOR TABLE a, b;
CREATE PUBLICATION p_all FOR ALL TABLES;
CREATE PUBLICATION p WITH (publish = 'insert,update,delete,truncate', publish_via_partition_root = true);
ALTER PUBLICATION p ADD TABLE c;
ALTER TABLE a REPLICA IDENTITY FULL;            -- if no PK
```
Subscriber (schema must exist: `pg_dump -s | psql`):
```sql
CREATE SUBSCRIPTION s CONNECTION 'host=pub dbname=db user=repl password=pw' PUBLICATION p;
CREATE SUBSCRIPTION s ... WITH (copy_data = false, create_slot = false, slot_name = 'x', streaming = parallel, two_phase = true, failover = true);
ALTER SUBSCRIPTION s REFRESH PUBLICATION;
ALTER SUBSCRIPTION s DISABLE;  ALTER SUBSCRIPTION s ENABLE;
ALTER SUBSCRIPTION s SKIP (lsn = '0/12345');
DROP SUBSCRIPTION s;
SELECT * FROM pg_stat_subscription;   SELECT * FROM pg_subscription_rel;
```
Limits: DDL not replicated; sequences not replicated (sync manually); large objects no.

## Replication slots housekeeping
```sql
SELECT slot_name, slot_type, active, wal_status, restart_lsn, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained FROM pg_replication_slots;
SELECT pg_drop_replication_slot('s');
-- cap: max_slot_wal_keep_size = '20GB'
```

## Backup strategy
| Method | Tool | Restores to | Notes |
|---|---|---|---|
| Logical | `pg_dump` / `pg_dumpall` | any version, selective | slow on huge DB |
| Physical base | `pg_basebackup` | same major version | + WAL archive → PITR |
| Incremental (17+) | `pg_basebackup --incremental` + `pg_combinebackup` | same major | needs `summarize_wal=on` |
| Filesystem snapshot | LVM/ZFS/EBS | same major | pair with `pg_backup_start/stop` or snapshot whole DATA+WAL atomically |
| Managed tools | pgBackRest, Barman, WAL-G | | retention, parallel, encryption, S3 |

### pgBackRest quick reference
```ini
# /etc/pgbackrest/pgbackrest.conf
[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2
[main]
pg1-path=/var/lib/postgresql/18/main
```
```
archive_mode = on
archive_command = 'pgbackrest --stanza=main archive-push %p'
restore_command = 'pgbackrest --stanza=main archive-get %f "%p"'
```
```bash
pgbackrest --stanza=main stanza-create
pgbackrest --stanza=main --type=full backup
pgbackrest --stanza=main --type=diff backup
pgbackrest --stanza=main --type=incr backup
pgbackrest --stanza=main info
pgbackrest --stanza=main check
pgbackrest --stanza=main restore --delta
pgbackrest --stanza=main --type=time --target="2026-11-11 10:00:00+05:30" --target-action=promote restore
```

## PITR step-by-step
```bash
pg_ctl stop -D $PGDATA
mv $PGDATA ${PGDATA}.broken
mkdir $PGDATA && tar -xf base.tar -C $PGDATA          # restore base backup (+ pg_wal if needed)
cat >> $PGDATA/postgresql.conf <<EOF
restore_command = 'cp /archive/%f %p'
recovery_target_time = '2026-11-11 10:00:00+05:30'     # or recovery_target_lsn / recovery_target_xid / recovery_target_name = 'label' / recovery_target = 'immediate'
recovery_target_action = 'promote'                     # pause | promote | shutdown
recovery_target_inclusive = on
EOF
touch $PGDATA/recovery.signal
pg_ctl start -D $PGDATA
```
Named restore point: `SELECT pg_create_restore_point('before_migration');`
After promote, timeline increases (`recovery_target_timeline = 'latest'`).

## HA tooling
| Tool | Role |
|---|---|
| **Patroni** (+ etcd/Consul/ZooKeeper) | automatic failover, leader election |
| **repmgr** | replication manager, switchover |
| **pg_auto_failover** | monitor-based failover |
| **HAProxy / pgbouncer / pgpool-II** | routing, pooling, load balancing |
| **keepalived / VIP** | floating IP |
```bash
patronictl -c /etc/patroni.yml list
patronictl -c /etc/patroni.yml switchover
patronictl -c /etc/patroni.yml failover
patronictl -c /etc/patroni.yml reinit <member>
patronictl -c /etc/patroni.yml edit-config
repmgr -f /etc/repmgr.conf standby clone -h primary
repmgr -f /etc/repmgr.conf standby register
repmgr -f /etc/repmgr.conf standby switchover
repmgr -f /etc/repmgr.conf cluster show
```

## Read scaling
Hot standbys for read queries; route via HAProxy/pgpool; `target_session_attrs=read-write|read-only|primary|standby|prefer-standby` in libpq URL:
`postgresql://h1,h2/db?target_session_attrs=read-write`

## Declarative DR checklist
1. Test restores monthly · 2. Monitor `pg_stat_archiver` failures · 3. Alert on replication lag & slot retention · 4. Store backups off-host · 5. Keep `pg_dumpall -g` for roles · 6. Document RPO/RTO.
