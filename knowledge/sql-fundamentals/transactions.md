# Transactions Deep Dive

## ACID Properties Detailed

### Atomicity
All operations in transaction succeed or all fail. No partial updates.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- Debit
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- Credit
-- If either fails, both are rolled back
COMMIT;
```

### Consistency
Database moves from one valid state to another. Constraints enforced.

```sql
-- Balance constraint
ALTER TABLE accounts ADD CONSTRAINT chk_balance CHECK (balance >= 0);

BEGIN;
UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
-- If balance would go negative, constraint violation → rollback
COMMIT;
```

### Isolation
Concurrent transactions don't interfere with each other.

### Durability
Committed changes survive system crashes (WAL, redo logs).

## Isolation Levels Deep Dive

### Read Phenomena

| Phenomenon | Description |
|------------|-------------|
| Dirty Read | Read uncommitted data from another transaction |
| Non-Repeatable Read | Same query returns different results (row changed) |
| Phantom Read | Same query returns different rows (rows added/deleted) |
| Lost Update | Two transactions update same row, one overwrites other |
| Write Skew | Two transactions make decisions based on overlapping reads |

### READ UNCOMMITTED

```sql
-- Transaction A
BEGIN;
UPDATE accounts SET balance = 0 WHERE id = 1;
-- No commit yet

-- Transaction B (READ UNCOMMITTED)
SELECT balance FROM accounts WHERE id = 1;  -- Reads 0 (dirty read)

-- Transaction A
ROLLBACK;  -- Balance was never really 0
```

### READ COMMITTED (PostgreSQL default)

```sql
-- Transaction A
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- 100

-- Transaction B
BEGIN;
UPDATE accounts SET balance = 200 WHERE id = 1;
COMMIT;

-- Transaction A
SELECT balance FROM accounts WHERE id = 1;  -- 200 (non-repeatable read)
COMMIT;
```

### REPEATABLE READ

```sql
-- Transaction A
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- 100

-- Transaction B
UPDATE accounts SET balance = 200 WHERE id = 1;
COMMIT;

-- Transaction A
SELECT balance FROM accounts WHERE id = 1;  -- Still 100 (snapshot)
COMMIT;
```

### SERIALIZABLE

```sql
-- Prevents write skew
-- Example: On-call schedule (need at least one person)

-- Transaction A
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN;
SELECT COUNT(*) FROM oncall WHERE date = '2024-01-15';  -- 2
-- Decides it's safe to remove self
DELETE FROM oncall WHERE user_id = 1 AND date = '2024-01-15';

-- Transaction B (simultaneously)
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN;
SELECT COUNT(*) FROM oncall WHERE date = '2024-01-15';  -- 2
-- Also decides it's safe
DELETE FROM oncall WHERE user_id = 2 AND date = '2024-01-15';

-- One will fail with serialization error
-- Without SERIALIZABLE, both succeed → 0 on-call (write skew)
```

## Locking Mechanisms

### Lock Modes

| Lock | Conflicts With | Use Case |
|------|----------------|----------|
| ACCESS SHARE | ACCESS EXCLUSIVE | SELECT |
| ROW SHARE | EXCLUSIVE, ACCESS EXCLUSIVE | SELECT FOR UPDATE |
| ROW EXCLUSIVE | SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE | INSERT, UPDATE, DELETE |
| SHARE UPDATE EXCLUSIVE | SHARE UPDATE EXCLUSIVE, SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE | VACUUM, ANALYZE |
| SHARE | ROW EXCLUSIVE, SHARE UPDATE EXCLUSIVE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE | CREATE INDEX |
| SHARE ROW EXCLUSIVE | ROW EXCLUSIVE, SHARE UPDATE EXCLUSIVE, SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE | CREATE TRIGGER |
| EXCLUSIVE | ROW SHARE, ROW EXCLUSIVE, SHARE UPDATE EXCLUSIVE, SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE | Block all but SELECT |
| ACCESS EXCLUSIVE | All | ALTER TABLE, DROP TABLE |

### Row-Level Locks

```sql
-- Exclusive lock on rows
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;

-- Shared lock (other transactions can read but not update)
SELECT * FROM accounts WHERE id = 1 FOR SHARE;

-- No wait (fail immediately if locked)
SELECT * FROM accounts WHERE id = 1 FOR UPDATE NOWAIT;

-- Skip locked rows (process unlocked only)
SELECT * FROM orders WHERE status = 'pending'
FOR UPDATE SKIP LOCKED
LIMIT 10;
```

## Deadlock Detection

```
Transaction A:
1. Lock row 1
2. Request lock row 2 ← BLOCKED (B has it)

Transaction B:
1. Lock row 2
2. Request lock row 1 ← BLOCKED (A has it)

= DEADLOCK
```

### Prevention Strategies

```sql
-- 1. Lock ordering (always lock in same order)
BEGIN;
SELECT * FROM accounts WHERE id IN (1, 2) ORDER BY id FOR UPDATE;
-- Always lock lower ID first

-- 2. Lock timeout
SET lock_timeout = '5s';

-- 3. Retry logic (application)
-- On deadlock, random backoff and retry
```

## Two-Phase Commit (2PC)

For distributed transactions:

```
Phase 1: PREPARE
├── Coordinator sends PREPARE to all nodes
├── Nodes write to log, respond YES/NO
└── If any NO, abort

Phase 2: COMMIT
├── Coordinator sends COMMIT to all nodes
├── Nodes commit and respond ACK
└── Coordinator marks complete
```

## MVCC (Multi-Version Concurrency Control)

PostgreSQL approach:
- Each row has xmin (created by txn) and xmax (deleted by txn)
- Readers don't block writers
- Writers don't block readers
- Old versions kept until no transaction needs them (VACUUM)

```sql
-- Check row visibility
SELECT xmin, xmax, * FROM users;

-- Current transaction ID
SELECT txid_current();
```

## Savepoints

```sql
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE id = 1;
SAVEPOINT after_debit;

UPDATE accounts SET balance = balance + 100 WHERE id = 2;
-- Oops, wrong account

ROLLBACK TO SAVEPOINT after_debit;

UPDATE accounts SET balance = balance + 100 WHERE id = 3;
COMMIT;
```

## Best Practices

1. **Keep transactions short** - Long transactions hold locks
2. **Access resources in consistent order** - Prevents deadlocks
3. **Use appropriate isolation level** - Don't over-isolate
4. **Handle errors properly** - Always rollback on error
5. **Avoid user interaction in transactions** - Don't hold locks waiting for input
6. **Batch large operations** - Commit periodically
