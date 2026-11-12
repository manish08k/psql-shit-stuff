# Server Apps — WAL, Control Data, Checksums, Diagnostics

## pg_controldata
```bash
pg_controldata -D $PGDATA
pg_controldata $PGDATA | grep -E 'cluster state|checkpoint location|REDO|TimeLineID|checksum'
```

## pg_resetwal (last resort — can corrupt!)
```bash
pg_resetwal -n -D $PGDATA                      # dry run (show values)
pg_resetwal -f -D $PGDATA                      # force reset
pg_resetwal -x XID -D $PGDATA                  # next xid
pg_resetwal -o OID / -m MXID,MXID / -O MXOFF / -l walfile / -c XID,XID / -e epoch
pg_resetwal --wal-segsize=SIZE -D $PGDATA
```

## pg_waldump
```bash
pg_waldump /wal/dir/000000010000000000000001
pg_waldump -p /wal/dir -s 0/1000000 -e 0/2000000
pg_waldump -r Heap|Btree|Transaction|XLOG ... -p /wal/dir 000000010000000000000001
pg_waldump -b / --bkp-details / -f / --follow / -n 20 / -x XID / -R rel / -F fork / -t timeline / --stats[=record] / -z
```

## pg_walsummary
```bash
pg_walsummary /pgdata/pg_wal/summaries/*.summary
pg_walsummary -i / --individual file
```
(used with `summarize_wal = on` for incremental backups)

## pg_checksums
```bash
pg_checksums --check  -D $PGDATA               # cluster must be stopped cleanly
pg_checksums --enable -D $PGDATA [-P] [-v]
pg_checksums --disable -D $PGDATA
pg_checksums --check -D $PGDATA -f filenode -r relfilenode
```

## pg_archivecleanup
```bash
pg_archivecleanup /archive 000000010000000000000010
pg_archivecleanup -n -d /archive 000000010000000000000010       # dry run, debug
pg_archivecleanup -x .gz /archive 000000010000000000000010
# in postgresql.conf standby: archive_cleanup_command = 'pg_archivecleanup /archive %r'
```

## WAL archiving settings
```
archive_mode = on
archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'
restore_command = 'cp /archive/%f %p'
archive_cleanup_command = 'pg_archivecleanup /archive %r'
recovery_target_time = '2026-11-11 10:00:00+05:30'     # PITR
recovery_target_action = 'promote'
```
PITR: restore base backup → create `recovery.signal` → set restore_command + target → start server.

## Benchmark helpers
```bash
pg_test_fsync -f /pgdata/testfile -s 5            # compare wal_sync_method
pg_test_timing -d 3                               # timing overhead (seconds)
```
