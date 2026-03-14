# SQL Server Deep Dive

## SQL Server Architecture

### Database Engine Components

```
SQL Server Instance
├── Protocol Layer (TDS - Tabular Data Stream)
├── Relational Engine
│   ├── Query Parser
│   ├── Query Optimizer
│   └── Query Executor
├── Storage Engine
│   ├── Access Methods (heap, B-tree, columnstore)
│   ├── Buffer Manager
│   ├── Transaction Manager
│   └── Lock Manager
└── SQLOS (Memory, Scheduling, I/O)
```

### Memory Architecture

| Component | Function |
|-----------|----------|
| Buffer Pool | Data page cache |
| Plan Cache | Execution plans |
| Log Cache | Transaction log buffer |
| Lock Memory | Lock structures |
| Connection Memory | Per-connection buffers |

## SQL Server Data Types

### Exact Numerics

```sql
-- Integer types
BIGINT              -- -2^63 to 2^63-1 (8 bytes)
INT                 -- -2^31 to 2^31-1 (4 bytes)
SMALLINT            -- -32,768 to 32,767 (2 bytes)
TINYINT             -- 0 to 255 (1 byte)
BIT                 -- 0, 1, or NULL

-- Decimal types
DECIMAL(18,4)       -- Precision 1-38, scale 0-38
NUMERIC(18,4)       -- Same as DECIMAL
MONEY               -- -922T to 922T, 4 decimal places
SMALLMONEY          -- -214K to 214K, 4 decimal places
```

### Approximate Numerics

```sql
FLOAT(24)           -- 7 digits precision (4 bytes)
FLOAT(53)           -- 15 digits precision (8 bytes), default
REAL                -- Same as FLOAT(24)
```

### Date and Time

```sql
-- Date only
DATE                -- 0001-01-01 to 9999-12-31 (3 bytes)

-- Time only
TIME(7)             -- 00:00:00.0000000 to 23:59:59.9999999

-- Date + Time
DATETIME            -- 1753-01-01 to 9999-12-31, 3.33ms precision
DATETIME2(7)        -- 0001-01-01 to 9999-12-31, 100ns precision
SMALLDATETIME       -- 1900-01-01 to 2079-06-06, minute precision
DATETIMEOFFSET(7)   -- DATETIME2 with timezone offset

-- Examples
DECLARE @d DATE = '2024-01-15';
DECLARE @t TIME(3) = '14:30:00.123';
DECLARE @dt DATETIME2(3) = '2024-01-15 14:30:00.123';
DECLARE @dto DATETIMEOFFSET = '2024-01-15 14:30:00 -05:00';
```

### String Types

```sql
-- Non-Unicode
CHAR(10)            -- Fixed, max 8000
VARCHAR(100)        -- Variable, max 8000
VARCHAR(MAX)        -- Variable, up to 2GB
TEXT                -- Deprecated, use VARCHAR(MAX)

-- Unicode
NCHAR(10)           -- Fixed, max 4000
NVARCHAR(100)       -- Variable, max 4000
NVARCHAR(MAX)       -- Variable, up to 2GB
NTEXT               -- Deprecated, use NVARCHAR(MAX)
```

### Binary Types

```sql
BINARY(10)          -- Fixed, max 8000 bytes
VARBINARY(100)      -- Variable, max 8000 bytes
VARBINARY(MAX)      -- Variable, up to 2GB
IMAGE               -- Deprecated, use VARBINARY(MAX)
```

### Special Types

```sql
UNIQUEIDENTIFIER    -- 16-byte GUID
HIERARCHYID         -- Hierarchical position
GEOMETRY            -- Planar spatial data
GEOGRAPHY           -- Geodetic spatial data
XML                 -- XML data with optional schema
SQL_VARIANT         -- Can store any type
```

## Index Types

### Clustered Index

```sql
-- Table physically sorted by clustered index
CREATE CLUSTERED INDEX ix_orders_date
ON orders (order_date);

-- Primary key creates clustered by default
ALTER TABLE orders ADD CONSTRAINT pk_orders
PRIMARY KEY CLUSTERED (order_id);

-- Only one clustered index per table
```

### Nonclustered Index

