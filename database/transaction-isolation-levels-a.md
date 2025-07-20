# SQL Server Transaction Isolation Levels and Locks

This document explains SQL Server **transaction isolation levels**, their impact on data consistency, and how **shared** and **exclusive locks** work to manage concurrency, including how they lock rows. It includes practical examples using a sample table and addresses common phenomena like dirty reads, phantom reads, non-repeatable reads, and deadlocks.

## Sample Table Setup

To demonstrate isolation levels and locks, we use a sample table created by the following stored procedure:

```sql
ALTER PROCEDURE [dbo].[CreateSampleTable]
AS
BEGIN
    DROP TABLE IF EXISTS [SampleTable]

    CREATE TABLE [SampleTable]
    (
        [Id]          [int] IDENTITY(1,1) NOT NULL,
        [Name]        [varchar](100) NULL,
        [Value]       [varchar](100) NULL,
        [DateChanged] [datetime] DEFAULT(GETDATE()) NULL,
        CONSTRAINT [PK_SampleTable] PRIMARY KEY CLUSTERED ([Id] ASC)
    )

    INSERT INTO SampleTable(Name, Value) 
    SELECT 'Name1', 'Value1'
    UNION ALL 
    SELECT 'Name2', 'Value2'
    UNION ALL 
    SELECT 'Name3', 'Value3'

    SELECT * FROM SampleTable
END
```

**Output**:
| Id | Name  | Value  | DateChanged         |
|----|-------|--------|---------------------|
| 1  | Name1 | Value1 | 2025-07-20 17:46:00 |
| 2  | Name2 | Value2 | 2025-07-20 17:46:00 |
| 3  | Name3 | Value3 | 2025-07-20 17:46:00 |

## Transaction Isolation Levels

Isolation levels define how transactions interact, balancing data consistency with concurrency. They control three phenomena:
- **Dirty Read**: Reading uncommitted (dirty) data from another transaction.
- **Non-Repeatable Read**: Re-reading data in a transaction and finding it changed due to another transaction’s updates.
- **Phantom Read**: Re-reading data and seeing new or missing rows due to another transaction’s inserts or deletes.

### Overview Table

| Isolation Level        | Dirty Read | Phantom Read | Non-Repeatable Read Prevented |
|-----------------------|------------|--------------|-------------------------------|
| Read Uncommitted      | Yes        | Yes          | No                            |
| Read Committed        | No         | Yes          | No                            |
| Repeatable Read       | No         | Yes          | Yes                           |
| Serializable          | No         | No           | Yes                           |
| Snapshot              | No         | No           | Yes                           |

### 1. Read Uncommitted

**Description**: The least strict level, allowing dirty reads, phantom reads, and non-repeatable reads. Ignores locks, reading uncommitted data.

**Example**:
```sql
-- Transaction 1 (T1)
BEGIN TRANSACTION
    INSERT INTO SampleTable(Name, Value) VALUES('Name4', 'Value4') -- Exclusive lock
    UPDATE SampleTable SET Name = Name + Name WHERE Id=1       -- Exclusive lock
    WAITFOR DELAY '00:00:15'
ROLLBACK

-- Transaction 2 (T2, in another session)
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED
SELECT * FROM SampleTable           -- Not blocked, sees Name4 and Name1Name1
SELECT * FROM SampleTable WITH(NOLOCK) -- Same as above
SELECT * FROM SampleTable WHERE Id=2 -- Not blocked
SELECT * FROM SampleTable WHERE Name='Name2' -- Not blocked
SELECT * FROM SampleTable WHERE Id=2 AND Name='Name2' -- Not blocked
```

**Result**: All queries in T2 read uncommitted changes (e.g., `Name4` or `Name1Name1`) without blocking, showing dirty data. After T1 rolls back, T2’s results are inconsistent with the final state.

### 2. Read Committed

**Description**: Prevents dirty reads but allows phantom reads and non-repeatable reads. Shared locks are acquired during reads but released immediately after each row is read.

