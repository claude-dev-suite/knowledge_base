# SQL Fundamentals Deep Dive

## SQL Standard Evolution

| Standard | Year | Key Features |
|----------|------|--------------|
| SQL-86 | 1986 | Original standard |
| SQL-89 | 1989 | Integrity enhancements |
| SQL-92 | 1992 | JOIN syntax, CASE, subqueries |
| SQL:1999 | 1999 | CTEs, triggers, OO features |
| SQL:2003 | 2003 | XML, window functions, MERGE |
| SQL:2008 | 2008 | TRUNCATE, INSTEAD OF triggers |
| SQL:2011 | 2011 | Temporal tables |
| SQL:2016 | 2016 | JSON, row pattern matching |
| SQL:2023 | 2023 | Property Graph Queries |

## DML Deep Patterns

### Advanced SELECT

```sql
-- DISTINCT ON (PostgreSQL)
SELECT DISTINCT ON (customer_id) *
FROM orders
ORDER BY customer_id, created_at DESC;

-- FETCH FIRST (SQL:2008)
SELECT * FROM products
ORDER BY price DESC
FETCH FIRST 10 ROWS ONLY;

-- FETCH WITH TIES
SELECT * FROM products
ORDER BY price DESC
FETCH FIRST 10 ROWS WITH TIES;

-- LATERAL JOIN
SELECT u.*, recent.*
FROM users u
CROSS JOIN LATERAL (
    SELECT * FROM orders
    WHERE user_id = u.id
    ORDER BY created_at DESC
    LIMIT 5
) recent;
```

### INSERT Advanced

```sql
-- INSERT with CTE
WITH new_user AS (
    INSERT INTO users (name, email)
    VALUES ('John', 'john@example.com')
    RETURNING id
)
INSERT INTO user_settings (user_id, theme)
SELECT id, 'dark' FROM new_user;

-- INSERT with conflict handling
INSERT INTO metrics (date, value)
VALUES ('2024-01-15', 100)
ON CONFLICT (date) DO UPDATE
SET value = EXCLUDED.value + metrics.value;

-- INSERT with SELECT and conditional
INSERT INTO archive (id, data, archived_at)
SELECT id, data, NOW()
FROM records
WHERE created_at < NOW() - INTERVAL '1 year'
AND NOT EXISTS (SELECT 1 FROM archive WHERE archive.id = records.id);
```

### UPDATE with Complex Logic

```sql
-- UPDATE with window function result
WITH ranked AS (
    SELECT id, ROW_NUMBER() OVER (PARTITION BY category ORDER BY sales DESC) as rn
    FROM products
)
UPDATE products p
SET is_bestseller = (r.rn <= 3)
FROM ranked r
WHERE p.id = r.id;

-- Conditional UPDATE
UPDATE orders
SET status = CASE
    WHEN shipped_at IS NOT NULL AND delivered_at IS NULL THEN 'in_transit'
    WHEN delivered_at IS NOT NULL THEN 'delivered'
    WHEN paid_at IS NOT NULL THEN 'processing'
    ELSE 'pending'
END
WHERE status NOT IN ('cancelled', 'refunded');
```

## JOIN Algorithms

### Nested Loop Join
- For each row in outer table, scan inner table
- Best for: Small tables, indexed inner table
- O(n * m) without index

### Hash Join
- Build hash table from smaller table
- Probe with larger table
- Best for: Large tables, equality joins
- Requires memory

### Merge Join
- Sort both tables on join key
- Merge sorted streams
- Best for: Already sorted data, equality joins
- Efficient for large datasets

```sql
-- Force join type (PostgreSQL)
SET enable_hashjoin = off;
SET enable_mergejoin = off;
-- Only nested loop will be considered
```

## Subquery Types

### Scalar Subquery
```sql
SELECT name,
       (SELECT AVG(salary) FROM employees) as avg_salary
FROM employees;
```

### Table Subquery
```sql
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (ORDER BY id) as rn
    FROM users
) sub WHERE rn <= 10;
```

### Correlated Subquery
```sql
SELECT * FROM employees e1
WHERE salary > (
    SELECT AVG(salary) FROM employees e2
    WHERE e2.department_id = e1.department_id
);
```

### EXISTS Subquery
```sql
SELECT * FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.customer_id = c.id
    AND o.total > 1000
);
```

## Set Operations

```sql
-- UNION (removes duplicates)
SELECT email FROM customers
UNION
SELECT email FROM suppliers;

-- UNION ALL (keeps duplicates)
SELECT email FROM customers
UNION ALL
SELECT email FROM suppliers;

-- INTERSECT (common rows)
SELECT email FROM customers
INTERSECT
SELECT email FROM newsletter_subscribers;

-- EXCEPT (rows in first but not second)
SELECT email FROM customers
EXCEPT
SELECT email FROM unsubscribed;
```

## Normalization Forms

### 1NF (First Normal Form)
- Atomic values (no arrays/lists in cells)
- Each row unique (has primary key)

```sql
-- Violation: Multiple values in one cell
-- name: "John", phones: "123,456,789"

-- Correct: Separate table
CREATE TABLE user_phones (
    user_id INT REFERENCES users(id),
    phone VARCHAR(20),
    PRIMARY KEY (user_id, phone)
);
```

### 2NF (Second Normal Form)
- 1NF + No partial dependencies on composite key

```sql
-- Violation: product_name depends only on product_id, not full key
-- order_items(order_id, product_id, quantity, product_name)

-- Correct: Separate products table
CREATE TABLE products (product_id PRIMARY KEY, product_name);
CREATE TABLE order_items (order_id, product_id, quantity);
```

### 3NF (Third Normal Form)
- 2NF + No transitive dependencies

```sql
-- Violation: city depends on zip, not directly on user
-- users(id, name, zip, city)

-- Correct: Separate zip_codes table
CREATE TABLE zip_codes (zip PRIMARY KEY, city);
CREATE TABLE users (id, name, zip REFERENCES zip_codes);
```

### BCNF (Boyce-Codd Normal Form)
- 3NF + Every determinant is a candidate key

## Query Execution Order

```
1. FROM        - Get tables
2. JOIN        - Join tables
3. WHERE       - Filter rows
4. GROUP BY    - Group rows
5. HAVING      - Filter groups
6. SELECT      - Select columns
7. DISTINCT    - Remove duplicates
8. ORDER BY    - Sort results
9. LIMIT/OFFSET - Limit results
```

Understanding this order explains why:
- Can't use column alias in WHERE
- Can use column alias in ORDER BY
- HAVING can use aggregates but WHERE can't

## NULL Semantics

```sql
-- NULL comparisons
NULL = NULL    -- NULL (not TRUE)
NULL <> NULL   -- NULL (not TRUE)
NULL AND TRUE  -- NULL
NULL OR TRUE   -- TRUE

-- Safe NULL comparison
x IS NOT DISTINCT FROM y  -- TRUE if both NULL
COALESCE(x, y, z)         -- First non-NULL
NULLIF(x, y)              -- NULL if x = y
```
