# T-SQL Deep Dive

## T-SQL Architecture

- Compiled to query execution plans
- Plans cached in plan cache
- Recompilation when statistics change
- Parameter sniffing for optimization

## Variable Declaration

```sql
-- Basic declaration
DECLARE @var1 INT;
DECLARE @var2 VARCHAR(100) = 'default';
DECLARE @var3 INT, @var4 NVARCHAR(50);

-- Table variable
DECLARE @temp TABLE (
    id INT PRIMARY KEY,
    name NVARCHAR(100),
    INDEX idx_name (name)  -- Index in table variable (2014+)
);

-- Assign from query
DECLARE @max_id INT;
SELECT @max_id = MAX(id) FROM orders;

-- Multiple assignment
SELECT @var1 = col1, @var2 = col2 FROM table1 WHERE id = 1;

-- SET vs SELECT
SET @var1 = (SELECT col1 FROM table1 WHERE id = 1);  -- NULL if no row
SELECT @var1 = col1 FROM table1 WHERE id = 1;        -- Unchanged if no row
```

## Control Flow

### IF...ELSE

```sql
IF @condition = 1
BEGIN
    -- Multiple statements need BEGIN/END
    PRINT 'Condition 1';
    SET @result = 'A';
END
ELSE IF @condition = 2
    PRINT 'Condition 2';  -- Single statement, no BEGIN needed
ELSE
BEGIN
    PRINT 'Other';
    SET @result = 'C';
END;
```

### WHILE

```sql
DECLARE @i INT = 1;
WHILE @i <= 10
BEGIN
    IF @i = 5
        CONTINUE;  -- Skip to next iteration

    IF @i = 8
        BREAK;     -- Exit loop

    PRINT @i;
    SET @i = @i + 1;
END;
```

### GOTO

```sql
GOTO ErrorHandler;

-- ... code ...

ErrorHandler:
    PRINT 'Error occurred';
```

### WAITFOR

```sql
-- Wait for specific time
WAITFOR DELAY '00:00:05';  -- 5 seconds

-- Wait until specific time
WAITFOR TIME '14:30:00';

-- Timeout (2005+)
WAITFOR DELAY '00:00:05' TIMEOUT 3000;  -- 3 second timeout
```

## Error Handling

### TRY...CATCH

```sql
BEGIN TRY
    -- Code that might fail
    INSERT INTO table1 VALUES (1);
END TRY
BEGIN CATCH
    -- Error handling
    SELECT
        ERROR_NUMBER() AS ErrorNumber,
        ERROR_SEVERITY() AS ErrorSeverity,
        ERROR_STATE() AS ErrorState,
        ERROR_LINE() AS ErrorLine,
        ERROR_PROCEDURE() AS ErrorProcedure,
        ERROR_MESSAGE() AS ErrorMessage;
END CATCH;
```

### Nested TRY...CATCH

```sql
BEGIN TRY
    BEGIN TRY
        -- Inner operation
        SELECT 1/0;
    END TRY
    BEGIN CATCH
        -- Can re-throw or handle
        THROW;  -- Re-throw to outer catch
    END CATCH
END TRY
BEGIN CATCH
    -- Outer handler
    PRINT ERROR_MESSAGE();
END CATCH;
```

### XACT_ABORT

```sql
SET XACT_ABORT ON;  -- Auto-rollback on error

BEGIN TRANSACTION;
    INSERT INTO t1 VALUES (1);
    INSERT INTO t1 VALUES (1);  -- Error: entire transaction rolls back
COMMIT;  -- Never reached
```

### XACT_STATE()

```sql
BEGIN TRY
    BEGIN TRANSACTION;
    -- Operations
    COMMIT;
END TRY
BEGIN CATCH
    IF XACT_STATE() = -1
        -- Transaction is uncommittable, must rollback
        ROLLBACK;
    ELSE IF XACT_STATE() = 1
        -- Transaction is committable (could commit partial work)
        ROLLBACK;  -- Usually still want to rollback

    THROW;
END CATCH;
```

## Temporary Tables

### Local Temp Tables (#)

```sql
-- Visible only in current session
CREATE TABLE #temp (id INT, name VARCHAR(100));

INSERT INTO #temp VALUES (1, 'Test');

-- Automatically dropped when session ends
-- Or explicitly
DROP TABLE #temp;
```

### Global Temp Tables (##)

```sql
-- Visible to all sessions
CREATE TABLE ##global_temp (id INT);

-- Dropped when creating session ends AND no other sessions using it
```

### Table Variables

```sql
DECLARE @tv TABLE (
    id INT PRIMARY KEY,
    name VARCHAR(100)
);

INSERT INTO @tv VALUES (1, 'Test');

-- Scope: current batch only
-- Statistics: No (uses estimate of 1 row)
-- Transaction: Follows transaction
```

### Temp Table vs Table Variable