**Example 1: Blocking Behavior**
```sql
-- Transaction 1 (T1)
BEGIN TRANSACTION
    INSERT INTO SampleTable(Name, Value) VALUES('Name4', 'Value4') -- Exclusive lock
    UPDATE SampleTable SET Name = Name + Name WHERE Id=1       -- Exclusive lock
    WAITFOR DELAY '00:00:15'
ROLLBACK

-- Queries (run in separate sessions)
SELECT * FROM SampleTable                           -- Q1: Blocked
SELECT * FROM SampleTable WITH(NOLOCK)              -- Q2: Not blocked, dirty read
SELECT * FROM SampleTable WHERE Id=2                -- Q3: Not blocked
SELECT * FROM SampleTable WHERE Name='Name2'        -- Q4: May block (no index)
SELECT * FROM SampleTable WHERE Id=2 AND Name='Name2' -- Q5: Not blocked
```

**Result**:
- **Q1**: Blocked due to exclusive locks on modified rows.
- **Q2**: Reads uncommitted data with `NOLOCK`, acting like Read Uncommitted.
- **Q3 & Q5**: Not blocked because `Id=2` is unaffected by T1, and the primary key allows precise access.
- **Q4**: May block if `Name` has no index, requiring a table scan that conflicts with T1’s locks.

**Example 2: Read vs. Write**
```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED
BEGIN TRANSACTION
    SELECT * FROM SampleTable WHERE Id=2 -- Shared lock, released after read
    WAITFOR DELAY '00:00:15'
ROLLBACK

-- Transaction 2
BEGIN TRANSACTION
    INSERT INTO SampleTable(Name, Value) VALUES('Name4', 'Value4')
    UPDATE SampleTable SET Name = Name + Name WHERE Id=2
COMMIT
```

**Result**: T2’s write is not blocked because Read Committed releases shared locks after reading, allowing writes to proceed.

### 3. Repeatable Read

**Description**: Prevents dirty reads and non-repeatable reads by holding shared locks on read rows until the transaction ends. Allows phantom reads (new rows can be inserted).

**Example 1: Insert**
```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ
BEGIN TRANSACTION
    SELECT * FROM SampleTable WHERE Name='Name1' -- Q1: 1 row (Id=1)
    WAITFOR DELAY '00:00:15'
    SELECT * FROM SampleTable WHERE Name='Name1' -- Q2: 2 rows
ROLLBACK

-- Transaction 2
BEGIN TRANSACTION
    INSERT INTO SampleTable(Name, Value) VALUES('Name1', 'Value4')
COMMIT
```

**Result**: T2’s insert is not blocked, and Q2 sees the new row (`Name1`, `Value4`), showing a phantom read.

**Example 2: Delete**
```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ
BEGIN TRANSACTION
    SELECT * FROM SampleTable WHERE Name='Name1' -- Q1: 1 row (Id=1)
    WAITFOR DELAY '00:00:15'
    SELECT * FROM SampleTable WHERE Name='Name1' -- Q2: 1 row
ROLLBACK

-- Transaction 2
BEGIN TRANSACTION
    DELETE FROM SampleTable WHERE Id=1
COMMIT
```

**Result**: T2 is blocked until T1 completes because Repeatable Read’s shared locks on `Id=1` prevent deletes. Q1 and Q2 return the same row.

**Example 3: Update**
```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ
BEGIN TRANSACTION
    SELECT * FROM SampleTable WHERE Name='Name1' -- Q1: 1 row (Id=1)
    WAITFOR DELAY '00:00:15'
    SELECT * FROM SampleTable WHERE Name='Name1' -- Q2: 1 row
ROLLBACK

-- Transaction 2
BEGIN TRANSACTION
    UPDATE SampleTable SET Value = Value + Value WHERE Id=1
COMMIT
```

**Result**: T2 is blocked because Repeatable Read prevents updates to read rows. Q1 and Q2 return the same row.

