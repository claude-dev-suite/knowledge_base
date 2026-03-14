# Database Migrations Deep Dive

## Migration Philosophy

### Evolutionary Database Design
- Treat database schema as code
- Version control all changes
- Make small, incremental changes
- Test migrations before production
- Automate deployment

### Key Principles
1. **Backward Compatibility**: New schema works with old code
2. **Forward Compatibility**: Old schema works with new code
3. **Idempotency**: Migration can run multiple times safely
4. **Reversibility**: Can undo changes if needed

## Migration Tools Comparison

### Flyway

```properties
# flyway.conf
flyway.url=jdbc:postgresql://localhost:5432/mydb
flyway.user=myuser
flyway.password=secret
flyway.locations=filesystem:./migrations
flyway.baselineOnMigrate=true
```

```bash
flyway migrate
flyway info
flyway validate
flyway repair
flyway baseline
flyway clean  # DANGEROUS: drops all objects
```

### Liquibase

```xml
<!-- changelog-master.xml -->
<databaseChangeLog>
    <include file="changelog-1.0.xml"/>
    <include file="changelog-2.0.xml"/>
</databaseChangeLog>
```

```bash
liquibase update
liquibase status
liquibase rollback-count 1
liquibase diff
liquibase generate-changelog
```

### Alembic (Python/SQLAlchemy)

```python
# alembic/versions/001_create_users.py
def upgrade():
    op.create_table('users',
        sa.Column('id', sa.Integer(), primary_key=True),
        sa.Column('email', sa.String(255), nullable=False)
    )

def downgrade():
    op.drop_table('users')
```

```bash
alembic upgrade head
alembic downgrade -1
alembic history
alembic current
alembic revision -m "description"
```

### Prisma Migrate

```bash
npx prisma migrate dev --name init
npx prisma migrate deploy
npx prisma migrate reset
npx prisma migrate status
```

### Rails Migrations

```ruby
class CreateUsers < ActiveRecord::Migration[7.0]
  def change
    create_table :users do |t|
      t.string :email, null: false
      t.timestamps
    end
    add_index :users, :email, unique: true
  end
end
```

```bash
rails db:migrate
rails db:rollback
rails db:migrate:status
```

## Zero-Downtime Patterns

### Adding a NOT NULL Column

```sql
-- Step 1: Add nullable column
ALTER TABLE users ADD COLUMN status VARCHAR(20);

-- Step 2: Backfill with default
UPDATE users SET status = 'active' WHERE status IS NULL;

-- Step 3: Add NOT NULL constraint
ALTER TABLE users ALTER COLUMN status SET NOT NULL;

-- Step 4: Add default for future inserts
ALTER TABLE users ALTER COLUMN status SET DEFAULT 'active';
```

### Renaming a Column (Safe)

```sql
-- Step 1: Add new column
ALTER TABLE users ADD COLUMN email_address VARCHAR(255);

-- Step 2: Create trigger to sync
CREATE OR REPLACE FUNCTION sync_email() RETURNS TRIGGER AS $$
BEGIN
    NEW.email_address := COALESCE(NEW.email_address, NEW.email);
    NEW.email := COALESCE(NEW.email, NEW.email_address);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tr_sync_email BEFORE INSERT OR UPDATE ON users
FOR EACH ROW EXECUTE FUNCTION sync_email();

-- Step 3: Backfill
UPDATE users SET email_address = email WHERE email_address IS NULL;

-- Step 4: Update application to use new column

-- Step 5: Drop old column and trigger
DROP TRIGGER tr_sync_email ON users;
DROP FUNCTION sync_email();
ALTER TABLE users DROP COLUMN email;
```

### Changing Column Type

```sql
-- Example: VARCHAR(100) → VARCHAR(255)
-- Safe: Just alter
ALTER TABLE users ALTER COLUMN name TYPE VARCHAR(255);

-- Example: VARCHAR → INTEGER (with data conversion)
-- Step 1: Add new column
ALTER TABLE products ADD COLUMN price_cents INTEGER;

-- Step 2: Backfill
UPDATE products SET price_cents = (price * 100)::INTEGER;

-- Step 3: Update application

-- Step 4: Drop old column
ALTER TABLE products DROP COLUMN price;
ALTER TABLE products RENAME COLUMN price_cents TO price;
```

### Adding Foreign Key

```sql
-- Step 1: Add column (nullable)
ALTER TABLE orders ADD COLUMN customer_id INTEGER;

-- Step 2: Backfill
UPDATE orders SET customer_id = (SELECT id FROM customers WHERE...) WHERE customer_id IS NULL;

-- Step 3: Add constraint (VALIDATE separately for performance)
ALTER TABLE orders ADD CONSTRAINT fk_customer
    FOREIGN KEY (customer_id) REFERENCES customers(id)
    NOT VALID;

-- Step 4: Validate (can take time but doesn't lock)
ALTER TABLE orders VALIDATE CONSTRAINT fk_customer;
```

