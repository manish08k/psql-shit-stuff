# Cheatsheet — Common Workflows

## 1. New app database + user
```sql
CREATE ROLE app LOGIN PASSWORD 'strong_pw';
CREATE DATABASE appdb OWNER app;
\c appdb
REVOKE ALL ON SCHEMA public FROM PUBLIC;
GRANT USAGE, CREATE ON SCHEMA public TO app;
```

## 2. Read-only user
```sql
CREATE ROLE ro LOGIN PASSWORD 'pw';
GRANT CONNECT ON DATABASE appdb TO ro;
GRANT USAGE ON SCHEMA public TO ro;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO ro;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO ro;
-- or: GRANT pg_read_all_data TO ro;
```

## 3. Reset postgres password
```bash
sudo -u postgres psql -c "ALTER USER postgres PASSWORD 'new';"
```

## 4. Enable remote access
```
# postgresql.conf
listen_addresses = '*'
# pg_hba.conf
host  all  all  0.0.0.0/0  scram-sha-256
```
```bash
sudo systemctl restart postgresql
```

## 5. Logical backup + restore
```bash
pg_dump -Fc -f appdb.dump appdb
createdb appdb_new
pg_restore -j 4 -d appdb_new appdb.dump
```

## 6. Clone a database (same server)
```sql
CREATE DATABASE copy TEMPLATE appdb;              -- no active connections on appdb
```

## 7. Rename DB / move schema
```sql
ALTER DATABASE old RENAME TO new;                 -- must not be connected to it
ALTER TABLE s1.t SET SCHEMA s2;
```

## 8. Import / export CSV
```bash
psql -d db -c "\copy t FROM 'data.csv' CSV HEADER"
psql -d db -c "\copy (SELECT * FROM t) TO 'out.csv' CSV HEADER"
```

## 9. Zero-downtime-ish index / column changes
```sql
CREATE INDEX CONCURRENTLY idx ON t (col);
ALTER TABLE t ADD COLUMN c int;                                   -- fast (no rewrite) with constant default
ALTER TABLE t ADD CONSTRAINT ck CHECK (c > 0) NOT VALID;
ALTER TABLE t VALIDATE CONSTRAINT ck;
ALTER TABLE t ADD CONSTRAINT fk FOREIGN KEY (x) REFERENCES o(id) NOT VALID;
ALTER TABLE t VALIDATE CONSTRAINT fk;
SET lock_timeout = '3s';                                          -- before DDL to avoid lock pile-ups
```

## 10. Streaming replica
```bash
# primary: wal_level=replica, max_wal_senders>=5, pg_hba 'host replication repl <ip>/32 scram-sha-256'
psql -c "CREATE ROLE repl REPLICATION LOGIN PASSWORD 'pw';"
# standby:
pg_basebackup -h primary -U repl -D $PGDATA -R -Xs -P
pg_ctl start -D $PGDATA
# failover:
pg_ctl promote -D $PGDATA
```

## 11. Point-in-time recovery
```bash
# 1. stop server, move old data aside, restore base backup into $PGDATA
# 2. postgresql.conf:
#    restore_command = 'cp /archive/%f %p'
#    recovery_target_time = '2026-11-11 10:00:00+05:30'
#    recovery_target_action = 'promote'
touch $PGDATA/recovery.signal
pg_ctl start -D $PGDATA
```

## 12. Major-version upgrade
```bash
pg_dumpall -g > globals.sql                                     # safety backup
pg_upgrade -b OLD/bin -B NEW/bin -d OLD/data -D NEW/data --check
pg_upgrade -b OLD/bin -B NEW/bin -d OLD/data -D NEW/data --link -j 4
vacuumdb --all --analyze-in-stages
```

## 13. Routine maintenance
```bash
vacuumdb -a -z -j 4
reindexdb -a --concurrently
pg_amcheck -a -j 4
```

## 14. Find & kill long queries
```sql
SELECT pid, now()-query_start AS d, query FROM pg_stat_activity WHERE state='active' AND now()-query_start > interval '5 min';
SELECT pg_cancel_backend(<pid>);
```

## 15. Auto-update timestamp trigger
```sql
CREATE FUNCTION set_updated() RETURNS trigger LANGUAGE plpgsql AS $$ BEGIN NEW.updated_at = now(); RETURN NEW; END $$;
CREATE TRIGGER t_upd BEFORE UPDATE ON t FOR EACH ROW EXECUTE FUNCTION set_updated();
```

## 16. Pagination / upsert / soft-delete
```sql
SELECT * FROM t ORDER BY id LIMIT 20 OFFSET 40;
SELECT * FROM t WHERE id > :last_id ORDER BY id LIMIT 20;         -- keyset
INSERT INTO t (k, v) VALUES ('a', 1) ON CONFLICT (k) DO UPDATE SET v = EXCLUDED.v;
UPDATE t SET deleted_at = now() WHERE id = 1;
```
