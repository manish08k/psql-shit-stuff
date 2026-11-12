# Client Apps — Maintenance Tools

## vacuumdb
```bash
vacuumdb mydb
vacuumdb -a                                   # all databases
vacuumdb -z mydb                              # vacuum + analyze
vacuumdb -Z mydb                              # analyze only
vacuumdb --analyze-in-stages mydb             # after pg_upgrade
vacuumdb --analyze-only --missing-stats-only mydb
vacuumdb -f mydb                              # FULL
vacuumdb -F mydb                              # FREEZE
vacuumdb -t tbl -t 'tbl2(col)' mydb
vacuumdb -n schema / -N schema mydb
vacuumdb -j 4 mydb                            # parallel
vacuumdb -v mydb
vacuumdb --min-xid-age=1000000 --min-mxid-age=1000000 mydb
vacuumdb --no-index-cleanup --no-truncate --no-process-toast --skip-locked --parallel=2 mydb
vacuumdb --buffer-usage-limit=256kB mydb
```

## reindexdb
```bash
reindexdb mydb
reindexdb -a                                  # all databases
reindexdb -t tbl mydb
reindexdb -i idx mydb
reindexdb -S schema mydb
reindexdb -s mydb                             # system catalogs
reindexdb --concurrently mydb
reindexdb -j 4 mydb
reindexdb --tablespace=ts mydb
reindexdb --jobs=4 --concurrently -v mydb
```

## clusterdb
```bash
clusterdb mydb
clusterdb -a
clusterdb -t tbl mydb
clusterdb -v mydb
```

## pg_amcheck (corruption check)
```bash
pg_amcheck mydb
pg_amcheck -a                                 # all databases
pg_amcheck -t public.tbl mydb
pg_amcheck -i idx mydb
pg_amcheck -s schema mydb
pg_amcheck -j 4 mydb
pg_amcheck --heapallindexed --rootdescend --parent-check mydb
pg_amcheck --install-missing mydb             # installs amcheck extension
pg_amcheck --startblock=0 --endblock=1000 -t tbl mydb
pg_amcheck --no-dependent-indexes --on-error-stop -v mydb
```