**Example 4: Update Different Row**
```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ
BEGIN TRANSACTION
    SELECT * FROM SampleTable WHERE Name='Name1' -- Q1: 1 row (Id=1)
    WAITFOR DELAY '00:00:15'
    SELECT * FROM SampleTable WHERE Name='Name1' -- Q2: 2 rows
ROLLBACK

-- Transaction 2
BEGIN TRANSACTION
    UPDATE SampleTable SET Name = 'Name1' WHERE Id=2
COMMIT
```

**Result**: T2 is not blocked because it updates `Id=2`, which T1 didn’t read. Q2 sees the updated row (`Id=2`, `Name='Name1'`), showing a phantom read.

### 4. Serializable

**Description**: The strictest level, preventing dirty reads, non-repeatable reads, and phantom reads. Uses key-range locks to block any inserts, updates, or deletes that could affect the transaction’s results.

**Example**:
```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE
BEGIN TRANSACTION
    SELECT * FROM SampleTable WHERE Name='Name1' -- Shared lock + key-range lock
    WAITFOR DELAY '00:00:15'
    SELECT * FROM SampleTable WHERE Name='Name1'
ROLLBACK

-- Transaction 2
BEGIN TRANSACTION
    INSERT INTO SampleTable(Name, Value) VALUES('Name1', 'Value4')
COMMIT
```

**Result**: T2 is blocked because Serializable locks the key range for `Name='Name1'`, preventing inserts that could cause phantom reads. Both SELECTs return the same rows.

### 5. Snapshot

**Description**: Prevents dirty reads, non-repeatable reads, and phantom reads using row versioning. Reads a consistent snapshot of data from the transaction’s start, stored in `tempdb`, without blocking writes. Requires enabling snapshot isolation:
```sql
ALTER DATABASE TestDb
SET ALLOW_SNAPSHOT_ISOLATION ON
```

**Example**:
```sql
SET TRANSACTION ISOLATION LEVEL SNAPSHOT
BEGIN TRANSACTION
    SELECT * FROM SampleTable WHERE Name='Name1' -- Reads snapshot
    WAITFOR DELAY '00:00:15'
    SELECT * FROM SampleTable WHERE Name='Name1' -- Same snapshot
ROLLBACK

-- Transaction 2
BEGIN TRANSACTION
    INSERT INTO SampleTable(Name, Value) VALUES('Name1', 'Value4')
    UPDATE SampleTable SET Value = 'NewValue' WHERE Id=1
COMMIT
```

**Result**: T2 is not blocked because Snapshot uses row versioning, not locks. Both SELECTs in T1 return the same rows from the snapshot, unaffected by T2’s changes. Snapshot increases `tempdb` usage, making it resource-intensive.

## Shared and Exclusive Locks

Locks manage concurrent access to database resources (rows, pages, or tables) to ensure consistency. SQL Server uses **shared locks** for reads and **exclusive locks** for writes.

### Shared Lock (S)
- **Purpose**: Allows reading data while preventing modifications.
- **Compatibility**: Compatible with other shared locks (multiple transactions can read) but incompatible with exclusive locks.
- **Duration**:
  - **Read Committed**: Released after reading each row.
  - **Repeatable Read/Serializable**: Held until transaction ends.
- **Example**:
  ```sql
  SET TRANSACTION ISOLATION LEVEL READ COMMITTED
  BEGIN TRANSACTION
      SELECT * FROM SampleTable WHERE Id=1 -- Shared lock on Id=1
  COMMIT
  ```
  - Another transaction can read `Id=1` but cannot modify it until the shared lock is released.

### Exclusive Lock (X)
- **Purpose**: Allows modifying data while preventing reads or other modifications.
- **Compatibility**: Incompatible with all other locks (shared or exclusive).
- **Duration**: Held until the transaction commits or rolls back.
- **Example**:
  ```sql
  BEGIN TRANSACTION
      UPDATE SampleTable SET Name='Name1_Updated' WHERE Id=1 -- Exclusive lock
      WAITFOR DELAY '00:00:15'
  COMMIT
  ```
  - No other transaction can read or modify `Id=1` until the transaction completes.

