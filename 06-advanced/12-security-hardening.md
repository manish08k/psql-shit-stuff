# 12 — Security Hardening

## Authentication
```
# postgresql.conf
password_encryption = scram-sha-256
ssl = on
ssl_cert_file = 'server.crt'   ssl_key_file = 'server.key'   ssl_ca_file = 'root.crt'
ssl_min_protocol_version = 'TLSv1.2'
listen_addresses = 'localhost'          # or specific IPs
```
```
# pg_hba.conf  (first match wins)
# TYPE     DB     USER   ADDRESS        METHOD   [options]
local      all    postgres               peer
local      all    all                    scram-sha-256
host       all    all    127.0.0.1/32    scram-sha-256
hostssl    appdb  app    10.0.0.0/24     scram-sha-256
hostssl    appdb  app    10.0.0.0/24     cert clientcert=verify-full
hostnossl  all    all    0.0.0.0/0       reject
host       replication repl 10.0.1.0/24  scram-sha-256
```
Methods: `trust` (avoid) · `reject` · `scram-sha-256` · `md5` (legacy) · `password` (plaintext!) · `peer` · `ident` · `cert` · `gss` · `sspi` · `ldap` · `radius` · `pam` · `oauth` (18)
```sql
SELECT * FROM pg_hba_file_rules;       SELECT pg_reload_conf();
ALTER ROLE r PASSWORD 'x';             -- stored as SCRAM when password_encryption=scram-sha-256
SELECT rolname, left(rolpassword,14) FROM pg_authid;       -- SCRAM-SHA-256$...
```
Client SSL: `psql "host=h dbname=d user=u sslmode=verify-full sslrootcert=root.crt"` · modes: `disable allow prefer require verify-ca verify-full`
Generate self-signed:
```bash
openssl req -new -x509 -days 365 -nodes -text -out server.crt -keyout server.key -subj "/CN=dbhost"
chmod 600 server.key
```

## Least privilege
```sql
REVOKE ALL ON DATABASE db FROM PUBLIC;
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
CREATE ROLE app_owner NOLOGIN;  CREATE ROLE app_rw NOLOGIN;  CREATE ROLE app_ro NOLOGIN;  CREATE ROLE svc LOGIN PASSWORD '...' IN ROLE app_rw;
GRANT CONNECT ON DATABASE db TO app_ro, app_rw;
GRANT USAGE ON SCHEMA app TO app_ro, app_rw;
GRANT SELECT ON ALL TABLES IN SCHEMA app TO app_ro;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app TO app_rw;
ALTER DEFAULT PRIVILEGES FOR ROLE app_owner IN SCHEMA app GRANT SELECT ON TABLES TO app_ro;
ALTER DEFAULT PRIVILEGES FOR ROLE app_owner IN SCHEMA app GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_rw;
ALTER ROLE svc SET statement_timeout = '30s';  ALTER ROLE svc CONNECTION LIMIT 50;
-- built-in roles
pg_read_all_data  pg_write_all_data  pg_monitor  pg_read_all_settings  pg_read_all_stats  pg_stat_scan_tables
pg_signal_backend  pg_read_server_files  pg_write_server_files  pg_execute_server_program  pg_checkpoint  pg_maintain  pg_use_reserved_connections  pg_create_subscription
```

## Row-level security (multi-tenant)
```sql
ALTER TABLE doc ENABLE ROW LEVEL SECURITY;
ALTER TABLE doc FORCE ROW LEVEL SECURITY;                                  -- applies to owner too
CREATE POLICY tenant_iso ON doc USING (tenant_id = current_setting('app.tenant_id')::int)
                                WITH CHECK (tenant_id = current_setting('app.tenant_id')::int);
-- per request:  SET LOCAL app.tenant_id = '42';
CREATE POLICY admin_all ON doc TO admin USING (true);
CREATE POLICY ro ON doc FOR SELECT TO app_ro USING (published);
-- bypass: roles with BYPASSRLS / superusers / (table owner unless FORCE)
SELECT * FROM pg_policies;
```

## SECURITY DEFINER safely
```sql
CREATE FUNCTION f() RETURNS ... SECURITY DEFINER SET search_path = pg_catalog, pg_temp AS $$ ... $$ LANGUAGE sql;
REVOKE EXECUTE ON FUNCTION f() FROM PUBLIC;
GRANT EXECUTE ON FUNCTION f() TO app_rw;
```

## Column-level & view-based
```sql
GRANT SELECT (id, name) ON users TO support;
CREATE VIEW users_safe WITH (security_barrier = true, security_invoker = true) AS SELECT id, name FROM users;
```

## Encryption of data
```sql
CREATE EXTENSION pgcrypto;
INSERT INTO t (secret) VALUES (pgp_sym_encrypt('data', 'key'));
SELECT pgp_sym_decrypt(secret, 'key') FROM t;
SELECT crypt('password', gen_salt('bf', 12));
SELECT digest('x','sha256');     SELECT encode(gen_random_bytes(32),'hex');
```
At rest → disk/volume encryption (LUKS, EBS, etc.); in transit → SSL.

## Auditing
```
shared_preload_libraries = 'pgaudit'
pgaudit.log = 'write, ddl, role'
log_connections = on   log_disconnections = on   log_statement = 'ddl'
log_line_prefix = '%m [%p] %q%u@%d/%a '
```
```sql
CREATE EXTENSION pgaudit;
-- or trigger-based audit table (see 09-plpgsql-advanced.md)
```

## SQL-injection safety
```sql
-- always parameterize in app code; in dynamic SQL use format('%I','%L') / quote_ident / quote_literal / USING
EXECUTE format('SELECT * FROM %I WHERE id = $1', tbl) USING id;
```

## Hardening checklist
- No `trust` in pg_hba; SCRAM + TLS; bind only needed interfaces; firewall 5432
- Revoke PUBLIC CREATE on `public` schema; separate owner / app / readonly roles
- Superuser only for admin, never for the app; `ALTER ROLE ... NOSUPERUSER`
- `statement_timeout`, `idle_in_transaction_session_timeout`, connection limits
- Patch minor versions promptly (`SELECT version();`)
- Backups encrypted & restore-tested; protect `postgresql.auto.conf`, `.pgpass` (600)
- Monitor `pg_stat_activity`, failed logins in log, `pg_hba_file_rules`