```sql
-- B-tree index with pointer to data
CREATE NONCLUSTERED INDEX ix_orders_customer
ON orders (customer_id);

-- Composite index
CREATE INDEX ix_orders_cust_date
ON orders (customer_id, order_date);

-- With included columns (covering index)
CREATE INDEX ix_orders_customer_inc
ON orders (customer_id)
INCLUDE (order_date, total_amount);

-- Filtered index
CREATE INDEX ix_orders_pending
ON orders (order_date)
WHERE status = 'pending';
```

### Columnstore Index

```sql
-- For analytical workloads (data warehouse)
CREATE CLUSTERED COLUMNSTORE INDEX cci_sales
ON sales;

-- Nonclustered columnstore
CREATE NONCLUSTERED COLUMNSTORE INDEX ncci_orders
ON orders (order_date, customer_id, total_amount);

-- Hybrid: Columnstore with B-tree for OLTP
-- Use filtered nonclustered columnstore
```

### Unique Index

```sql
CREATE UNIQUE INDEX uix_users_email
ON users (email);

-- With NULL handling (only one NULL allowed)
CREATE UNIQUE INDEX uix_users_ssn
ON users (ssn)
WHERE ssn IS NOT NULL;  -- Multiple NULLs allowed
```

### Index Options

```sql
CREATE INDEX ix_orders_date ON orders (order_date)
WITH (
    FILLFACTOR = 80,           -- Leave 20% free space
    PAD_INDEX = ON,            -- Apply fillfactor to intermediate pages
    SORT_IN_TEMPDB = ON,       -- Use tempdb for sort
    ONLINE = ON,               -- No blocking (Enterprise)
    DATA_COMPRESSION = PAGE,   -- Page compression
    RESUMABLE = ON             -- Can pause and resume (2019+)
);
```

## Partitioning

### Create Partition Function

```sql
-- Define boundaries
CREATE PARTITION FUNCTION pf_date_monthly (DATE)
AS RANGE RIGHT FOR VALUES (
    '2023-01-01', '2023-02-01', '2023-03-01',
    '2023-04-01', '2023-05-01', '2023-06-01',
    '2023-07-01', '2023-08-01', '2023-09-01',
    '2023-10-01', '2023-11-01', '2023-12-01'
);

-- RANGE LEFT: values <= boundary go left
-- RANGE RIGHT: values >= boundary go right
```

### Create Partition Scheme

```sql
-- Map partitions to filegroups
CREATE PARTITION SCHEME ps_date_monthly
AS PARTITION pf_date_monthly
TO (fg_2022, fg_jan, fg_feb, fg_mar, fg_apr, fg_may,
    fg_jun, fg_jul, fg_aug, fg_sep, fg_oct, fg_nov, fg_dec);

-- Or all to one filegroup
CREATE PARTITION SCHEME ps_date_monthly_single
AS PARTITION pf_date_monthly
ALL TO ([PRIMARY]);
```

### Create Partitioned Table

```sql
CREATE TABLE orders (
    order_id INT IDENTITY PRIMARY KEY NONCLUSTERED,
    order_date DATE NOT NULL,
    customer_id INT NOT NULL,
    total_amount MONEY
) ON ps_date_monthly (order_date);

-- Clustered index on partition key
CREATE CLUSTERED INDEX cix_orders_date
ON orders (order_date)
ON ps_date_monthly (order_date);
```

### Partition Maintenance

```sql
-- Check partition info
SELECT
    $PARTITION.pf_date_monthly('2023-06-15') AS partition_number;

SELECT
    partition_number,
    rows
FROM sys.partitions
WHERE object_id = OBJECT_ID('orders');

-- Split partition (add new boundary)
ALTER PARTITION FUNCTION pf_date_monthly()
SPLIT RANGE ('2024-01-01');

-- Merge partitions (remove boundary)
ALTER PARTITION FUNCTION pf_date_monthly()
MERGE RANGE ('2023-01-01');

-- Switch partition (instant move to archive)
ALTER TABLE orders
SWITCH PARTITION 1 TO orders_archive PARTITION 1;
```

## Temporal Tables (System-Versioned)

