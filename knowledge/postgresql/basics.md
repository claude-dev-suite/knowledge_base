# PostgreSQL Fundamentals

> Essential PostgreSQL concepts, SQL patterns, and best practices for application development.

## Table of Contents

1. [Data Types](#data-types)
2. [Table Design](#table-design)
3. [Constraints](#constraints)
4. [Indexes](#indexes)
5. [Queries](#queries)
6. [Transactions](#transactions)
7. [JSON and JSONB](#json-and-jsonb)
8. [Common Table Expressions](#common-table-expressions)
9. [Window Functions](#window-functions)
10. [Full-Text Search](#full-text-search)

---

## Data Types

### Commonly Used Types

```sql
-- Numeric Types
CREATE TABLE numeric_examples (
    id SERIAL PRIMARY KEY,                    -- Auto-incrementing integer
    id_big BIGSERIAL,                         -- Auto-incrementing bigint
    small_num SMALLINT,                       -- -32768 to 32767
    regular_num INTEGER,                      -- -2147483648 to 2147483647
    big_num BIGINT,                           -- Very large integers
    decimal_num NUMERIC(10, 2),               -- Exact decimal (10 digits, 2 decimal places)
    float_num REAL,                           -- 6 decimal digits precision
    double_num DOUBLE PRECISION               -- 15 decimal digits precision
);

-- Text Types
CREATE TABLE text_examples (
    fixed_char CHAR(10),                      -- Fixed-length, space-padded
    variable_char VARCHAR(255),               -- Variable-length with limit
    unlimited_text TEXT,                      -- Unlimited length
    uuid_col UUID DEFAULT gen_random_uuid()   -- UUID type
);

-- Date/Time Types
CREATE TABLE datetime_examples (
    date_only DATE,                           -- YYYY-MM-DD
    time_only TIME,                           -- HH:MI:SS
    timestamp_tz TIMESTAMPTZ,                 -- With timezone (recommended)
    timestamp_no_tz TIMESTAMP,                -- Without timezone
    interval_col INTERVAL,                    -- Time interval
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Boolean and Binary
CREATE TABLE misc_examples (
    is_active BOOLEAN DEFAULT true,
    binary_data BYTEA,                        -- Binary data
    ip_address INET,                          -- IP address
    mac_address MACADDR,                      -- MAC address
    network CIDR                              -- Network address
);

-- Arrays
CREATE TABLE array_examples (
    tags TEXT[],                              -- Array of text
    scores INTEGER[],                         -- Array of integers
    matrix INTEGER[][]                        -- Multi-dimensional array
);

-- JSON Types
CREATE TABLE json_examples (
    data JSON,                                -- Stores exact input
    metadata JSONB                            -- Binary JSON (indexable, faster)
);

-- Enum Types
CREATE TYPE status_type AS ENUM ('pending', 'active', 'completed', 'cancelled');
CREATE TABLE enum_example (
    status status_type DEFAULT 'pending'
);
```

### Type Conversions

```sql
-- Explicit casting
SELECT '123'::INTEGER;
SELECT CAST('2024-01-15' AS DATE);
SELECT 123::TEXT;

-- Common conversions
SELECT
    '100'::INTEGER,                           -- String to integer
    100::TEXT,                                -- Integer to string
    '2024-01-15'::DATE,                       -- String to date
    NOW()::DATE,                              -- Timestamp to date
    NOW()::TIME,                              -- Timestamp to time
    'true'::BOOLEAN,                          -- String to boolean
    '{"a": 1}'::JSONB;                        -- String to JSONB
```

---

## Table Design

### Basic Table Creation

```sql
-- Users table with common patterns
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    uuid UUID DEFAULT gen_random_uuid() UNIQUE,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    is_active BOOLEAN DEFAULT true,
    role VARCHAR(50) DEFAULT 'user',
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Add comment for documentation
COMMENT ON TABLE users IS 'Application users';
COMMENT ON COLUMN users.uuid IS 'Public identifier for API responses';

-- Create updated_at trigger
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at();
```

### Relationships

```sql
-- One-to-Many: User has many orders
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    status VARCHAR(50) DEFAULT 'pending',
    total_amount NUMERIC(10, 2) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_orders_user_id ON orders(user_id);

-- Many-to-Many: Products and Categories
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    price NUMERIC(10, 2) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE categories (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE
);

CREATE TABLE product_categories (
    product_id BIGINT REFERENCES products(id) ON DELETE CASCADE,
    category_id BIGINT REFERENCES categories(id) ON DELETE CASCADE,
    PRIMARY KEY (product_id, category_id)
);

-- One-to-One: User profile
CREATE TABLE user_profiles (
    user_id BIGINT PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    bio TEXT,
    avatar_url VARCHAR(500),
    social_links JSONB DEFAULT '{}'
);
```

### Partitioning

```sql
-- Range partitioning by date
CREATE TABLE events (
    id BIGSERIAL,
    event_type VARCHAR(50),
    payload JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2024_01 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE events_2024_02 PARTITION OF events
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- List partitioning by region
CREATE TABLE sales (
    id BIGSERIAL,
    region VARCHAR(50),
    amount NUMERIC(10, 2),
    sale_date DATE
) PARTITION BY LIST (region);

CREATE TABLE sales_na PARTITION OF sales
    FOR VALUES IN ('US', 'CA', 'MX');

CREATE TABLE sales_eu PARTITION OF sales
    FOR VALUES IN ('UK', 'DE', 'FR');
```

---

## Constraints

### All Constraint Types

```sql
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,                                    -- Primary key

    sku VARCHAR(50) UNIQUE NOT NULL,                             -- Unique + Not null

    name VARCHAR(255) NOT NULL,

    price NUMERIC(10, 2) CHECK (price >= 0),                     -- Check constraint

    discount_price NUMERIC(10, 2),
    CONSTRAINT valid_discount CHECK (discount_price < price),    -- Named check

    category_id BIGINT REFERENCES categories(id)                 -- Foreign key
        ON DELETE SET NULL
        ON UPDATE CASCADE,

    -- Composite unique constraint
    CONSTRAINT unique_name_category UNIQUE (name, category_id)
);

-- Add constraint to existing table
ALTER TABLE products
    ADD CONSTRAINT positive_price CHECK (price > 0);

-- Remove constraint
ALTER TABLE products DROP CONSTRAINT positive_price;

-- Add foreign key
ALTER TABLE orders
    ADD CONSTRAINT fk_orders_user
    FOREIGN KEY (user_id) REFERENCES users(id)
    ON DELETE CASCADE;

-- Deferrable constraints (checked at commit)
ALTER TABLE orders
    ADD CONSTRAINT fk_orders_user
    FOREIGN KEY (user_id) REFERENCES users(id)
    DEFERRABLE INITIALLY DEFERRED;
```

### Foreign Key Actions

```sql
-- ON DELETE options:
-- CASCADE    - Delete child rows
-- SET NULL   - Set FK to NULL
-- SET DEFAULT - Set FK to default value
-- RESTRICT   - Prevent deletion (default)
-- NO ACTION  - Same as RESTRICT, but deferrable

CREATE TABLE comments (
    id BIGSERIAL PRIMARY KEY,
    post_id BIGINT REFERENCES posts(id) ON DELETE CASCADE,
    author_id BIGINT REFERENCES users(id) ON DELETE SET NULL
);
```

---

## Indexes

### Index Types

```sql
-- B-tree (default) - equality, range, ordering
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_date ON orders(created_at DESC);

-- Multi-column index
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- Partial index - smaller, faster for specific queries
CREATE INDEX idx_active_users ON users(email)
    WHERE is_active = true;

CREATE INDEX idx_pending_orders ON orders(created_at)
    WHERE status = 'pending';

-- Covering index (PostgreSQL 11+)
CREATE INDEX idx_orders_covering ON orders(user_id)
    INCLUDE (status, total_amount);

-- Expression index
CREATE INDEX idx_users_lower_email ON users(LOWER(email));
CREATE INDEX idx_orders_month ON orders(DATE_TRUNC('month', created_at));

-- GIN index for arrays and JSONB
CREATE INDEX idx_products_tags ON products USING GIN(tags);
CREATE INDEX idx_users_metadata ON users USING GIN(metadata);
CREATE INDEX idx_users_metadata_path ON users USING GIN(metadata jsonb_path_ops);

-- GiST index for geometric/full-text
CREATE INDEX idx_documents_search ON documents USING GiST(to_tsvector('english', content));

-- BRIN index for large naturally ordered data
CREATE INDEX idx_logs_timestamp ON logs USING BRIN(created_at);

-- Unique index
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);

-- Concurrent index creation (no locks)
CREATE INDEX CONCURRENTLY idx_orders_amount ON orders(total_amount);
```

### Index Management

```sql
-- Check index usage
SELECT
    indexrelname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY idx_scan DESC;

-- Find unused indexes
SELECT
    indexrelname,
    idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%_pkey';

-- Drop unused index
DROP INDEX CONCURRENTLY idx_unused;

-- Reindex
REINDEX INDEX CONCURRENTLY idx_orders_date;
REINDEX TABLE CONCURRENTLY orders;
```

---

## Queries

### Basic CRUD Operations

```sql
-- INSERT
INSERT INTO users (email, first_name, last_name)
VALUES ('john@example.com', 'John', 'Doe')
RETURNING id, uuid, created_at;

-- INSERT multiple rows
INSERT INTO users (email, first_name) VALUES
    ('user1@example.com', 'User 1'),
    ('user2@example.com', 'User 2'),
    ('user3@example.com', 'User 3');

-- INSERT from SELECT
INSERT INTO user_backups (user_id, email, backed_up_at)
SELECT id, email, NOW()
FROM users
WHERE is_active = true;

-- UPSERT (INSERT ... ON CONFLICT)
INSERT INTO users (email, first_name)
VALUES ('john@example.com', 'John')
ON CONFLICT (email)
DO UPDATE SET
    first_name = EXCLUDED.first_name,
    updated_at = NOW();

-- UPDATE
UPDATE users
SET first_name = 'Jane', updated_at = NOW()
WHERE id = 1
RETURNING *;

-- UPDATE with JOIN
UPDATE orders o
SET status = 'cancelled'
FROM users u
WHERE o.user_id = u.id
  AND u.is_active = false;

-- DELETE
DELETE FROM users WHERE id = 1 RETURNING *;

-- DELETE with JOIN
DELETE FROM orders
USING users
WHERE orders.user_id = users.id
  AND users.is_active = false;

-- TRUNCATE (fast delete all)
TRUNCATE TABLE logs;
TRUNCATE TABLE orders, order_items RESTART IDENTITY CASCADE;
```

### SELECT Patterns

```sql
-- Basic SELECT with filtering
SELECT id, email, first_name
FROM users
WHERE is_active = true
  AND created_at > NOW() - INTERVAL '30 days'
ORDER BY created_at DESC
LIMIT 10
OFFSET 20;

-- Aggregations
SELECT
    status,
    COUNT(*) AS order_count,
    SUM(total_amount) AS total_revenue,
    AVG(total_amount) AS avg_order_value,
    MIN(created_at) AS first_order,
    MAX(created_at) AS last_order
FROM orders
GROUP BY status
HAVING COUNT(*) > 10
ORDER BY total_revenue DESC;

-- JOINs
SELECT
    u.email,
    COUNT(o.id) AS order_count,
    COALESCE(SUM(o.total_amount), 0) AS total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.is_active = true
GROUP BY u.id, u.email
ORDER BY total_spent DESC;

-- Subquery
SELECT *
FROM users
WHERE id IN (
    SELECT user_id
    FROM orders
    WHERE total_amount > 1000
);

-- EXISTS (usually faster than IN)
SELECT *
FROM users u
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.user_id = u.id
      AND o.total_amount > 1000
);

-- CASE expressions
SELECT
    email,
    CASE
        WHEN total_orders >= 100 THEN 'platinum'
        WHEN total_orders >= 50 THEN 'gold'
        WHEN total_orders >= 10 THEN 'silver'
        ELSE 'bronze'
    END AS tier
FROM (
    SELECT u.email, COUNT(o.id) AS total_orders
    FROM users u
    LEFT JOIN orders o ON u.id = o.user_id
    GROUP BY u.id, u.email
) user_stats;

-- COALESCE and NULLIF
SELECT
    COALESCE(first_name, 'Unknown') AS name,
    NULLIF(status, '') AS status  -- Returns NULL if status is empty string
FROM users;
```

### Pagination

```sql
-- Offset pagination (simple but slow for large offsets)
SELECT * FROM orders
ORDER BY created_at DESC
LIMIT 20 OFFSET 100;

-- Keyset pagination (faster, recommended)
SELECT * FROM orders
WHERE created_at < '2024-01-15 10:00:00'
ORDER BY created_at DESC
LIMIT 20;

-- With cursor-based ID
SELECT * FROM orders
WHERE (created_at, id) < ('2024-01-15 10:00:00', 12345)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

---

## Transactions

### Basic Transactions

```sql
-- Explicit transaction
BEGIN;

INSERT INTO accounts (user_id, balance) VALUES (1, 1000);
INSERT INTO transactions (account_id, amount, type) VALUES (1, 1000, 'deposit');

COMMIT;

-- Rollback on error
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

-- If something goes wrong
ROLLBACK;

-- Savepoints for partial rollback
BEGIN;

INSERT INTO orders (user_id, total_amount) VALUES (1, 100);
SAVEPOINT order_created;

INSERT INTO order_items (order_id, product_id, quantity) VALUES (1, 1, 2);
-- Error occurred
ROLLBACK TO SAVEPOINT order_created;

-- Continue with different items
INSERT INTO order_items (order_id, product_id, quantity) VALUES (1, 2, 1);

COMMIT;
```

### Transaction Isolation Levels

```sql
-- Read Committed (default)
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- Sees committed changes from other transactions during execution

-- Repeatable Read
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- Consistent snapshot throughout transaction

-- Serializable
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- Full isolation, may fail with serialization errors

-- Example with locking
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
-- Row is locked until COMMIT
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;

-- Skip locked rows (useful for queues)
SELECT * FROM jobs
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

### Advisory Locks

```sql
-- Application-level locks
SELECT pg_advisory_lock(12345);  -- Blocking lock
-- Do work...
SELECT pg_advisory_unlock(12345);

-- Non-blocking (returns false if not acquired)
SELECT pg_try_advisory_lock(12345);

-- Transaction-level (auto-released on commit)
SELECT pg_advisory_xact_lock(12345);
```

---

## JSON and JSONB

### JSONB Operations

```sql
-- Create table with JSONB
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    attributes JSONB DEFAULT '{}'
);

-- Insert JSON data
INSERT INTO products (name, attributes) VALUES
    ('Laptop', '{"brand": "Dell", "specs": {"ram": 16, "storage": 512}, "tags": ["electronics", "computer"]}'),
    ('Phone', '{"brand": "Apple", "specs": {"storage": 128}, "tags": ["electronics", "mobile"]}');

-- Query JSON fields
SELECT
    name,
    attributes->>'brand' AS brand,                    -- Get as text
    attributes->'specs'->>'ram' AS ram,               -- Nested access
    (attributes->'specs'->>'storage')::int AS storage -- Cast to int
FROM products;

-- Filter by JSON field
SELECT * FROM products
WHERE attributes->>'brand' = 'Dell';

SELECT * FROM products
WHERE (attributes->'specs'->>'ram')::int >= 16;

-- Check if key exists
SELECT * FROM products
WHERE attributes ? 'brand';

-- Check if any key exists
SELECT * FROM products
WHERE attributes ?| array['brand', 'model'];

-- Check if all keys exist
SELECT * FROM products
WHERE attributes ?& array['brand', 'specs'];

-- Contains check
SELECT * FROM products
WHERE attributes @> '{"brand": "Dell"}';

-- Contained by check
SELECT * FROM products
WHERE '{"brand": "Dell", "category": "laptops"}' @> attributes;

-- Array contains
SELECT * FROM products
WHERE attributes->'tags' ? 'electronics';

-- JSON path queries (PostgreSQL 12+)
SELECT * FROM products
WHERE jsonb_path_exists(attributes, '$.specs.ram ? (@ >= 16)');

SELECT jsonb_path_query(attributes, '$.tags[*]') AS tag
FROM products;
```

### Modifying JSONB

```sql
-- Set/update field
UPDATE products
SET attributes = jsonb_set(attributes, '{color}', '"silver"')
WHERE id = 1;

-- Update nested field
UPDATE products
SET attributes = jsonb_set(attributes, '{specs, ram}', '32')
WHERE id = 1;

-- Remove field
UPDATE products
SET attributes = attributes - 'color'
WHERE id = 1;

-- Remove nested field
UPDATE products
SET attributes = attributes #- '{specs, ram}'
WHERE id = 1;

-- Concatenate/merge
UPDATE products
SET attributes = attributes || '{"warranty": "2 years"}'
WHERE id = 1;

-- Insert into array
UPDATE products
SET attributes = jsonb_set(
    attributes,
    '{tags}',
    (attributes->'tags') || '"new-tag"'
)
WHERE id = 1;
```

### JSONB Indexes

```sql
-- GIN index for containment operators (@>, ?, ?&, ?|)
CREATE INDEX idx_products_attributes ON products USING GIN(attributes);

-- GIN with jsonb_path_ops (smaller, faster for @>)
CREATE INDEX idx_products_attrs_path ON products USING GIN(attributes jsonb_path_ops);

-- Expression index for specific field
CREATE INDEX idx_products_brand ON products((attributes->>'brand'));
```

---

## Common Table Expressions

### Basic CTEs

```sql
-- Simple CTE
WITH active_users AS (
    SELECT id, email
    FROM users
    WHERE is_active = true
)
SELECT u.email, COUNT(o.id) AS order_count
FROM active_users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.email;

-- Multiple CTEs
WITH
monthly_sales AS (
    SELECT
        DATE_TRUNC('month', created_at) AS month,
        SUM(total_amount) AS revenue
    FROM orders
    GROUP BY DATE_TRUNC('month', created_at)
),
avg_sales AS (
    SELECT AVG(revenue) AS avg_revenue
    FROM monthly_sales
)
SELECT
    ms.month,
    ms.revenue,
    as.avg_revenue,
    ms.revenue - as.avg_revenue AS variance
FROM monthly_sales ms
CROSS JOIN avg_sales as;
```

### Recursive CTEs

```sql
-- Hierarchical data (org chart, categories)
WITH RECURSIVE category_tree AS (
    -- Base case: top-level categories
    SELECT id, name, parent_id, 0 AS level, name::text AS path
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    -- Recursive case
    SELECT c.id, c.name, c.parent_id, ct.level + 1, ct.path || ' > ' || c.name
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree ORDER BY path;

-- Generate series alternative
WITH RECURSIVE dates AS (
    SELECT '2024-01-01'::date AS date

    UNION ALL

    SELECT date + 1
    FROM dates
    WHERE date < '2024-01-31'
)
SELECT * FROM dates;

-- Bill of materials / graph traversal
WITH RECURSIVE parts AS (
    SELECT id, name, parent_id, 1 AS quantity
    FROM products
    WHERE id = 1  -- Root product

    UNION ALL

    SELECT p.id, p.name, p.parent_id, parts.quantity * bp.quantity
    FROM products p
    JOIN bill_of_materials bp ON p.id = bp.part_id
    JOIN parts ON bp.product_id = parts.id
)
SELECT name, SUM(quantity) AS total_quantity
FROM parts
GROUP BY name;
```

### Materialized CTEs

```sql
-- Force materialization (PostgreSQL 12+)
WITH active_users AS MATERIALIZED (
    SELECT id, email
    FROM users
    WHERE is_active = true
)
SELECT * FROM active_users WHERE email LIKE 'a%';

-- Prevent materialization (inline)
WITH active_users AS NOT MATERIALIZED (
    SELECT id, email
    FROM users
    WHERE is_active = true
)
SELECT * FROM active_users WHERE email LIKE 'a%';
```

---

## Window Functions

### Basic Window Functions

```sql
-- Row numbering
SELECT
    id,
    name,
    price,
    ROW_NUMBER() OVER (ORDER BY price DESC) AS rank,
    RANK() OVER (ORDER BY price DESC) AS rank_with_ties,
    DENSE_RANK() OVER (ORDER BY price DESC) AS dense_rank
FROM products;

-- Partition by category
SELECT
    id,
    name,
    category_id,
    price,
    ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY price DESC) AS rank_in_category
FROM products;

-- Running totals
SELECT
    id,
    created_at,
    total_amount,
    SUM(total_amount) OVER (ORDER BY created_at) AS running_total,
    AVG(total_amount) OVER (ORDER BY created_at) AS running_avg
FROM orders;

-- Compare with previous/next row
SELECT
    id,
    created_at,
    total_amount,
    LAG(total_amount) OVER (ORDER BY created_at) AS prev_amount,
    LEAD(total_amount) OVER (ORDER BY created_at) AS next_amount,
    total_amount - LAG(total_amount) OVER (ORDER BY created_at) AS change
FROM orders;

-- First and last values
SELECT
    id,
    category_id,
    price,
    FIRST_VALUE(price) OVER (PARTITION BY category_id ORDER BY price) AS min_price,
    LAST_VALUE(price) OVER (
        PARTITION BY category_id
        ORDER BY price
        RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS max_price
FROM products;

-- Percentiles
SELECT
    id,
    price,
    PERCENT_RANK() OVER (ORDER BY price) AS percentile,
    NTILE(4) OVER (ORDER BY price) AS quartile
FROM products;
```

### Window Frame Specification

```sql
-- Moving average (last 7 days)
SELECT
    date,
    revenue,
    AVG(revenue) OVER (
        ORDER BY date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_avg_7d
FROM daily_sales;

-- Cumulative sum with frame
SELECT
    id,
    amount,
    SUM(amount) OVER (
        ORDER BY created_at
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_sum
FROM transactions;

-- Range-based frame (by value, not rows)
SELECT
    date,
    revenue,
    SUM(revenue) OVER (
        ORDER BY date
        RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW
    ) AS last_7_days_revenue
FROM daily_sales;
```

---

## Full-Text Search

### Setup and Basic Search

```sql
-- Create text search column
ALTER TABLE articles ADD COLUMN search_vector tsvector;

-- Populate search vector
UPDATE articles SET search_vector =
    setweight(to_tsvector('english', COALESCE(title, '')), 'A') ||
    setweight(to_tsvector('english', COALESCE(content, '')), 'B');

-- Create GIN index
CREATE INDEX idx_articles_search ON articles USING GIN(search_vector);

-- Auto-update with trigger
CREATE OR REPLACE FUNCTION articles_search_update() RETURNS trigger AS $$
BEGIN
    NEW.search_vector :=
        setweight(to_tsvector('english', COALESCE(NEW.title, '')), 'A') ||
        setweight(to_tsvector('english', COALESCE(NEW.content, '')), 'B');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER articles_search_trigger
    BEFORE INSERT OR UPDATE ON articles
    FOR EACH ROW EXECUTE FUNCTION articles_search_update();
```

### Search Queries

```sql
-- Basic search
SELECT *
FROM articles
WHERE search_vector @@ to_tsquery('english', 'postgresql & performance');

-- With ranking
SELECT
    id,
    title,
    ts_rank(search_vector, query) AS rank,
    ts_headline('english', content, query) AS snippet
FROM articles,
     to_tsquery('english', 'postgresql & performance') query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 10;

-- Phrase search
SELECT *
FROM articles
WHERE search_vector @@ phraseto_tsquery('english', 'database optimization');

-- Fuzzy matching with prefix
SELECT *
FROM articles
WHERE search_vector @@ to_tsquery('english', 'optim:*');  -- Matches optimize, optimization, etc.

-- OR search
SELECT *
FROM articles
WHERE search_vector @@ to_tsquery('english', 'postgresql | mysql | mongodb');

-- NOT search
SELECT *
FROM articles
WHERE search_vector @@ to_tsquery('english', 'database & !deprecated');
```

### Generated Column for Search

```sql
-- PostgreSQL 12+ generated column
CREATE TABLE articles (
    id BIGSERIAL PRIMARY KEY,
    title TEXT NOT NULL,
    content TEXT,
    search_vector tsvector GENERATED ALWAYS AS (
        setweight(to_tsvector('english', COALESCE(title, '')), 'A') ||
        setweight(to_tsvector('english', COALESCE(content, '')), 'B')
    ) STORED
);

CREATE INDEX idx_articles_search ON articles USING GIN(search_vector);
```

---

## Quick Reference

### Common Functions

```sql
-- String functions
LENGTH(text), LOWER(text), UPPER(text), TRIM(text)
CONCAT(str1, str2), CONCAT_WS(sep, str1, str2)
SUBSTRING(text, start, length), LEFT(text, n), RIGHT(text, n)
REPLACE(text, from, to), REGEXP_REPLACE(text, pattern, replacement)
SPLIT_PART(text, delimiter, field)

-- Date/Time functions
NOW(), CURRENT_DATE, CURRENT_TIME, CURRENT_TIMESTAMP
DATE_TRUNC('month', timestamp), EXTRACT(YEAR FROM date)
AGE(timestamp1, timestamp2), DATE_PART('day', interval)
TO_CHAR(timestamp, 'YYYY-MM-DD'), TO_DATE(text, 'YYYY-MM-DD')

-- Aggregate functions
COUNT(*), SUM(column), AVG(column), MIN(column), MAX(column)
ARRAY_AGG(column), STRING_AGG(column, delimiter)
BOOL_AND(boolean), BOOL_OR(boolean)

-- Conditional
COALESCE(val1, val2, ...), NULLIF(val1, val2)
GREATEST(val1, val2, ...), LEAST(val1, val2, ...)
CASE WHEN condition THEN result ELSE default END
```

### Useful Queries

```sql
-- Check table size
SELECT pg_size_pretty(pg_total_relation_size('table_name'));

-- List all tables
SELECT tablename FROM pg_tables WHERE schemaname = 'public';

-- Describe table
\d+ table_name
SELECT column_name, data_type, is_nullable
FROM information_schema.columns
WHERE table_name = 'users';

-- Current connections
SELECT * FROM pg_stat_activity;

-- Kill connection
SELECT pg_terminate_backend(pid);

-- Explain query
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';
```