| Feature | #Temp Table | @Table Variable |
|---------|-------------|-----------------|
| Statistics | Yes | No |
| Indexes | Full support | Limited (2014+) |
| Parallelism | Yes | Limited |
| Transaction log | More | Less |
| Scope | Session | Batch |
| Use case | Large data | Small data |

## Common Table Expressions

```sql
-- Basic CTE
;WITH cte AS (
    SELECT * FROM employees WHERE department_id = 10
)
SELECT * FROM cte;

-- Multiple CTEs
;WITH
cte1 AS (SELECT * FROM t1),
cte2 AS (SELECT * FROM t2),
cte3 AS (SELECT * FROM cte1 JOIN cte2 ON cte1.id = cte2.id)
SELECT * FROM cte3;

-- Recursive CTE
;WITH hierarchy AS (
    -- Anchor
    SELECT id, name, manager_id, 0 AS level
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive
    SELECT e.id, e.name, e.manager_id, h.level + 1
    FROM employees e
    JOIN hierarchy h ON e.manager_id = h.id
)
SELECT * FROM hierarchy
OPTION (MAXRECURSION 100);  -- Default is 100, 0 for unlimited
```

## Query Hints

### Table Hints

```sql
SELECT * FROM orders WITH (NOLOCK);           -- Dirty reads OK
SELECT * FROM orders WITH (READPAST);         -- Skip locked rows
SELECT * FROM orders WITH (UPDLOCK);          -- Update lock
SELECT * FROM orders WITH (HOLDLOCK);         -- Hold until commit
SELECT * FROM orders WITH (INDEX(idx_name));  -- Force index
```

### Query Hints

```sql
SELECT * FROM orders
OPTION (
    RECOMPILE,                    -- Recompile this execution
    MAXDOP 4,                     -- Max parallelism
    OPTIMIZE FOR (@param = 100),  -- Optimize for specific value
    FAST 10,                      -- Optimize for first 10 rows
    FORCE ORDER                   -- Use join order as written
);
```

### Join Hints

```sql
SELECT * FROM t1
INNER LOOP JOIN t2 ON t1.id = t2.id;   -- Force nested loop
INNER HASH JOIN t2 ON t1.id = t2.id;   -- Force hash join
INNER MERGE JOIN t2 ON t1.id = t2.id;  -- Force merge join
```

## OUTPUT Clause

```sql
-- Capture inserted rows
INSERT INTO orders (customer_id, amount)
OUTPUT INSERTED.order_id, INSERTED.created_at
VALUES (1, 100.00);

-- Capture deleted rows
DELETE FROM orders
OUTPUT DELETED.*
WHERE status = 'cancelled';

-- Capture both (UPDATE)
UPDATE orders
SET status = 'shipped'
OUTPUT DELETED.status AS old_status, INSERTED.status AS new_status
WHERE order_id = 123;

-- Into table
DECLARE @changes TABLE (order_id INT, old_status VARCHAR(20), new_status VARCHAR(20));

UPDATE orders
SET status = 'shipped'
OUTPUT INSERTED.order_id, DELETED.status, INSERTED.status INTO @changes
WHERE status = 'pending';
```

## MERGE Statement

```sql
MERGE INTO target_table AS target
USING source_table AS source
ON target.id = source.id

WHEN MATCHED AND source.delete_flag = 1 THEN
    DELETE

WHEN MATCHED THEN
    UPDATE SET target.name = source.name

WHEN NOT MATCHED BY TARGET THEN
    INSERT (id, name) VALUES (source.id, source.name)

WHEN NOT MATCHED BY SOURCE THEN
    DELETE

OUTPUT $action, INSERTED.*, DELETED.*;
```

## Window Functions

```sql
SELECT
    *,
    ROW_NUMBER() OVER (ORDER BY id) AS rn,
    ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC) AS dept_rank,
    LAG(salary, 1, 0) OVER (ORDER BY hire_date) AS prev_salary,
    LEAD(salary, 1, 0) OVER (ORDER BY hire_date) AS next_salary,
    SUM(salary) OVER (ORDER BY hire_date ROWS UNBOUNDED PRECEDING) AS running_total,
    AVG(salary) OVER (ORDER BY hire_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg
FROM employees;
```

## JSON Support (2016+)

```sql
DECLARE @json NVARCHAR(MAX) = '{"name":"John","age":30}';

-- Check valid JSON
SELECT ISJSON(@json);

-- Extract values
SELECT JSON_VALUE(@json, '$.name');    -- Scalar value
SELECT JSON_QUERY(@json, '$.address'); -- Object or array

-- Modify JSON
SELECT JSON_MODIFY(@json, '$.age', 31);

-- Parse JSON array
SELECT * FROM OPENJSON('[1,2,3]');

-- Parse JSON object
SELECT * FROM OPENJSON(@json)
WITH (
    name NVARCHAR(100) '$.name',
    age INT '$.age'
);

-- Generate JSON
SELECT * FROM employees FOR JSON AUTO;
SELECT * FROM employees FOR JSON PATH, ROOT('employees');
```