## Lock-Free Index Creation

### PostgreSQL

```sql
-- CONCURRENTLY doesn't lock table
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);

-- If it fails partway through, clean up
DROP INDEX CONCURRENTLY IF EXISTS idx_users_email;
-- Then retry
```

### MySQL

```sql
-- ALGORITHM=INPLACE avoids table copy
-- LOCK=NONE allows concurrent DML
ALTER TABLE users ADD INDEX idx_email (email), ALGORITHM=INPLACE, LOCK=NONE;
```

### SQL Server

```sql
-- ONLINE allows concurrent access (Enterprise only)
CREATE INDEX idx_users_email ON users(email) WITH (ONLINE = ON);
```

## Large Table Migrations

### Batch Processing

```sql
-- PostgreSQL: Using advisory lock
CREATE OR REPLACE FUNCTION migrate_batch() RETURNS INTEGER AS $$
DECLARE
    batch_size INTEGER := 10000;
    rows_migrated INTEGER := 0;
BEGIN
    -- Acquire advisory lock
    IF NOT pg_try_advisory_lock(hashtext('migration_lock')) THEN
        RAISE NOTICE 'Migration already running';
        RETURN 0;
    END IF;

    -- Process batch
    WITH batch AS (
        UPDATE users
        SET new_column = calculate_value(old_column)
        WHERE id IN (
            SELECT id FROM users
            WHERE new_column IS NULL
            LIMIT batch_size
            FOR UPDATE SKIP LOCKED
        )
        RETURNING 1
    )
    SELECT COUNT(*) INTO rows_migrated FROM batch;

    -- Release lock
    PERFORM pg_advisory_unlock(hashtext('migration_lock'));

    RETURN rows_migrated;
END;
$$ LANGUAGE plpgsql;

-- Call repeatedly until done
SELECT migrate_batch();  -- Run until returns 0
```

### Ghost Tables (pt-online-schema-change style)

```sql
-- 1. Create new table with desired schema
CREATE TABLE users_new (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    new_column VARCHAR(100) DEFAULT 'value'
);

-- 2. Create triggers to keep in sync
CREATE OR REPLACE FUNCTION sync_to_new() RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO users_new VALUES (NEW.id, NEW.email, 'default');
    ELSIF TG_OP = 'UPDATE' THEN
        UPDATE users_new SET email = NEW.email WHERE id = NEW.id;
    ELSIF TG_OP = 'DELETE' THEN
        DELETE FROM users_new WHERE id = OLD.id;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tr_sync AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION sync_to_new();

-- 3. Copy existing data (in batches)
INSERT INTO users_new (id, email)
SELECT id, email FROM users
ON CONFLICT (id) DO NOTHING;

-- 4. Swap tables atomically
BEGIN;
ALTER TABLE users RENAME TO users_old;
ALTER TABLE users_new RENAME TO users;
DROP TRIGGER tr_sync ON users_old;
COMMIT;

-- 5. Clean up
DROP TABLE users_old;
```

## Testing Migrations

### Pre-Production Testing

```bash
#!/bin/bash
# test_migration.sh

# 1. Clone production database
pg_dump -Fc production_db > /tmp/prod_backup.dump
pg_restore -d test_db /tmp/prod_backup.dump

# 2. Run migration
flyway -url=jdbc:postgresql://localhost/test_db migrate

# 3. Run application tests
pytest tests/

# 4. Run data integrity checks
psql test_db -f verify_migration.sql

# 5. Measure performance
pgbench -c 10 -j 2 -t 1000 test_db
```

### Verify Script

```sql
-- verify_migration.sql

-- Check no orphaned records
SELECT COUNT(*) AS orphaned_orders
FROM orders o
WHERE NOT EXISTS (SELECT 1 FROM customers c WHERE c.id = o.customer_id);

-- Check data integrity
SELECT COUNT(*) AS invalid_emails
FROM users
WHERE email NOT LIKE '%@%';

-- Check constraints
SELECT COUNT(*) AS null_required_fields
FROM users
WHERE email IS NULL OR name IS NULL;

-- Check indexes exist
SELECT indexname FROM pg_indexes WHERE tablename = 'users';
```

## Rollback Strategies

### Code-First Rollback

Instead of database rollback, deploy code that works with old schema:

```
1. Issue detected after migration
2. Deploy old code (works with new schema due to backward compatibility)
3. Fix issue in new code
4. Deploy fixed code
```

### Database Rollback

```sql
-- Only if data not yet affected
-- Run undo migration
-- U20240115_add_column.sql
ALTER TABLE users DROP COLUMN IF EXISTS new_column;
```

### Point-in-Time Recovery

```bash
# PostgreSQL
pg_restore -d mydb \
    --target-time="2024-01-15 10:00:00" \
    backup.dump

# Or use pg_rewind for faster recovery
```
