# 06 — Roles, Privileges, Security

## Roles
```sql
CREATE ROLE app LOGIN PASSWORD 'secret';
CREATE ROLE admin SUPERUSER LOGIN PASSWORD 'secret';
CREATE ROLE readonly NOLOGIN;
ALTER ROLE app WITH PASSWORD 'new' VALID UNTIL '2027-01-01';
ALTER ROLE app CONNECTION LIMIT 20;
ALTER ROLE app SET search_path = app, public;
DROP ROLE app;
GRANT readonly TO app;                          -- membership
GRANT readonly TO app WITH ADMIN OPTION;
GRANT readonly TO app WITH INHERIT FALSE, SET TRUE;
REVOKE readonly FROM app;
SET ROLE app;            RESET ROLE;
SET SESSION AUTHORIZATION app;
```

## GRANT
```sql
GRANT CONNECT, TEMPORARY, CREATE ON DATABASE db TO r;
GRANT USAGE, CREATE ON SCHEMA s TO r;
GRANT SELECT, INSERT, UPDATE, DELETE, TRUNCATE, REFERENCES, TRIGGER, MAINTAIN ON t TO r;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA s TO r;
GRANT SELECT (col1, col2) ON t TO r;            -- column-level
GRANT UPDATE (col1) ON t TO r;
GRANT USAGE, SELECT, UPDATE ON SEQUENCE seq TO r;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA s TO r;
GRANT EXECUTE ON FUNCTION f(int) TO r;
GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA s TO r;
GRANT EXECUTE ON PROCEDURE p(int) TO r;
GRANT USAGE ON TYPE t TO r;       GRANT USAGE ON DOMAIN d TO r;
GRANT USAGE ON LANGUAGE l TO r;
GRANT USAGE ON FOREIGN DATA WRAPPER w TO r;    GRANT USAGE ON FOREIGN SERVER srv TO r;
GRANT CREATE ON TABLESPACE ts TO r;
GRANT SELECT, UPDATE ON LARGE OBJECT 123 TO r;
GRANT SET, ALTER SYSTEM ON PARAMETER log_min_messages TO r;
GRANT pg_read_all_data TO r;   GRANT pg_write_all_data TO r;   GRANT pg_monitor TO r;
GRANT SELECT ON t TO r WITH GRANT OPTION;
GRANT SELECT ON t TO PUBLIC;
```

## REVOKE
```sql
REVOKE ALL ON t FROM r;
REVOKE INSERT, UPDATE ON t FROM r CASCADE;
REVOKE GRANT OPTION FOR SELECT ON t FROM r;
REVOKE CONNECT ON DATABASE db FROM PUBLIC;
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
```

## Default privileges
```sql
ALTER DEFAULT PRIVILEGES FOR ROLE owner IN SCHEMA s GRANT SELECT ON TABLES TO r;
ALTER DEFAULT PRIVILEGES GRANT USAGE ON SEQUENCES TO r;
ALTER DEFAULT PRIVILEGES REVOKE EXECUTE ON FUNCTIONS FROM PUBLIC;
```

## Ownership clean-up
```sql
ALTER TABLE t OWNER TO r;
REASSIGN OWNED BY old TO new;
DROP OWNED BY old;
```

## Row-level security
```sql
ALTER TABLE t ENABLE ROW LEVEL SECURITY;
ALTER TABLE t FORCE ROW LEVEL SECURITY;
CREATE POLICY own_rows ON t FOR ALL TO app USING (owner = current_user) WITH CHECK (owner = current_user);
CREATE POLICY p ON t AS RESTRICTIVE FOR SELECT USING (tenant_id = current_setting('app.tenant')::int);
ALTER POLICY p ON t USING (...);
DROP POLICY p ON t;
```

## Security labels
```sql
SECURITY LABEL [FOR provider] ON TABLE t IS 'label';
SECURITY LABEL ON ROLE r IS NULL;
```

## Inspect privileges
```sql
\du            \dp            \ddp            \l+
SELECT * FROM information_schema.role_table_grants WHERE grantee = 'r';
SELECT has_table_privilege('r', 't', 'SELECT');
SELECT * FROM pg_roles;     SELECT * FROM pg_auth_members;
SELECT current_user, session_user;
```

## pg_hba.conf line format
```
# TYPE   DATABASE  USER   ADDRESS         METHOD
local    all       all                    peer
host     all       all    127.0.0.1/32    scram-sha-256
hostssl  db        app    10.0.0.0/8      scram-sha-256
host     replication repl 10.0.0.0/8      scram-sha-256
```
Reload: `SELECT pg_reload_conf();` or `pg_ctl reload -D $PGDATA`
