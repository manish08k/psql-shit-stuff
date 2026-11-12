# Client Apps — Other Utilities

## pg_isready
```bash
pg_isready
pg_isready -h host -p 5432 -d db -U user -t 5 -q
```
Exit codes: `0` accepting · `1` rejecting · `2` no response · `3` no attempt

## pg_config
```bash
pg_config --bindir --libdir --includedir --sharedir --sysconfdir --pkglibdir --version
pg_config --configure --cc --cflags --ldflags
```

## pgbench
```bash
pgbench -i -s 50 mydb                         # init, scale 50 (~5M rows)
pgbench -i --foreign-keys --partitions=4 mydb
pgbench -c 16 -j 4 -T 60 -P 5 mydb            # 16 clients, 4 threads, 60s, report each 5s
pgbench -c 10 -t 1000 mydb                    # 1000 tx per client
pgbench -S -c 16 -T 30 mydb                   # select-only
pgbench -N -c 16 -T 30 mydb                   # skip updates on branches/tellers
pgbench -f custom.sql@1 -c 8 -T 30 mydb       # custom script, weight 1
pgbench -M prepared|simple|extended mydb
pgbench -r mydb                               # per-statement latencies
pgbench -R 500 -L 100 mydb                    # rate limit tps, latency limit ms
pgbench --progress-timestamp -l mydb          # write log
```
Script variables: `\set aid random(1, 100000 * :scale)`

## pg_receivewal (stream WAL)
```bash
pg_receivewal -D /walarchive -h primary -U repl
pg_receivewal -D /walarchive -S slot1 --create-slot
pg_receivewal -D /walarchive -S slot1 --drop-slot
pg_receivewal -D /walarchive --synchronous -Z 5 --compress=gzip
pg_receivewal -D /walarchive --no-loop --if-not-exists -v
```

## pg_recvlogical (logical decoding)
```bash
pg_recvlogical -d db --slot=s1 --create-slot -P test_decoding
pg_recvlogical -d db --slot=s1 --start -f -
pg_recvlogical -d db --slot=s1 --start -o include-xids=1 -f changes.out
pg_recvlogical -d db --slot=s1 --drop-slot
pg_recvlogical -d db --slot=s1 --create-slot --two-phase --enable-failover
```

## ecpg (embedded SQL in C)
```bash
ecpg prog.pgc                    # produces prog.c
ecpg -o out.c prog.pgc
ecpg -I dir -D SYMBOL -C ORACLE -c -t -r no_indicator prog.pgc
cc -I$(pg_config --includedir) prog.c -L$(pg_config --libdir) -lecpg
```
