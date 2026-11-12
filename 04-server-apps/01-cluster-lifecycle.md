# Server Apps — Cluster Lifecycle

## initdb
```bash
initdb -D /var/lib/postgresql/18/main
initdb -D $PGDATA -U postgres --auth-local=peer --auth-host=scram-sha-256
initdb -D $PGDATA -W                                  # prompt superuser password
initdb -D $PGDATA --pwfile=pw.txt
initdb -D $PGDATA -E UTF8 --locale=en_US.UTF-8 --locale-provider=icu --icu-locale=en-US
initdb -D $PGDATA -k                                  # --data-checksums
initdb -D $PGDATA --no-data-checksums
initdb -D $PGDATA -X /wal/dir                         # separate WAL dir (--waldir)
initdb -D $PGDATA --wal-segsize=64
initdb -D $PGDATA -g                                  # group access
initdb -D $PGDATA -n / --no-clean / -N (--no-sync) / -S (--sync-only)
```

## pg_ctl
```bash
pg_ctl init  -D $PGDATA
pg_ctl start -D $PGDATA -l logfile -o "-p 5433"
pg_ctl start -D $PGDATA -w -t 60                      # wait
pg_ctl stop  -D $PGDATA -m smart|fast|immediate       # fast = default
pg_ctl restart -D $PGDATA -m fast
pg_ctl reload  -D $PGDATA                             # re-read config
pg_ctl status  -D $PGDATA
pg_ctl promote -D $PGDATA                             # standby → primary
pg_ctl logrotate -D $PGDATA
pg_ctl kill HUP|INT|TERM|QUIT <pid>
pg_ctl register|unregister -N svc -D $PGDATA          # Windows service
```
Debian/Ubuntu wrappers:
```bash
pg_lsclusters
pg_createcluster 18 main
pg_ctlcluster 18 main start|stop|restart|reload|status
pg_dropcluster 18 main
pg_upgradecluster 17 main
```
systemd: `systemctl start|stop|restart|reload|status|enable postgresql[@18-main]`

## postgres (server binary)
```bash
postgres -D $PGDATA
postgres -D $PGDATA -p 5433 -c shared_buffers=1GB -c max_connections=200
postgres -C data_directory -D $PGDATA                 # print a parameter and exit
postgres -C shared_memory_size_in_huge_pages -D $PGDATA
postgres --single -D $PGDATA mydb                     # single-user mode (repair)
postgres --check -D $PGDATA                           # bootstrap check mode
postgres --describe-config
postgres -V
```
Common `-c` / flags: `-B shared_buffers  -N max_connections  -h listen_addresses  -k unix_socket_directories  -i  -F (fsync off, unsafe)  -d debug_level  -e (euro dates)  -s (stats)`

## Key data-dir files
```
postgresql.conf       postgresql.auto.conf     pg_hba.conf     pg_ident.conf
PG_VERSION            postmaster.pid           standby.signal  recovery.signal
base/  global/  pg_wal/  pg_xact/  pg_tblspc/  pg_stat/  log/
```