```sql
-- Create temporal table
CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    name NVARCHAR(100),
    salary MONEY,

    -- System time columns
    valid_from DATETIME2 GENERATED ALWAYS AS ROW START NOT NULL,
    valid_to DATETIME2 GENERATED ALWAYS AS ROW END NOT NULL,
    PERIOD FOR SYSTEM_TIME (valid_from, valid_to)
)
WITH (SYSTEM_VERSIONING = ON (
    HISTORY_TABLE = dbo.employees_history
));

-- Query current data
SELECT * FROM employees;

-- Query at specific point in time
SELECT * FROM employees
FOR SYSTEM_TIME AS OF '2024-01-15 10:00:00';

-- Query all versions
SELECT * FROM employees
FOR SYSTEM_TIME ALL
WHERE employee_id = 100;

-- Query between timestamps
SELECT * FROM employees
FOR SYSTEM_TIME BETWEEN '2024-01-01' AND '2024-01-31';

-- Turn off versioning for maintenance
ALTER TABLE employees SET (SYSTEM_VERSIONING = OFF);
-- Make changes to structure
ALTER TABLE employees SET (SYSTEM_VERSIONING = ON);
```

## In-Memory OLTP

```sql
-- Create memory-optimized filegroup
ALTER DATABASE mydb
ADD FILEGROUP mydb_mod CONTAINS MEMORY_OPTIMIZED_DATA;

ALTER DATABASE mydb
ADD FILE (NAME='mydb_mod1', FILENAME='C:\data\mydb_mod1')
TO FILEGROUP mydb_mod;

-- Create memory-optimized table
CREATE TABLE orders_hot (
    order_id INT NOT NULL PRIMARY KEY NONCLUSTERED
        HASH WITH (BUCKET_COUNT = 1000000),
    customer_id INT NOT NULL INDEX ix_customer HASH WITH (BUCKET_COUNT = 100000),
    order_date DATETIME2 NOT NULL,
    status NVARCHAR(20) NOT NULL
) WITH (MEMORY_OPTIMIZED = ON, DURABILITY = SCHEMA_AND_DATA);

-- DURABILITY options:
-- SCHEMA_AND_DATA: Survives restart
-- SCHEMA_ONLY: Data lost on restart (temp tables)

-- Natively compiled stored procedure
CREATE PROCEDURE usp_insert_order
    @customer_id INT,
    @order_date DATETIME2
WITH NATIVE_COMPILATION, SCHEMABINDING
AS BEGIN ATOMIC WITH (
    TRANSACTION ISOLATION LEVEL = SNAPSHOT,
    LANGUAGE = N'English'
)
    INSERT INTO dbo.orders_hot (customer_id, order_date, status)
    VALUES (@customer_id, @order_date, N'pending');
END;
```

## Query Store

```sql
-- Enable Query Store
ALTER DATABASE mydb SET QUERY_STORE = ON;

ALTER DATABASE mydb SET QUERY_STORE (
    OPERATION_MODE = READ_WRITE,
    CLEANUP_POLICY = (STALE_QUERY_THRESHOLD_DAYS = 30),
    DATA_FLUSH_INTERVAL_SECONDS = 900,
    INTERVAL_LENGTH_MINUTES = 60,
    MAX_STORAGE_SIZE_MB = 1000,
    QUERY_CAPTURE_MODE = AUTO,
    SIZE_BASED_CLEANUP_MODE = AUTO,
    WAIT_STATS_CAPTURE_MODE = ON
);

-- View query performance
SELECT
    q.query_id,
    qt.query_sql_text,
    rs.avg_duration,
    rs.avg_cpu_time,
    rs.avg_logical_io_reads
FROM sys.query_store_query q
JOIN sys.query_store_query_text qt ON q.query_text_id = qt.query_text_id
JOIN sys.query_store_plan p ON q.query_id = p.query_id
JOIN sys.query_store_runtime_stats rs ON p.plan_id = rs.plan_id
ORDER BY rs.avg_duration DESC;

-- Force a plan
EXEC sp_query_store_force_plan @query_id = 1, @plan_id = 1;

-- Unforce
EXEC sp_query_store_unforce_plan @query_id = 1, @plan_id = 1;
```

## Window Functions

