# Prisma Migrations

## Table of Contents

1. [Migration Workflow Overview](#migration-workflow-overview)
2. [prisma migrate dev](#prisma-migrate-dev)
3. [prisma migrate deploy](#prisma-migrate-deploy)
4. [prisma migrate reset](#prisma-migrate-reset)
5. [prisma migrate resolve](#prisma-migrate-resolve)
6. [prisma migrate status](#prisma-migrate-status)
7. [prisma db push (Prototyping)](#prisma-db-push-prototyping)
8. [prisma db pull (Introspection)](#prisma-db-pull-introspection)
9. [Migration Files Structure](#migration-files-structure)
10. [Customizing Migrations (Manual Edits)](#customizing-migrations-manual-edits)
11. [Data Migrations](#data-migrations)
12. [Squashing Migrations](#squashing-migrations)
13. [Rolling Back Migrations](#rolling-back-migrations)
14. [Baseline Migrations (Existing Databases)](#baseline-migrations-existing-databases)
15. [Shadow Database](#shadow-database)
16. [Team Workflow and CI/CD](#team-workflow-and-cicd)
17. [Troubleshooting Common Issues](#troubleshooting-common-issues)
18. [Best Practices](#best-practices)

---

## Migration Workflow Overview

Prisma Migrate is the official database migration tool for Prisma. It enables you to evolve your database schema over time in a controlled, versioned manner. Migrations are generated from changes to your Prisma schema and stored as SQL files that can be version-controlled and applied across different environments.

### Core Concepts

**Schema-First Approach**: You define your data model in the Prisma schema file (`schema.prisma`), and Prisma Migrate generates the SQL migrations needed to update your database to match.

**Migration History**: Each migration is stored as a timestamped folder containing SQL files. Prisma tracks which migrations have been applied in a `_prisma_migrations` table in your database.

**Deterministic Migrations**: The same migrations applied in the same order will always produce the same database schema, ensuring consistency across environments.

### Migration States

Migrations can be in one of several states:

| State | Description |
|-------|-------------|
| **Pending** | Migration exists but has not been applied to the database |
| **Applied** | Migration has been successfully applied |
| **Failed** | Migration was attempted but failed during execution |
| **Rolled Back** | Migration was marked as rolled back after a failure |

### Development vs Production Workflow

**Development** (using `prisma migrate dev`):
1. Modify your `schema.prisma` file
2. Run `prisma migrate dev` to generate and apply migrations
3. Prisma Client is automatically regenerated
4. Shadow database is used for validation

**Production** (using `prisma migrate deploy`):
1. Migrations are generated during development
2. Migrations are committed to version control
3. `prisma migrate deploy` applies pending migrations
4. No interactive prompts or shadow database required

### Basic Command Reference

```bash
# Development: Create and apply migrations
npx prisma migrate dev --name <migration-name>

# Production: Apply pending migrations
npx prisma migrate deploy

# Reset database to initial state (dev only)
npx prisma migrate reset

# Check migration status
npx prisma migrate status

# Mark migration as applied/rolled-back
npx prisma migrate resolve --applied <migration-name>
npx prisma migrate resolve --rolled-back <migration-name>

# Generate Prisma Client (without migrations)
npx prisma generate

# Prototyping: Push schema directly (no migration files)
npx prisma db push

# Introspection: Pull schema from existing database
npx prisma db pull
```

---

## prisma migrate dev

The `prisma migrate dev` command is the primary tool for managing migrations during development. It creates new migrations, applies them to your development database, and regenerates Prisma Client.

### Basic Usage

```bash
# Create a new migration with a descriptive name
npx prisma migrate dev --name add_user_table

# Create migration without applying (review first)
npx prisma migrate dev --create-only --name add_user_table

# Skip seeding after migration
npx prisma migrate dev --name add_user_table --skip-seed

# Skip generating Prisma Client
npx prisma migrate dev --name add_user_table --skip-generate
```

### What prisma migrate dev Does

1. **Reads** the current Prisma schema from `schema.prisma`
2. **Creates** a shadow database to validate migrations
3. **Compares** the schema to the current database state
4. **Generates** SQL migration file for the differences
5. **Applies** the migration to the development database
6. **Triggers** `prisma generate` to regenerate Prisma Client
7. **Runs** seed script if configured in `package.json`

### Command Options

| Option | Description |
|--------|-------------|
| `--name <name>` | Name for the migration (required for new migrations) |
| `--create-only` | Create migration file without applying it |
| `--skip-seed` | Skip running the seed script after migration |
| `--skip-generate` | Skip regenerating Prisma Client |
| `--schema <path>` | Path to schema file (default: `prisma/schema.prisma`) |

### Interactive Prompts

When running `prisma migrate dev`, you may encounter interactive prompts:

**Schema Drift Detected**:
```
Drift detected: Your database schema is not in sync with your migration history.

The following is a summary of the differences between the expected database schema given your
migrations files, and the actual schema of the database.

[+] Added tables
  - new_table

We need to reset the database to apply the changes.
Do you want to continue? All data will be lost.
```

**Data Loss Warning**:
```
We need to reset the database to apply the changes.
Do you want to continue? All data will be lost.
› Yes
› No (I want to check the SQL first)
```

### Example Development Workflow

```bash
# Step 1: Start with initial schema
# schema.prisma
model User {
  id    Int     @id @default(autoincrement())
  email String  @unique
  name  String?
}

# Step 2: Create initial migration
npx prisma migrate dev --name init

# Step 3: Add a new field to the schema
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String?
  role      String   @default("USER")  # New field
  createdAt DateTime @default(now())   # New field
}

# Step 4: Create migration for changes
npx prisma migrate dev --name add_user_fields

# Step 5: Review generated SQL
# prisma/migrations/20240115120000_add_user_fields/migration.sql
```

### Handling Existing Data

When modifying existing columns, Prisma will warn you about potential data loss:

```bash
# Adding NOT NULL constraint to existing column with data
# Prisma will prompt:

We need to change the column `email` on the table `User` to make it required.
There are 5 records in the `User` table where `email` is NULL.

? Do you want to continue?
› Yes, I have a plan to handle this
› No, let me review
```

### Environment Variables

```bash
# Use specific database URL
DATABASE_URL="postgresql://user:pass@localhost:5432/mydb" npx prisma migrate dev --name init

# Or set in .env file
# .env
DATABASE_URL="postgresql://user:pass@localhost:5432/mydb"
```

---

## prisma migrate deploy

The `prisma migrate deploy` command applies pending migrations to your database in production or staging environments. Unlike `migrate dev`, it does not generate new migrations or use a shadow database.

### Basic Usage

```bash
# Apply all pending migrations
npx prisma migrate deploy

# Specify schema location
npx prisma migrate deploy --schema ./path/to/schema.prisma
```

### Characteristics

- **Non-interactive**: Does not prompt for user input
- **Safe**: Only applies existing migration files
- **No shadow database**: Does not require additional database connections
- **No Prisma Client generation**: Must run `prisma generate` separately if needed
- **CI/CD friendly**: Designed for automated deployment pipelines

### Command Options

| Option | Description |
|--------|-------------|
| `--schema <path>` | Path to schema file |

### Behavior

1. Reads migration history from `prisma/migrations` folder
2. Compares against `_prisma_migrations` table in database
3. Applies any pending migrations in chronological order
4. Records each applied migration in `_prisma_migrations` table
5. Exits with error code if any migration fails

### Exit Codes

| Code | Meaning |
|------|---------|
| 0 | All migrations applied successfully |
| 1 | Migration failure or error |

### Production Deployment Example

```bash
# In CI/CD pipeline
npx prisma migrate deploy

# Check status after deployment
npx prisma migrate status
```

### Docker Deployment

```dockerfile
# Dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
COPY prisma ./prisma/

RUN npm ci

COPY . .

RUN npx prisma generate

# Run migrations at container startup
CMD ["sh", "-c", "npx prisma migrate deploy && node dist/index.js"]
```

### Kubernetes Job for Migrations

```yaml
# migration-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: prisma-migrate
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: your-app:latest
          command: ["npx", "prisma", "migrate", "deploy"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: url
      restartPolicy: Never
  backoffLimit: 3
```

### Handling Failures

If `prisma migrate deploy` fails:

1. The failed migration is marked in `_prisma_migrations` with error details
2. Subsequent migrations are not attempted
3. You must resolve the issue before retrying

```bash
# Check what went wrong
npx prisma migrate status

# After fixing, mark as rolled back and retry
npx prisma migrate resolve --rolled-back 20240115120000_problematic_migration

# Or mark as applied if manually fixed
npx prisma migrate resolve --applied 20240115120000_problematic_migration
```

---

## prisma migrate reset

The `prisma migrate reset` command drops the database, recreates it, applies all migrations, and runs seed scripts. This is useful during development to start fresh.

### Basic Usage

```bash
# Reset database (interactive - asks for confirmation)
npx prisma migrate reset

# Skip confirmation prompt
npx prisma migrate reset --force

# Skip running seed script
npx prisma migrate reset --skip-seed

# Skip generating Prisma Client
npx prisma migrate reset --skip-generate
```

### What prisma migrate reset Does

1. **Drops** the database (or all tables if DROP DATABASE is not supported)
2. **Creates** a new database with the same name
3. **Applies** all migrations from the beginning
4. **Runs** seed script if configured
5. **Regenerates** Prisma Client

### Command Options

| Option | Description |
|--------|-------------|
| `--force` | Skip confirmation prompt |
| `--skip-seed` | Do not run the seed script |
| `--skip-generate` | Do not regenerate Prisma Client |
| `--schema <path>` | Path to schema file |

### Warning

**DATA LOSS**: This command deletes all data in your database. Only use in development environments.

### Use Cases

**Start fresh during development**:
```bash
# Reset when database gets into a bad state
npx prisma migrate reset --force
```

**Test migration sequence**:
```bash
# Verify all migrations apply cleanly from scratch
npx prisma migrate reset
```

**After squashing migrations**:
```bash
# Reset to apply the new squashed migration
rm -rf prisma/migrations
npx prisma migrate dev --name init
npx prisma migrate reset
```

### Database Provider Behavior

| Provider | Reset Behavior |
|----------|----------------|
| PostgreSQL | Drops and recreates database |
| MySQL | Drops and recreates database |
| SQLite | Deletes database file and recreates |
| SQL Server | Drops all tables (cannot drop database) |
| MongoDB | Not supported (use db push) |

### Seeding After Reset

```json
// package.json
{
  "prisma": {
    "seed": "ts-node prisma/seed.ts"
  }
}
```

```typescript
// prisma/seed.ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

async function main() {
  await prisma.user.createMany({
    data: [
      { email: 'alice@example.com', name: 'Alice' },
      { email: 'bob@example.com', name: 'Bob' },
    ],
  });
  console.log('Database seeded');
}

main()
  .catch((e) => {
    console.error(e);
    process.exit(1);
  })
  .finally(async () => {
    await prisma.$disconnect();
  });
```

---

## prisma migrate resolve

The `prisma migrate resolve` command is used to resolve issues with the migration history. It marks migrations as applied or rolled back without actually running them.

### Basic Usage

```bash
# Mark a migration as successfully applied
npx prisma migrate resolve --applied 20240115120000_migration_name

# Mark a migration as rolled back
npx prisma migrate resolve --rolled-back 20240115120000_migration_name
```

### Command Options

| Option | Description |
|--------|-------------|
| `--applied <migration>` | Mark migration as applied |
| `--rolled-back <migration>` | Mark migration as rolled back |
| `--schema <path>` | Path to schema file |

### When to Use

**Baselining an existing database**:
```bash
# After creating baseline migration for existing database
npx prisma migrate resolve --applied 20240101000000_baseline
```

**After manually fixing a failed migration**:
```bash
# If you manually ran the SQL to fix the database
npx prisma migrate resolve --applied 20240115120000_failed_migration
```

**After rolling back a migration manually**:
```bash
# If you manually undid the migration changes
npx prisma migrate resolve --rolled-back 20240115120000_problematic_migration
```

**Hotfix applied directly to production**:
```bash
# If DBA applied changes directly, mark migration as done
npx prisma migrate resolve --applied 20240115120000_hotfix
```

### Example Workflow: Failed Migration

```bash
# 1. Migration fails during deploy
npx prisma migrate deploy
# Error: Migration 20240115120000_add_column failed

# 2. Check the status
npx prisma migrate status
# Shows: 20240115120000_add_column - Failed

# 3. Manually fix the issue in the database
psql $DATABASE_URL -c "ALTER TABLE users ADD COLUMN status VARCHAR(50);"

# 4. Mark as applied
npx prisma migrate resolve --applied 20240115120000_add_column

# 5. Verify status
npx prisma migrate status
# Shows: All migrations applied
```

### Example Workflow: Baseline Existing Database

```bash
# 1. Pull schema from existing database
npx prisma db pull

# 2. Create baseline migration (without applying)
npx prisma migrate dev --create-only --name baseline

# 3. Mark as already applied (database already has this schema)
npx prisma migrate resolve --applied 20240101000000_baseline

# 4. Now future migrations will work normally
npx prisma migrate dev --name add_new_feature
```

---

## prisma migrate status

The `prisma migrate status` command shows the current status of migrations in your database.

### Basic Usage

```bash
# Check migration status
npx prisma migrate status

# Specify schema location
npx prisma migrate status --schema ./path/to/schema.prisma
```

### Output Examples

**All migrations applied**:
```
Prisma schema loaded from prisma/schema.prisma
Datasource "db": PostgreSQL database "mydb", schema "public" at "localhost:5432"

3 migrations found in prisma/migrations

Status    Name
  Applied 20240101000000_init
  Applied 20240102000000_add_posts
  Applied 20240103000000_add_comments

Database schema is up to date!
```

**Pending migrations**:
```
Prisma schema loaded from prisma/schema.prisma
Datasource "db": PostgreSQL database "mydb", schema "public" at "localhost:5432"

3 migrations found in prisma/migrations

Status    Name
  Applied 20240101000000_init
  Applied 20240102000000_add_posts
  Pending 20240103000000_add_comments

Following migrations have not yet been applied:
20240103000000_add_comments

To apply migrations in production, run:
  npx prisma migrate deploy
```

**Failed migration**:
```
Prisma schema loaded from prisma/schema.prisma
Datasource "db": PostgreSQL database "mydb", schema "public" at "localhost:5432"

3 migrations found in prisma/migrations

Status     Name
  Applied  20240101000000_init
  Applied  20240102000000_add_posts
  Failed   20240103000000_add_comments

The migration 20240103000000_add_comments failed.
- When: 2024-01-03 12:00:00 UTC
- Reason: ERROR: relation "users" does not exist

Run prisma migrate resolve --rolled-back "20240103000000_add_comments" to mark as rolled back.
```

**Schema drift detected**:
```
Prisma schema loaded from prisma/schema.prisma
Datasource "db": PostgreSQL database "mydb", schema "public" at "localhost:5432"

Drift detected: Your database schema is not in sync with your migration history.

The following is a summary of the differences:
[+] Added column `status` to table `users`

To fix this, you can:
- Run prisma migrate dev to create a migration
- Or manually update the database
```

### Exit Codes

| Code | Meaning |
|------|---------|
| 0 | All migrations applied, no drift |
| 1 | Pending migrations or drift detected |

### CI/CD Usage

```bash
# Fail CI if migrations are pending
npx prisma migrate status || exit 1

# Or check status and deploy if needed
if ! npx prisma migrate status; then
  npx prisma migrate deploy
fi
```

---

## prisma db push (Prototyping)

The `prisma db push` command pushes schema changes directly to the database without creating migration files. It is designed for rapid prototyping and development.

### Basic Usage

```bash
# Push schema to database
npx prisma db push

# Accept data loss (for destructive changes)
npx prisma db push --accept-data-loss

# Skip generating Prisma Client
npx prisma db push --skip-generate

# Force push even if there would be data loss
npx prisma db push --force-reset
```

### Command Options

| Option | Description |
|--------|-------------|
| `--accept-data-loss` | Allow changes that may delete data |
| `--skip-generate` | Do not regenerate Prisma Client |
| `--force-reset` | Reset database if needed to apply changes |
| `--schema <path>` | Path to schema file |

### Differences: db push vs migrate dev

| Aspect | db push | migrate dev |
|--------|---------|-------------|
| Migration files | No | Yes |
| Version control | Not tracked | Tracked |
| Shadow database | No | Yes |
| Production use | No | Yes |
| Speed | Faster | Slower |
| Rollback support | No | Yes |
| Team collaboration | Poor | Good |

### When to Use db push

**Early prototyping**:
```bash
# Rapid iteration on schema design
# Edit schema.prisma
npx prisma db push
# Repeat until schema is finalized
```

**Local development exploration**:
```bash
# Try out schema changes without commitment
npx prisma db push
# If it works, create proper migration
npx prisma migrate dev --name finalize_schema
```

**MongoDB and other non-relational databases**:
```bash
# MongoDB does not support migrations
npx prisma db push
```

### When NOT to Use db push

- Production deployments
- Team environments
- When you need migration history
- When rollback capability is required
- For relational databases in production

### Handling Warnings

```bash
# Prisma warns about potential data loss
npx prisma db push

# Output:
# Warning: These changes may result in data loss:
#   - The column `email` on the `User` table would be dropped and recreated.
#
# Use --accept-data-loss to proceed anyway.
```

### Transition from Prototyping to Migrations

```bash
# Step 1: Finish prototyping with db push
npx prisma db push

# Step 2: When ready for production
# Option A: Create baseline migration
npx prisma migrate dev --create-only --name baseline

# Option B: Start fresh (if data is not important)
npx prisma migrate reset
npx prisma migrate dev --name init
```

---

## prisma db pull (Introspection)

The `prisma db pull` command introspects an existing database and generates a Prisma schema that matches its structure.

### Basic Usage

```bash
# Pull schema from database
npx prisma db pull

# Print to stdout instead of file
npx prisma db pull --print

# Force overwrite existing schema
npx prisma db pull --force
```

### Command Options

| Option | Description |
|--------|-------------|
| `--print` | Print schema to stdout |
| `--force` | Overwrite existing models in schema |
| `--schema <path>` | Path to schema file |

### What Gets Introspected

- Tables as models
- Columns as fields
- Indexes
- Foreign keys as relations
- Enums
- Default values
- Primary keys

### Example Output

**Database tables**:
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    content TEXT,
    author_id INTEGER REFERENCES users(id)
);
```

**Introspected schema**:
```prisma
model users {
  id         Int       @id @default(autoincrement())
  email      String    @unique @db.VarChar(255)
  name       String?   @db.VarChar(255)
  created_at DateTime? @default(now())
  posts      posts[]
}

model posts {
  id        Int     @id @default(autoincrement())
  title     String  @db.VarChar(255)
  content   String?
  author_id Int?
  users     users?  @relation(fields: [author_id], references: [id])
}
```

### Post-Introspection Cleanup

After introspection, you typically need to:

1. **Rename models** to follow conventions:
```prisma
// Before
model users {
  // ...
}

// After
model User {
  // ...
  @@map("users")  // Keep table name mapping
}
```

2. **Add relation names**:
```prisma
model User {
  posts Post[] @relation("UserPosts")
}

model Post {
  author User? @relation("UserPosts", fields: [authorId], references: [id])
}
```

3. **Add missing relations**:
```prisma
model Post {
  authorId Int?   @map("author_id")
  author   User?  @relation(fields: [authorId], references: [id])
}
```

### Use Cases

**Adopting Prisma in existing project**:
```bash
# 1. Pull existing schema
npx prisma db pull

# 2. Clean up schema
# Edit schema.prisma

# 3. Create baseline migration
npx prisma migrate dev --create-only --name baseline

# 4. Mark as applied
npx prisma migrate resolve --applied <migration-name>

# 5. Generate client
npx prisma generate
```

**Keeping schema in sync with manual DB changes**:
```bash
# After DBA makes direct changes
npx prisma db pull

# Review changes
git diff prisma/schema.prisma

# Create migration for changes
npx prisma migrate dev --name sync_manual_changes
```

---

## Migration Files Structure

Prisma stores migrations in a specific folder structure that enables version control and reproducible deployments.

### Folder Structure

```
project/
├── prisma/
│   ├── schema.prisma           # Prisma schema file
│   ├── seed.ts                 # Seed script (optional)
│   └── migrations/
│       ├── migration_lock.toml # Prevents concurrent migrations
│       ├── 20240101000000_init/
│       │   └── migration.sql
│       ├── 20240102000000_add_posts/
│       │   └── migration.sql
│       └── 20240103000000_add_comments/
│           └── migration.sql
```

### Migration Naming Convention

```
<timestamp>_<name>/
└── migration.sql

# Example:
20240115143022_add_user_profile/
└── migration.sql

# Timestamp format: YYYYMMDDHHmmss (UTC)
```

### migration.sql File

```sql
-- prisma/migrations/20240101000000_init/migration.sql

-- CreateEnum
CREATE TYPE "Role" AS ENUM ('USER', 'ADMIN', 'MODERATOR');

-- CreateTable
CREATE TABLE "User" (
    "id" TEXT NOT NULL,
    "email" TEXT NOT NULL,
    "name" TEXT,
    "role" "Role" NOT NULL DEFAULT 'USER',
    "createdAt" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updatedAt" TIMESTAMP(3) NOT NULL,

    CONSTRAINT "User_pkey" PRIMARY KEY ("id")
);

-- CreateTable
CREATE TABLE "Post" (
    "id" TEXT NOT NULL,
    "title" TEXT NOT NULL,
    "content" TEXT,
    "published" BOOLEAN NOT NULL DEFAULT false,
    "authorId" TEXT NOT NULL,
    "createdAt" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT "Post_pkey" PRIMARY KEY ("id")
);

-- CreateIndex
CREATE UNIQUE INDEX "User_email_key" ON "User"("email");

-- CreateIndex
CREATE INDEX "Post_authorId_idx" ON "Post"("authorId");

-- AddForeignKey
ALTER TABLE "Post" ADD CONSTRAINT "Post_authorId_fkey"
    FOREIGN KEY ("authorId") REFERENCES "User"("id")
    ON DELETE RESTRICT ON UPDATE CASCADE;
```

### migration_lock.toml

```toml
# prisma/migrations/migration_lock.toml

# Please do not edit this file manually
# It should be added in your version-control system (e.g., Git)

provider = "postgresql"
```

This file:
- Locks the database provider
- Prevents accidental provider changes
- Should be committed to version control

### _prisma_migrations Table

Prisma creates a table in your database to track applied migrations:

```sql
CREATE TABLE _prisma_migrations (
    id                  VARCHAR(36) PRIMARY KEY,
    checksum            VARCHAR(64) NOT NULL,
    finished_at         TIMESTAMP WITH TIME ZONE,
    migration_name      VARCHAR(255) NOT NULL,
    logs                TEXT,
    rolled_back_at      TIMESTAMP WITH TIME ZONE,
    started_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
    applied_steps_count INTEGER NOT NULL DEFAULT 0
);
```

| Column | Description |
|--------|-------------|
| id | Unique identifier (UUID) |
| checksum | SHA-256 hash of migration file |
| finished_at | When migration completed |
| migration_name | Folder name (e.g., 20240101000000_init) |
| logs | Error logs if migration failed |
| rolled_back_at | When marked as rolled back |
| started_at | When migration started |
| applied_steps_count | Number of SQL statements executed |

### Querying Migration History

```sql
-- View all migrations
SELECT migration_name, finished_at, rolled_back_at
FROM _prisma_migrations
ORDER BY started_at;

-- Find failed migrations
SELECT migration_name, logs
FROM _prisma_migrations
WHERE finished_at IS NULL AND rolled_back_at IS NULL;

-- Check if specific migration was applied
SELECT EXISTS(
    SELECT 1 FROM _prisma_migrations
    WHERE migration_name = '20240101000000_init'
    AND finished_at IS NOT NULL
);
```

---

## Customizing Migrations (Manual Edits)

Prisma allows you to customize generated migrations before applying them. This is useful for adding data transformations, custom indexes, or database-specific features.

### Creating a Migration for Review

```bash
# Generate migration without applying
npx prisma migrate dev --create-only --name custom_migration

# Review and edit the file
code prisma/migrations/20240115120000_custom_migration/migration.sql

# Apply the edited migration
npx prisma migrate dev
```

### Common Customizations

**Adding custom indexes**:
```sql
-- Generated by Prisma
CREATE INDEX "Post_authorId_idx" ON "Post"("authorId");

-- Custom additions
CREATE INDEX "Post_title_search_idx" ON "Post" USING gin(to_tsvector('english', "title"));
CREATE INDEX CONCURRENTLY "Post_createdAt_idx" ON "Post"("createdAt" DESC);
```

**Adding database triggers**:
```sql
-- After table creation, add trigger
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW."updatedAt" = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER set_updated_at
    BEFORE UPDATE ON "User"
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at();
```

**Adding check constraints**:
```sql
-- Add constraint not supported in Prisma schema
ALTER TABLE "User" ADD CONSTRAINT "User_email_format"
    CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');
```

**Creating partial indexes**:
```sql
-- Partial index for active users only
CREATE INDEX "User_active_email_idx" ON "User"("email")
    WHERE "deleted" = false;
```

**Adding extensions**:
```sql
-- Enable PostgreSQL extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";
```

### Data Migration in Custom SQL

```sql
-- Step 1: Add new column as nullable
ALTER TABLE "User" ADD COLUMN "fullName" TEXT;

-- Step 2: Backfill data
UPDATE "User" SET "fullName" = CONCAT("firstName", ' ', "lastName");

-- Step 3: Make column required
ALTER TABLE "User" ALTER COLUMN "fullName" SET NOT NULL;

-- Step 4: Drop old columns (optional)
ALTER TABLE "User" DROP COLUMN "firstName";
ALTER TABLE "User" DROP COLUMN "lastName";
```

### Splitting Large Migrations

For complex changes, create multiple migrations:

```bash
# Migration 1: Add new structure
npx prisma migrate dev --create-only --name add_new_columns

# Migration 2: Backfill data (custom SQL)
npx prisma migrate dev --create-only --name backfill_data

# Migration 3: Add constraints
npx prisma migrate dev --create-only --name add_constraints

# Migration 4: Remove old structure
npx prisma migrate dev --create-only --name cleanup_old_columns
```

### Warning About Checksum

After editing a migration, Prisma stores the checksum. If you edit an already-applied migration:

```bash
# This will cause an error
npx prisma migrate deploy
# Error: Migration 20240115120000_custom_migration has been modified
```

To fix:
1. Never edit applied migrations
2. Create a new migration for additional changes
3. Or reset the database in development

---

## Data Migrations

Data migrations transform existing data as part of schema changes. Prisma requires manual handling for data migrations.

### Strategies for Data Migrations

**1. Inline in migration SQL**:
```sql
-- Add column with default
ALTER TABLE "User" ADD COLUMN "status" TEXT DEFAULT 'ACTIVE';

-- Update existing records
UPDATE "User" SET "status" = 'LEGACY' WHERE "createdAt" < '2024-01-01';
```

**2. Separate migration file**:
```bash
# Create migration for schema change
npx prisma migrate dev --create-only --name add_status_column

# Create migration for data backfill
npx prisma migrate dev --create-only --name backfill_status
```

```sql
-- 20240115120000_backfill_status/migration.sql
UPDATE "User"
SET "status" = CASE
    WHEN "role" = 'ADMIN' THEN 'ACTIVE'
    WHEN "lastLogin" < NOW() - INTERVAL '90 days' THEN 'INACTIVE'
    ELSE 'ACTIVE'
END
WHERE "status" IS NULL;
```

**3. Application-level migration**:
```typescript
// scripts/migrate-data.ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

async function migrateData() {
  const users = await prisma.user.findMany({
    where: { status: null },
  });

  for (const user of users) {
    await prisma.user.update({
      where: { id: user.id },
      data: {
        status: calculateStatus(user),
      },
    });
  }
}

function calculateStatus(user: any): string {
  // Complex logic that can't be done in SQL
  return 'ACTIVE';
}

migrateData()
  .then(() => console.log('Data migration complete'))
  .catch(console.error)
  .finally(() => prisma.$disconnect());
```

### Example: Renaming a Column

```bash
# Step 1: Create migration with --create-only
npx prisma migrate dev --create-only --name rename_user_name
```

```sql
-- 20240115120000_rename_user_name/migration.sql

-- Rename column (preserves data)
ALTER TABLE "User" RENAME COLUMN "name" TO "displayName";
```

```prisma
// Update schema.prisma
model User {
  id          String @id
  displayName String? // Renamed from "name"
}
```

### Example: Splitting a Table

```sql
-- Migration 1: Create new table
CREATE TABLE "Profile" (
    "id" TEXT NOT NULL PRIMARY KEY,
    "userId" TEXT NOT NULL UNIQUE REFERENCES "User"("id"),
    "bio" TEXT,
    "avatar" TEXT
);

-- Migration 2: Copy data
INSERT INTO "Profile" ("id", "userId", "bio", "avatar")
SELECT gen_random_uuid(), "id", "bio", "avatarUrl"
FROM "User"
WHERE "bio" IS NOT NULL OR "avatarUrl" IS NOT NULL;

-- Migration 3: Drop old columns
ALTER TABLE "User" DROP COLUMN "bio";
ALTER TABLE "User" DROP COLUMN "avatarUrl";
```

### Example: Merging Tables

```sql
-- Migration 1: Add columns to target table
ALTER TABLE "User" ADD COLUMN "addressStreet" TEXT;
ALTER TABLE "User" ADD COLUMN "addressCity" TEXT;
ALTER TABLE "User" ADD COLUMN "addressZip" TEXT;

-- Migration 2: Copy data from Address table
UPDATE "User" u
SET
    "addressStreet" = a."street",
    "addressCity" = a."city",
    "addressZip" = a."zipCode"
FROM "Address" a
WHERE a."userId" = u."id";

-- Migration 3: Drop old table
DROP TABLE "Address";
```

### Batch Processing Large Tables

```sql
-- Process in batches to avoid locking
DO $$
DECLARE
    batch_size INT := 1000;
    processed INT := 0;
BEGIN
    LOOP
        UPDATE "User"
        SET "status" = 'MIGRATED'
        WHERE "id" IN (
            SELECT "id" FROM "User"
            WHERE "status" IS NULL
            LIMIT batch_size
        );

        GET DIAGNOSTICS processed = ROW_COUNT;

        EXIT WHEN processed = 0;

        COMMIT;
        RAISE NOTICE 'Processed % rows', processed;
    END LOOP;
END $$;
```

---

## Squashing Migrations

Squashing combines multiple migrations into a single migration. This simplifies the migration history and improves performance.

### When to Squash

- Many small migrations accumulated during development
- Starting a new version/release
- Migration history has become unwieldy
- Before going to production with a feature

### Method 1: Fresh Baseline (Development)

```bash
# 1. Backup current migrations (optional)
cp -r prisma/migrations prisma/migrations_backup

# 2. Delete all migrations
rm -rf prisma/migrations

# 3. Create fresh migration
npx prisma migrate dev --name init

# 4. Verify schema matches
npx prisma migrate status
```

### Method 2: Squash for Existing Production Database

```bash
# 1. Ensure production database is current
npx prisma migrate deploy

# 2. Delete migration files locally
rm -rf prisma/migrations

# 3. Create new baseline migration
npx prisma migrate dev --create-only --name squashed_baseline

# 4. Mark existing production database as baselined
# (Run this against production)
npx prisma migrate resolve --applied 20240115120000_squashed_baseline
```

### Method 3: Keep Some History

```bash
# Keep initial migration, squash the rest
# 1. Note the initial migration
ls prisma/migrations
# 20240101000000_init
# 20240102000000_add_users
# 20240103000000_add_posts
# ... many more

# 2. Create a checkpoint schema
npx prisma db pull > prisma/checkpoint-schema.prisma

# 3. Delete migrations after init
rm -rf prisma/migrations/20240102*
rm -rf prisma/migrations/20240103*
# ... etc

# 4. Create squashed migration
npx prisma migrate dev --create-only --name squash_to_checkpoint
```

### Squashing Best Practices

1. **Never squash in production** - Only squash in development
2. **Coordinate with team** - Everyone should reset their local database
3. **Tag in version control** - Create a tag before squashing
4. **Update CI/CD** - Ensure pipelines can handle the squash
5. **Document the squash** - Note what was combined

### Post-Squash Team Coordination

```bash
# Team member workflow after squash
git pull origin main

# Reset local database to apply squashed migration
npx prisma migrate reset

# Or if you have important local data
npx prisma migrate resolve --applied <squashed-migration-name>
```

---

## Rolling Back Migrations

Prisma does not have a built-in rollback command. Rollbacks must be handled manually or through additional migrations.

### Understanding Rollback Limitations

- Prisma does not generate "down" migrations
- No automatic rollback command exists
- Manual intervention required for rollbacks

### Method 1: Create Reverse Migration

```bash
# Create a new migration that undoes the previous one
npx prisma migrate dev --create-only --name revert_add_column
```

```sql
-- 20240115130000_revert_add_column/migration.sql

-- Undo: DROP COLUMN that was added
ALTER TABLE "User" DROP COLUMN "status";
```

### Method 2: Manual Database Rollback

```bash
# 1. Identify the migration to roll back
npx prisma migrate status

# 2. Write reverse SQL manually
psql $DATABASE_URL << 'EOF'
BEGIN;
-- Reverse the migration changes
ALTER TABLE "User" DROP COLUMN "status";
DELETE FROM _prisma_migrations WHERE migration_name = '20240115120000_add_status';
COMMIT;
EOF

# 3. Verify status
npx prisma migrate status
```

### Method 3: Reset to Previous State (Development)

```bash
# 1. Remove the migration file
rm -rf prisma/migrations/20240115120000_add_status

# 2. Revert schema.prisma changes
git checkout HEAD~1 -- prisma/schema.prisma

# 3. Reset database
npx prisma migrate reset
```

### Creating Rollback Scripts

Maintain rollback scripts alongside migrations:

```
prisma/migrations/
├── 20240115120000_add_status/
│   ├── migration.sql      # Forward migration
│   └── rollback.sql       # Manual rollback (not used by Prisma)
```

```sql
-- rollback.sql (manual reference)
ALTER TABLE "User" DROP COLUMN IF EXISTS "status";
```

### Handling Failed Production Migrations

```bash
# 1. Check status
npx prisma migrate status
# Shows: 20240115120000_add_status - Failed

# 2. Option A: Fix and mark as applied
# Manually fix the database
npx prisma migrate resolve --applied 20240115120000_add_status

# 2. Option B: Roll back and mark
# Manually undo partial changes
npx prisma migrate resolve --rolled-back 20240115120000_add_status

# 3. Create a fixed migration
npx prisma migrate dev --create-only --name add_status_fixed
```

### Preventing Rollback Issues

1. **Test migrations thoroughly** in staging
2. **Keep migrations small** and focused
3. **Use transactions** where possible
4. **Back up data** before major migrations
5. **Document rollback procedures** for each migration

---

## Baseline Migrations (Existing Databases)

Baselining allows you to adopt Prisma Migrate for an existing database without recreating it.

### When to Baseline

- Adopting Prisma for an existing project
- Database was created manually or with another tool
- Moving from `prisma db push` to migrations

### Baseline Process

```bash
# Step 1: Introspect existing database
npx prisma db pull

# Step 2: Review and clean up generated schema
# Edit prisma/schema.prisma

# Step 3: Create baseline migration (without applying)
npx prisma migrate dev --create-only --name baseline

# Step 4: Mark as applied (database already has this schema)
npx prisma migrate resolve --applied 20240101000000_baseline

# Step 5: Verify status
npx prisma migrate status
# Should show: baseline - Applied

# Step 6: Now you can create new migrations normally
npx prisma migrate dev --name add_new_feature
```

### Baseline with Custom Migration

```bash
# Step 1: Create empty migration
mkdir -p prisma/migrations/0_baseline

# Step 2: Create migration.sql with your existing schema
cat > prisma/migrations/0_baseline/migration.sql << 'EOF'
-- Baseline migration
-- This migration represents the existing database schema

CREATE TABLE IF NOT EXISTS "User" (
    "id" TEXT NOT NULL PRIMARY KEY,
    "email" TEXT NOT NULL UNIQUE,
    "name" TEXT
);

-- Add other existing tables...
EOF

# Step 3: Mark as applied
npx prisma migrate resolve --applied 0_baseline
```

### Handling Multiple Environments

When baselining across environments:

```bash
# Development: Already has the schema
npx prisma migrate resolve --applied 20240101000000_baseline

# Staging: Already has the schema
DATABASE_URL=$STAGING_URL npx prisma migrate resolve --applied 20240101000000_baseline

# Production: Already has the schema
DATABASE_URL=$PRODUCTION_URL npx prisma migrate resolve --applied 20240101000000_baseline
```

### Baseline Script for CI/CD

```bash
#!/bin/bash
# scripts/baseline-database.sh

BASELINE_MIGRATION="20240101000000_baseline"

# Check if baseline is already applied
if npx prisma migrate status 2>&1 | grep -q "$BASELINE_MIGRATION.*Applied"; then
    echo "Baseline already applied"
else
    echo "Applying baseline..."
    npx prisma migrate resolve --applied "$BASELINE_MIGRATION"
fi

# Apply any pending migrations
npx prisma migrate deploy
```

### Verifying Baseline Accuracy

```bash
# Compare schema to database
npx prisma migrate diff \
  --from-schema-datamodel prisma/schema.prisma \
  --to-schema-datasource prisma/schema.prisma

# Should show no differences if baseline is accurate
```

---

## Shadow Database

The shadow database is a temporary database that Prisma uses during development to validate migrations and detect drift.

### What is the Shadow Database?

- A temporary database created by `prisma migrate dev`
- Used to validate migration history
- Detects schema drift between migrations and database
- Automatically created and destroyed

### How It Works

1. Prisma creates a temporary database
2. Applies all migrations from history to shadow database
3. Compares shadow database to development database
4. Detects any drift or inconsistencies
5. Drops the shadow database

### Configuration

**Default behavior** (PostgreSQL/MySQL): Prisma creates the shadow database automatically.

**Custom shadow database URL**:
```prisma
// schema.prisma
datasource db {
  provider          = "postgresql"
  url               = env("DATABASE_URL")
  shadowDatabaseUrl = env("SHADOW_DATABASE_URL")
}
```

```env
# .env
DATABASE_URL="postgresql://user:pass@localhost:5432/mydb"
SHADOW_DATABASE_URL="postgresql://user:pass@localhost:5432/mydb_shadow"
```

### When Shadow Database is Required

- `prisma migrate dev` - Always uses shadow database
- `prisma migrate reset` - Uses shadow database for validation

### When Shadow Database is NOT Used

- `prisma migrate deploy` - Production command, no shadow database
- `prisma db push` - Direct schema push, no migrations
- `prisma db pull` - Introspection only

### Cloud Database Considerations

Some cloud providers restrict creating new databases:

**Azure SQL**: Cannot create databases programmatically
**PlanetScale**: Branches replace traditional shadow databases
**Supabase**: May require manual shadow database creation

**Solution**: Create a dedicated shadow database:
```env
# Use a separate database instance for shadow database
SHADOW_DATABASE_URL="postgresql://user:pass@localhost:5432/shadow_db"
```

### Shadow Database Permissions

The shadow database requires these permissions:
- CREATE DATABASE (or pre-created database)
- DROP DATABASE (or TRUNCATE all tables)
- Full DDL permissions within the database

```sql
-- Grant necessary permissions (PostgreSQL)
GRANT CREATE ON DATABASE postgres TO prisma_user;
-- Or create a dedicated shadow database
CREATE DATABASE shadow_db OWNER prisma_user;
```

### Disabling Shadow Database

For environments where shadow database is not possible:

```bash
# Use db push instead of migrate dev
npx prisma db push

# Or use migrate deploy (no shadow database)
npx prisma migrate deploy
```

### Troubleshooting Shadow Database Issues

**Error: Cannot create shadow database**:
```bash
# Provide a pre-created shadow database URL
export SHADOW_DATABASE_URL="postgresql://..."
npx prisma migrate dev
```

**Error: Permission denied**:
```sql
-- Ensure user has CREATE DATABASE permission
ALTER USER prisma_user CREATEDB;
```

---

## Team Workflow and CI/CD

Effective team collaboration and automated deployments require careful migration management.

### Team Development Workflow

**Developer A creates a migration**:
```bash
# Developer A
git checkout -b feature/add-comments
# Edit schema.prisma
npx prisma migrate dev --name add_comments
git add prisma/
git commit -m "Add comments model"
git push origin feature/add-comments
```

**Developer B pulls and applies**:
```bash
# Developer B
git checkout main
git pull origin main
npx prisma migrate dev
# Migrations are applied automatically
```

### Handling Concurrent Migrations

When multiple developers create migrations:

```bash
# Developer A: 20240115100000_add_comments
# Developer B: 20240115100500_add_likes

# After merging both, order is maintained by timestamp
prisma/migrations/
├── 20240115100000_add_comments/
└── 20240115100500_add_likes/
```

**Conflict resolution**:
```bash
# If migrations conflict
git merge feature/add-likes
# CONFLICT in prisma/schema.prisma

# Resolve schema conflict
# Edit schema.prisma to include both changes

# Regenerate if needed
npx prisma migrate dev --name merge_comments_and_likes
```

### Pull Request Checklist

- [ ] Schema changes are valid (`npx prisma validate`)
- [ ] Migration generates successfully
- [ ] Migration is reversible (document rollback)
- [ ] No destructive changes without approval
- [ ] Seed data updated if needed
- [ ] Tests pass with new schema

### GitHub Actions CI/CD

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  migrate:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Generate Prisma Client
        run: npx prisma generate

      - name: Check migration status
        run: npx prisma migrate status
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}

      - name: Deploy migrations
        run: npx prisma migrate deploy
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}

      - name: Deploy application
        run: npm run deploy
```

### GitLab CI/CD

```yaml
# .gitlab-ci.yml
stages:
  - migrate
  - deploy

migrate:
  stage: migrate
  image: node:20
  script:
    - npm ci
    - npx prisma generate
    - npx prisma migrate deploy
  only:
    - main
  variables:
    DATABASE_URL: $DATABASE_URL

deploy:
  stage: deploy
  needs: [migrate]
  script:
    - npm run deploy
  only:
    - main
```

### Migration Testing in CI

```yaml
# .github/workflows/test.yml
name: Test Migrations

on:
  pull_request:
    paths:
      - 'prisma/**'

jobs:
  test-migrations:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Test migrations
        run: |
          npx prisma migrate reset --force
          npx prisma migrate status
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/test
```

### Blue-Green Deployments

```bash
# 1. Deploy migrations to new (green) database
DATABASE_URL=$GREEN_DB_URL npx prisma migrate deploy

# 2. Deploy application to green environment
# 3. Run smoke tests
# 4. Switch traffic to green
# 5. Keep blue as fallback
```

### Database Migration Strategies

**Expand-Contract Pattern** for zero-downtime:
```bash
# Phase 1: Expand (add new column)
npx prisma migrate dev --name expand_add_new_column

# Phase 2: Migrate (application uses both old and new)
# Deploy application code that writes to both columns

# Phase 3: Contract (remove old column)
npx prisma migrate dev --name contract_remove_old_column
```

---

## Troubleshooting Common Issues

### Schema Drift

**Symptom**: Database schema differs from migration history.

```bash
# Detect drift
npx prisma migrate diff \
  --from-schema-datasource prisma/schema.prisma \
  --to-schema-datamodel prisma/schema.prisma
```

**Solutions**:
```bash
# Option 1: Reset database (development only)
npx prisma migrate reset

# Option 2: Create migration for the diff
npx prisma migrate dev --name fix_drift

# Option 3: Baseline current state
npx prisma db pull
npx prisma migrate dev --create-only --name baseline_drift
npx prisma migrate resolve --applied <migration-name>
```

### Failed Migration

**Symptom**: Migration marked as failed in `_prisma_migrations`.

```bash
# Check error details
npx prisma migrate status

# View error in database
SELECT migration_name, logs FROM _prisma_migrations
WHERE finished_at IS NULL;
```

**Solutions**:
```bash
# Option 1: Fix and mark as applied
# Manually run the missing SQL
npx prisma migrate resolve --applied <migration-name>

# Option 2: Mark as rolled back and recreate
npx prisma migrate resolve --rolled-back <migration-name>
npx prisma migrate dev --name fixed_migration
```

### Migration Checksum Mismatch

**Symptom**: `Migration checksum mismatch` error.

**Cause**: Migration file was modified after being applied.

**Solutions**:
```bash
# Option 1: Restore original migration file
git checkout <commit> -- prisma/migrations/<migration>/migration.sql

# Option 2: Reset database (development)
npx prisma migrate reset

# Option 3: Update checksum in database (use with caution)
UPDATE _prisma_migrations
SET checksum = '<new-checksum>'
WHERE migration_name = '<migration-name>';
```

### Shadow Database Errors

**Symptom**: Cannot create shadow database.

```bash
# Error: P3014
# Prisma Migrate could not create the shadow database
```

**Solutions**:
```bash
# Provide explicit shadow database URL
export SHADOW_DATABASE_URL="postgresql://user:pass@host:5432/shadow"
npx prisma migrate dev

# Or use db push for prototyping
npx prisma db push
```

### Provider Mismatch

**Symptom**: `migration_lock.toml` provider doesn't match.

```bash
# Error: The migration lock file provider does not match
```

**Solutions**:
```bash
# Update provider in schema.prisma to match lock file
# Or delete lock file and regenerate
rm prisma/migrations/migration_lock.toml
npx prisma migrate dev --name reinit
```

### Cannot Drop Table with Foreign Keys

**Symptom**: Migration fails when dropping table.

```sql
-- Error: cannot drop table because other objects depend on it
```

**Solution**: Edit migration to drop foreign keys first:
```sql
-- Drop foreign key constraints first
ALTER TABLE "Post" DROP CONSTRAINT IF EXISTS "Post_authorId_fkey";

-- Then drop the table
DROP TABLE "User";
```

### Column Cannot Be Cast Automatically

**Symptom**: Type change migration fails.

```sql
-- Error: column cannot be cast automatically to type
```

**Solution**: Edit migration for explicit cast:
```sql
-- Using explicit type conversion
ALTER TABLE "User"
ALTER COLUMN "age" TYPE INTEGER
USING "age"::integer;
```

### Timeout During Large Migrations

**Symptom**: Migration times out on large tables.

**Solutions**:
```sql
-- Option 1: Set statement timeout in migration
SET statement_timeout = '3600s';
ALTER TABLE "User" ADD COLUMN "status" TEXT;
RESET statement_timeout;

-- Option 2: Process in batches
DO $$
DECLARE
    batch_size INT := 10000;
BEGIN
    WHILE EXISTS (SELECT 1 FROM "User" WHERE "status" IS NULL LIMIT 1) LOOP
        UPDATE "User"
        SET "status" = 'ACTIVE'
        WHERE "id" IN (
            SELECT "id" FROM "User"
            WHERE "status" IS NULL
            LIMIT batch_size
        );
        COMMIT;
    END LOOP;
END $$;
```

### Database Connection Issues

**Symptom**: Cannot connect to database during migration.

```bash
# Test connection
npx prisma db execute --stdin <<< "SELECT 1"

# Check environment variable
echo $DATABASE_URL

# Validate schema
npx prisma validate
```

### Prisma Client Out of Sync

**Symptom**: Types don't match after migration.

```bash
# Regenerate Prisma Client
npx prisma generate

# Clear node_modules cache if needed
rm -rf node_modules/.prisma
npm install
npx prisma generate
```

---

## Best Practices

### Migration Naming Conventions

```bash
# Use descriptive, action-oriented names
npx prisma migrate dev --name add_user_profile_table
npx prisma migrate dev --name add_email_index_to_users
npx prisma migrate dev --name remove_deprecated_status_column
npx prisma migrate dev --name rename_username_to_display_name

# Avoid vague names
# Bad: update, fix, changes
# Good: add_created_at_to_posts, fix_user_email_constraint
```

### Keep Migrations Small

```bash
# Break large changes into smaller migrations

# Instead of one large migration:
# add_user_system (creates 10 tables with all relations)

# Create multiple focused migrations:
npx prisma migrate dev --name create_users_table
npx prisma migrate dev --name create_profiles_table
npx prisma migrate dev --name add_user_profile_relation
npx prisma migrate dev --name add_user_indexes
```

### Review Generated SQL

```bash
# Always review before applying
npx prisma migrate dev --create-only --name new_migration

# Review the generated SQL
cat prisma/migrations/*/migration.sql

# Apply after review
npx prisma migrate dev
```

### Version Control Practices

```gitignore
# .gitignore

# Include migrations in version control
# DO NOT add prisma/migrations to .gitignore

# Ignore generated files
node_modules/
.prisma/
```

```bash
# Commit migrations with meaningful messages
git add prisma/
git commit -m "feat(db): add user roles and permissions

- Add Role enum (USER, ADMIN, MODERATOR)
- Add role column to User table
- Add Permission table with many-to-many relation"
```

### Environment-Specific Configurations

```env
# .env.development
DATABASE_URL="postgresql://dev:dev@localhost:5432/myapp_dev"
SHADOW_DATABASE_URL="postgresql://dev:dev@localhost:5432/myapp_shadow"

# .env.production
DATABASE_URL="${DATABASE_URL}"
# No shadow database in production
```

### Safe Migration Patterns

**Adding a required column to existing table**:
```sql
-- Step 1: Add as nullable
ALTER TABLE "User" ADD COLUMN "status" TEXT;

-- Step 2: Backfill data
UPDATE "User" SET "status" = 'ACTIVE' WHERE "status" IS NULL;

-- Step 3: Add NOT NULL constraint
ALTER TABLE "User" ALTER COLUMN "status" SET NOT NULL;
```

**Adding an index on large table**:
```sql
-- Use CONCURRENTLY to avoid locking (PostgreSQL)
CREATE INDEX CONCURRENTLY "User_email_idx" ON "User"("email");
```

**Renaming with zero downtime**:
```sql
-- Step 1: Add new column
ALTER TABLE "User" ADD COLUMN "displayName" TEXT;

-- Step 2: Copy data
UPDATE "User" SET "displayName" = "name";

-- Step 3: Application uses both columns (deploy)

-- Step 4: Drop old column (later migration)
ALTER TABLE "User" DROP COLUMN "name";
```

### Testing Migrations

```typescript
// tests/migrations.test.ts
import { PrismaClient } from '@prisma/client';
import { execSync } from 'child_process';

describe('Migrations', () => {
  let prisma: PrismaClient;

  beforeAll(async () => {
    // Apply migrations to test database
    execSync('npx prisma migrate deploy', {
      env: {
        ...process.env,
        DATABASE_URL: process.env.TEST_DATABASE_URL,
      },
    });

    prisma = new PrismaClient();
  });

  afterAll(async () => {
    await prisma.$disconnect();
  });

  it('should have applied all migrations', async () => {
    const migrations = await prisma.$queryRaw`
      SELECT migration_name, finished_at
      FROM _prisma_migrations
      WHERE finished_at IS NOT NULL
    `;

    expect(migrations).toBeDefined();
  });

  it('should have correct schema', async () => {
    // Test that expected tables exist
    const users = await prisma.user.findMany({ take: 1 });
    expect(users).toBeDefined();
  });
});
```

### Documentation

Document complex migrations:
```sql
-- migration.sql

-- =============================================================================
-- Migration: add_audit_logging
-- Description: Adds audit logging infrastructure for compliance requirements
-- Author: developer@example.com
-- Date: 2024-01-15
-- Jira: PROJ-1234
--
-- Dependencies: None
-- Rollback: See prisma/migrations/20240115120000_add_audit_logging/rollback.sql
-- =============================================================================

-- Create audit log table for tracking all data changes
CREATE TABLE "AuditLog" (
    "id" TEXT NOT NULL PRIMARY KEY,
    "tableName" TEXT NOT NULL,
    "recordId" TEXT NOT NULL,
    "action" TEXT NOT NULL,
    "oldValues" JSONB,
    "newValues" JSONB,
    "userId" TEXT,
    "timestamp" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Index for querying by table and record
CREATE INDEX "AuditLog_tableName_recordId_idx" ON "AuditLog"("tableName", "recordId");

-- Index for querying by user
CREATE INDEX "AuditLog_userId_idx" ON "AuditLog"("userId");
```

### Monitoring and Alerts

```sql
-- Query for migration monitoring
SELECT
    migration_name,
    started_at,
    finished_at,
    EXTRACT(EPOCH FROM (finished_at - started_at)) AS duration_seconds,
    CASE WHEN finished_at IS NULL THEN 'RUNNING/FAILED' ELSE 'COMPLETED' END AS status
FROM _prisma_migrations
ORDER BY started_at DESC
LIMIT 10;
```

### Pre-Deployment Checklist

- [ ] Run `npx prisma validate` to check schema validity
- [ ] Run `npx prisma migrate dev --create-only` to preview SQL
- [ ] Review generated SQL for potential issues
- [ ] Test migrations on staging environment
- [ ] Verify rollback procedure exists
- [ ] Check for data loss risks
- [ ] Estimate migration duration for large tables
- [ ] Plan maintenance window if needed
- [ ] Notify team of pending database changes
- [ ] Backup production database before deployment

---

## Quick Reference

### Common Commands

```bash
# Development
npx prisma migrate dev --name <name>      # Create and apply migration
npx prisma migrate dev --create-only      # Create without applying
npx prisma migrate reset                  # Reset database
npx prisma db push                        # Push schema (no migration)
npx prisma db pull                        # Pull schema from database

# Production
npx prisma migrate deploy                 # Apply pending migrations
npx prisma migrate status                 # Check migration status
npx prisma migrate resolve --applied      # Mark as applied
npx prisma migrate resolve --rolled-back  # Mark as rolled back

# Utilities
npx prisma generate                       # Generate Prisma Client
npx prisma validate                       # Validate schema
npx prisma studio                         # Open database GUI
npx prisma db seed                        # Run seed script
```

### Environment Variables

```env
DATABASE_URL="postgresql://user:pass@host:5432/db"
SHADOW_DATABASE_URL="postgresql://user:pass@host:5432/shadow"
```

### Schema Configuration

```prisma
datasource db {
  provider          = "postgresql"  // postgresql, mysql, sqlite, sqlserver, mongodb
  url               = env("DATABASE_URL")
  shadowDatabaseUrl = env("SHADOW_DATABASE_URL")  // Optional
}

generator client {
  provider = "prisma-client-js"
}
```

### Migration States

| State | Description | Action |
|-------|-------------|--------|
| Applied | Successfully completed | None needed |
| Pending | Not yet applied | Run `migrate deploy` |
| Failed | Error during execution | Resolve manually |
| Rolled Back | Marked as undone | Create new migration |

This documentation covers the comprehensive workflow for managing database migrations with Prisma, from development through production deployment, including troubleshooting and best practices.
