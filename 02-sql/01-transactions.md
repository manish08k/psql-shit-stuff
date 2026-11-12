# 01 — Transactions, Savepoints, Locks, Two-Phase Commit

## Transaction control
```sql
BEGIN;                                   -- start transaction block
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;
START TRANSACTION;                       -- same as BEGIN
COMMIT;                                  -- commit
COMMIT AND CHAIN;                        -- commit + start new tx with same characteristics
END;                                     -- same as COMMIT
ROLLBACK;                                -- abort
ROLLBACK AND CHAIN;
ABORT;                                   -- same as ROLLBACK
```

## SET TRANSACTION
```sql
SET TRANSACTION ISOLATION LEVEL { SERIALIZABLE | REPEATABLE READ | READ COMMITTED | READ UNCOMMITTED };
SET TRANSACTION READ WRITE | READ ONLY;
SET TRANSACTION [NOT] DEFERRABLE;
SET TRANSACTION SNAPSHOT 'snapshot_id';  -- import exported snapshot
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL REPEATABLE READ;  -- session default
```

## SET CONSTRAINTS
```sql
SET CONSTRAINTS ALL DEFERRED;
SET CONSTRAINTS ALL IMMEDIATE;
SET CONSTRAINTS fk_name1, fk_name2 DEFERRED;
```

## Savepoints
```sql
SAVEPOINT sp1;
ROLLBACK TO SAVEPOINT sp1;
ROLLBACK TO sp1;
RELEASE SAVEPOINT sp1;
```

## LOCK
```sql
LOCK TABLE t IN ACCESS SHARE MODE;
LOCK TABLE t IN ROW SHARE MODE;
LOCK TABLE t IN ROW EXCLUSIVE MODE;
LOCK TABLE t IN SHARE UPDATE EXCLUSIVE MODE;
LOCK TABLE t IN SHARE MODE;
LOCK TABLE t IN SHARE ROW EXCLUSIVE MODE;
LOCK TABLE t IN EXCLUSIVE MODE;
LOCK TABLE t IN ACCESS EXCLUSIVE MODE;   -- default
LOCK TABLE t1, t2 IN SHARE MODE NOWAIT;
LOCK TABLE ONLY t IN SHARE MODE;
```

## Row-level locks (inside SELECT)
```sql
SELECT ... FOR UPDATE;
SELECT ... FOR NO KEY UPDATE;
SELECT ... FOR SHARE;
SELECT ... FOR KEY SHARE;
SELECT ... FOR UPDATE OF t NOWAIT;
SELECT ... FOR UPDATE SKIP LOCKED;
```

## Advisory locks (functions)
```sql
SELECT pg_advisory_lock(key);            SELECT pg_advisory_unlock(key);
SELECT pg_try_advisory_lock(key);
SELECT pg_advisory_xact_lock(key);       -- released at tx end
```

## Two-phase commit
```sql
PREPARE TRANSACTION 'gid';
COMMIT PREPARED 'gid';
ROLLBACK PREPARED 'gid';
SELECT * FROM pg_prepared_xacts;
```