## How Locks Work and Lock Rows

1. **Lock Granularity**:
   - **Row-Level**: Locks specific rows (e.g., `WHERE Id=1` uses the primary key).
   - **Page-Level**: Locks an 8 KB page if multiple rows on it are accessed.
   - **Table-Level**: Locks the entire table if many rows are affected or a table scan is needed. Lock escalation can occur for efficiency (configurable via `ALTER TABLE`).

2. **Row Locking Process**:
   - **Index-Based**: Queries using an index (e.g., primary key) lock specific rows via index keys.
     - Example: `SELECT * FROM SampleTable WHERE Id=1` places a shared lock on `Id=1`.
     - Example: `UPDATE SampleTable SET Name='New' WHERE Id=1` places an exclusive lock.
   - **Non-Indexed Queries**: Queries like `SELECT * FROM SampleTable WHERE Name='Name1'` (no index on `Name`) may require a table scan, locking all rows or pages scanned.
   - **Key-Range Locking** (Serializable): Locks a range of index keys to prevent phantom reads (e.g., `WHERE Name='Name1'` locks all potential `Name='Name1'` rows).

3. **Blocking**: Occurs when a lock request conflicts with an existing lock (e.g., shared lock vs. exclusive lock).

4. **Lock Compatibility**:
   | Requested Lock | Existing Shared Lock | Existing Exclusive Lock |
   |----------------|----------------------|-------------------------|
   | Shared (S)     | Compatible           | Incompatible            |
   | Exclusive (X)  | Incompatible         | Incompatible            |

## Deadlocks in Repeatable Read

**Description**: A deadlock occurs when two transactions hold locks that the other needs, causing a circular wait. SQL Server detects and terminates one transaction.

**Example**:
```sql
-- Transaction 1
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ
BEGIN TRANSACTION
    SELECT * FROM SampleTable WHERE Name='Name1' -- Shared lock on Id=1
    WAITFOR DELAY '00:00:15'
    UPDATE SampleTable SET Value='NewValue' WHERE Id=2 -- Requests exclusive lock
ROLLBACK

-- Transaction 2
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ
BEGIN TRANSACTION
    SELECT * FROM SampleTable WHERE Name='Name2' -- Shared lock on Id=2
    WAITFOR DELAY '00:00:15'
    UPDATE SampleTable SET Value='NewValue' WHERE Id=1 -- Requests exclusive lock
ROLLBACK
```

**Result**:
- T1 holds a shared lock on `Id=1` and requests an exclusive lock on `Id=2`.
- T2 holds a shared lock on `Id=2` and requests an exclusive lock on `Id=1`.
- Neither can proceed, causing a deadlock. SQL Server terminates one transaction (based on `DEADLOCK_PRIORITY` or rollback cost).

## Monitoring Locks

View active locks using:
```sql
SELECT * FROM sys.dm_tran_locks WHERE resource_database_id = DB_ID('TestDb')
```

## Performance Considerations
- **Blocking**: Shared and exclusive locks can reduce concurrency in high-transaction systems.
- **Snapshot Isolation**: Avoids blocking but increases `tempdb` usage.
- **Lock Escalation**: SQL Server may escalate row/page locks to table locks for efficiency. Disable with:
  ```sql
  ALTER TABLE SampleTable SET (LOCK_ESCALATION = DISABLE)
  ```

## Conclusion

Understanding isolation levels and locks is crucial for managing database concurrency:
- **Read Uncommitted**: High concurrency, risks dirty reads.
- **Read Committed**: Balances consistency and concurrency, allows phantom reads.
- **Repeatable Read**: Prevents non-repeatable reads, allows phantom reads, risks deadlocks.
- **Serializable**: Maximum consistency, low concurrency due to key-range locks.
- **Snapshot**: Consistent reads without blocking, but resource-intensive.

Choose the isolation level based on your application’s consistency and performance needs.