```sql
-- Ranking functions
SELECT
    employee_id,
    department_id,
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS row_num,
    RANK() OVER (ORDER BY salary DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank,
    NTILE(4) OVER (ORDER BY salary DESC) AS quartile,

    -- Within partition
    ROW_NUMBER() OVER (PARTITION BY department_id ORDER BY salary DESC) AS dept_rank
FROM employees;

-- Aggregate window functions
SELECT
    order_id,
    order_date,
    amount,
    SUM(amount) OVER (ORDER BY order_date) AS running_total,
    AVG(amount) OVER (ORDER BY order_date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg,
    SUM(amount) OVER (PARTITION BY customer_id) AS customer_total
FROM orders;

-- Value functions
SELECT
    employee_id,
    hire_date,
    salary,
    LAG(salary, 1, 0) OVER (ORDER BY hire_date) AS prev_salary,
    LEAD(salary, 1, 0) OVER (ORDER BY hire_date) AS next_salary,
    FIRST_VALUE(salary) OVER (ORDER BY hire_date) AS first_salary,
    LAST_VALUE(salary) OVER (ORDER BY hire_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS last_salary
FROM employees;
```

## JSON Support

```sql
-- Check valid JSON
SELECT ISJSON('{"name":"John","age":30}');

-- Extract values
SELECT JSON_VALUE('{"name":"John","age":30}', '$.name');      -- Scalar
SELECT JSON_QUERY('{"user":{"name":"John"}}', '$.user');      -- Object

-- Modify JSON
SELECT JSON_MODIFY('{"name":"John"}', '$.age', 30);

-- Parse JSON array
SELECT * FROM OPENJSON('[1,2,3,4,5]');

-- Parse with schema
SELECT * FROM OPENJSON('{"name":"John","age":30}')
WITH (
    name NVARCHAR(100) '$.name',
    age INT '$.age'
);

-- Generate JSON
SELECT name, email
FROM users
FOR JSON PATH;              -- [{...}, {...}]

SELECT name, email
FROM users
FOR JSON AUTO;              -- Infers structure from joins

-- With root element
SELECT name, email
FROM users
FOR JSON PATH, ROOT('users');
```

## Error Handling

```sql
BEGIN TRY
    BEGIN TRANSACTION;

    -- Operations
    INSERT INTO orders VALUES (...);
    UPDATE inventory SET qty = qty - 1 WHERE product_id = 1;

    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0
        ROLLBACK TRANSACTION;

    -- Get error info
    DECLARE @ErrorMessage NVARCHAR(4000) = ERROR_MESSAGE();
    DECLARE @ErrorSeverity INT = ERROR_SEVERITY();
    DECLARE @ErrorState INT = ERROR_STATE();
    DECLARE @ErrorLine INT = ERROR_LINE();
    DECLARE @ErrorProcedure NVARCHAR(200) = ERROR_PROCEDURE();
    DECLARE @ErrorNumber INT = ERROR_NUMBER();

    -- Log error
    INSERT INTO error_log (message, severity, state, line_number, procedure_name)
    VALUES (@ErrorMessage, @ErrorSeverity, @ErrorState, @ErrorLine, @ErrorProcedure);

    -- Re-throw (2012+)
    THROW;

    -- Or raise custom error
    -- RAISERROR(@ErrorMessage, @ErrorSeverity, @ErrorState);
END CATCH;
```

## Service Broker

```sql
-- Create message types
CREATE MESSAGE TYPE [OrderRequest]
    VALIDATION = WELL_FORMED_XML;
CREATE MESSAGE TYPE [OrderResponse]
    VALIDATION = WELL_FORMED_XML;

-- Create contract
CREATE CONTRACT [OrderContract] (
    [OrderRequest] SENT BY INITIATOR,
    [OrderResponse] SENT BY TARGET
);

-- Create queues
CREATE QUEUE OrderRequestQueue;
CREATE QUEUE OrderResponseQueue;

-- Create services
CREATE SERVICE [OrderService]
    ON QUEUE OrderRequestQueue ([OrderContract]);
CREATE SERVICE [OrderProcessingService]
    ON QUEUE OrderResponseQueue ([OrderContract]);

-- Send message
DECLARE @dialog_handle UNIQUEIDENTIFIER;

BEGIN DIALOG @dialog_handle
    FROM SERVICE [OrderService]
    TO SERVICE 'OrderProcessingService'
    ON CONTRACT [OrderContract];

SEND ON CONVERSATION @dialog_handle
    MESSAGE TYPE [OrderRequest]
    ('<order><id>123</id></order>');

-- Receive message
RECEIVE TOP(1)
    conversation_handle,
    message_type_name,
    message_body
FROM OrderRequestQueue;
```
