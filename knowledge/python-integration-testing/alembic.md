# Python Integration Testing — Alembic Migration Testing

Comprehensive guide to testing Alembic database migrations using pytest-alembic, testcontainers, and custom migration test patterns.

**Official Documentation:**
- pytest-alembic: https://pytest-alembic.readthedocs.io/
- Alembic: https://alembic.sqlalchemy.org/en/latest/
- testcontainers-python: https://testcontainers-python.readthedocs.io/

---

## Table of Contents

1. [pytest-alembic Installation and Setup](#pytest-alembic-installation-and-setup)
2. [The alembic_runner Fixture](#the-alembic_runner-fixture)
3. [Built-in Test Suite](#built-in-test-suite)
4. [Per-Function Migration Testing](#per-function-migration-testing)
5. [Testing Specific Migrations](#testing-specific-migrations)
6. [Integration with Testcontainers](#integration-with-testcontainers)
7. [Migration State Inspection](#migration-state-inspection)
8. [Common Anti-Patterns](#common-anti-patterns)

---

## pytest-alembic Installation and Setup

```bash
pip install pytest-alembic
```

`pytest-alembic` is a pytest plugin that provides fixtures and built-in tests for verifying that your Alembic migration history is sane and that your ORM model definitions match the actual database schema.

### Project Structure

```
myproject/
├── alembic/
│   ├── env.py
│   ├── script.py.mako
│   └── versions/
│       ├── 001_create_users.py
│       ├── 002_add_email_index.py
│       └── 003_add_orders_table.py
├── alembic.ini
├── myapp/
│   ├── database.py
│   └── models.py
└── tests/
    ├── conftest.py
    └── test_migrations.py
```

### alembic.ini

```ini
[alembic]
script_location = alembic
# URL will be overridden programmatically in tests
sqlalchemy.url = postgresql://user:password@localhost/mydb

[loggers]
keys = root,sqlalchemy,alembic

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = WARN
handlers = console
qualname =

[logger_sqlalchemy]
level = WARN
handlers =
qualname = sqlalchemy.engine

[logger_alembic]
level = INFO
handlers =
qualname = alembic

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(levelname)-5.5s [%(name)s] %(message)s
datefmt = %H:%M:%S
```

### conftest.py for pytest-alembic

The `alembic_config` fixture is the primary integration point. Override it to point pytest-alembic at your test database:

```python
# tests/conftest.py
import pytest
from alembic.config import Config
from sqlalchemy import create_engine
from sqlalchemy.pool import NullPool
from testcontainers.postgres import PostgresContainer


@pytest.fixture(scope="session")
def postgres_container():
    with PostgresContainer("postgres:16-alpine") as pg:
        yield pg


@pytest.fixture(scope="session")
def alembic_engine(postgres_container):
    """
    The engine fixture that pytest-alembic uses to run migrations.
    Must be named 'alembic_engine' OR passed to 'alembic_config'.
    """
    engine = create_engine(
        postgres_container.get_connection_url(),
        poolclass=NullPool,
    )
    yield engine
    engine.dispose()


@pytest.fixture
def alembic_config(alembic_engine):
    """
    Provide the Alembic Config object pointing at the test database.
    pytest-alembic looks for this fixture by name.
    """
    cfg = Config("alembic.ini")
    cfg.set_main_option("sqlalchemy.url", str(alembic_engine.url))
    return cfg
```

---

## The alembic_runner Fixture

The `alembic_runner` fixture is provided by pytest-alembic. It wraps the Alembic environment and exposes methods for running migrations up and down programmatically.

```python
# alembic_runner methods:

alembic_runner.migrate_up_to(revision: str)       # Upgrade to specific revision
alembic_runner.migrate_up_one()                    # Upgrade by one revision
alembic_runner.migrate_down_to(revision: str)      # Downgrade to specific revision
alembic_runner.migrate_down_one()                  # Downgrade by one revision
alembic_runner.current()                           # Return current revision ID(s)
alembic_runner.heads()                             # Return all head revisions
alembic_runner.history()                           # Return list of all revisions
```

### Basic Usage

```python
# tests/test_migrations.py
import pytest


def test_alembic_runner_available(alembic_runner):
    """Verify pytest-alembic fixture is properly configured."""
    heads = alembic_runner.heads()
    assert len(heads) >= 1, "At least one head revision must exist"


def test_can_upgrade_to_head(alembic_runner):
    """Manually invoke upgrade — equivalent to the built-in test."""
    alembic_runner.migrate_up_to("head")
    current = alembic_runner.current()
    assert current is not None


def test_can_downgrade_to_base(alembic_runner):
    """Verify all migrations can be rolled back to base."""
    alembic_runner.migrate_up_to("head")
    alembic_runner.migrate_down_to("base")
    # No assertion needed — success means no OperationalError
```

---

## Built-in Test Suite

pytest-alembic ships four built-in tests that are automatically collected when the plugin is active and `alembic_config`/`alembic_engine` fixtures are available. They can be run by naming the test module `test_migrations.py` and importing the built-in tests:

```python
# tests/test_migrations.py
# Import all built-in tests to make them available
from pytest_alembic.tests import (
    test_single_head_revision,
    test_upgrade,
    test_model_definitions_match_ddl,
    test_up_down_consistency,
)
```

Or run them without imports — pytest-alembic auto-collects them if the plugin is active:

```bash
# Run all built-in migration tests
pytest --test-alembic tests/

# Run only specific built-in tests
pytest --test-alembic -k "test_upgrade"
```

### test_single_head_revision

Ensures there is exactly one head revision. Multiple heads indicate a branched migration history, which causes `alembic upgrade head` to fail or produce inconsistent behavior.

```python
def test_single_head_revision(alembic_runner):
    """
    Built-in test: Assert there is exactly one head revision.

    Fails when:
    - Two developers create migrations from the same base independently
    - A merge migration was not created after resolving the branch

    Fix:
        alembic merge heads -m "merge_001_and_002"
    """
    # This is the built-in implementation equivalent:
    heads = alembic_runner.heads()
    assert len(heads) == 1, (
        f"Expected single head revision, found: {heads}. "
        "Run: alembic merge heads -m 'merge branch'"
    )
```

**When it fails:**
```bash
# Two developers branched from the same revision
# Revision A: abc123 → def456 (developer 1 adds users table)
# Revision B: abc123 → ghi789 (developer 2 adds products table)
# alembic heads → [def456, ghi789]  ← two heads!
# Fix:
alembic merge def456 ghi789 -m "merge_users_and_products"
```

### test_upgrade

Verifies that `alembic upgrade head` completes without error. This is the most fundamental migration test — if it fails, the migration history is broken.

```python
def test_upgrade(alembic_runner):
    """
    Built-in test: Upgrade the database to the latest migration.

    Catches:
    - SQL errors in migration scripts (column type mismatch, constraint violations)
    - Missing imports in env.py
    - Incomplete upgrade() functions in version files
    """
    alembic_runner.migrate_up_to("head")
    # Success = no exception raised
```

### test_model_definitions_match_ddl

Compares the actual database schema after migrations against the SQLAlchemy model definitions. Catches the common bug of adding a column to a model but forgetting to create a migration for it.

```python
def test_model_definitions_match_ddl(alembic_runner):
    """
    Built-in test: Verify ORM models match the migrated database schema.

    Internally uses alembic.runtime.migration.MigrationContext and
    sqlalchemy.engine.reflection.Inspector to compare:
    - Table names
    - Column names, types, nullability, defaults
    - Index definitions
    - Foreign key constraints
    - Unique constraints

    Fails when:
    - Model has a column that no migration adds to the DB
    - Migration adds a column but the model doesn't define it
    - Column type in model differs from migration (e.g., String vs Text)
    - Migration uses server_default but model has no default
    """
    alembic_runner.migrate_up_to("head")
    # The built-in test then runs alembic's compare_metadata internally
```

**Common failure scenarios:**

```python
# Scenario 1: Column added to model but no migration created
class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255))
    phone: Mapped[Optional[str]] = mapped_column(String(20))  # ← Added but no migration!

# Fix: alembic revision --autogenerate -m "add_phone_to_users"


# Scenario 2: Column type mismatch
# Migration:
op.add_column("users", sa.Column("bio", sa.TEXT()))
# Model:
bio: Mapped[str] = mapped_column(String(1000))  # ← String != TEXT

# Fix: Match types between model and migration
bio: Mapped[str] = mapped_column(Text)  # ← Use Text, not String(n)
```

### test_up_down_consistency

Verifies that for every migration, upgrading and then downgrading produces a clean state — no orphaned tables, columns, or constraints.

```python
def test_up_down_consistency(alembic_runner):
    """
    Built-in test: Verify upgrade→downgrade is idempotent for every revision.

    Algorithm:
      For each revision in history:
        1. Upgrade to revision N
        2. Downgrade from revision N to N-1
        3. Upgrade to revision N again
      Checks that no SQL errors occur at any step.

    Catches:
    - Missing downgrade() function in a version file
    - downgrade() that doesn't undo everything upgrade() does
    - Dropping objects that were already dropped by a later migration
    """
    pass  # pytest-alembic handles the full loop internally
```

**Fixing a broken downgrade:**
```python
# migrations/versions/003_add_orders_table.py

def upgrade():
    op.create_table(
        "orders",
        sa.Column("id", sa.Integer(), primary_key=True),
        sa.Column("user_id", sa.Integer(), sa.ForeignKey("users.id"), nullable=False),
        sa.Column("total", sa.Numeric(10, 2)),
    )
    op.create_index("ix_orders_user_id", "orders", ["user_id"])


def downgrade():
    # WRONG: Missing index drop — will fail if index exists
    # op.drop_table("orders")

    # CORRECT: Drop index before dropping table
    op.drop_index("ix_orders_user_id", table_name="orders")
    op.drop_table("orders")
```

---

## Per-Function Migration Testing

Sometimes you want each test to start with a freshly migrated database and roll back to base after. This is expensive but guarantees complete isolation of migration state.

```python
# tests/test_migrations.py
import pytest
from alembic import command


@pytest.fixture
def migrated_db(alembic_runner):
    """
    Apply all migrations before the test, roll back to base after.
    Useful for testing code that depends on a specific schema state.
    """
    alembic_runner.migrate_up_to("head")
    yield alembic_runner
    alembic_runner.migrate_down_to("base")


@pytest.fixture
def db_at_revision(alembic_runner):
    """Fixture factory for migrating to a specific revision."""
    def _migrate(revision: str):
        alembic_runner.migrate_up_to(revision)
        return alembic_runner

    return _migrate


def test_users_table_exists_after_migration(migrated_db, alembic_engine):
    """Verify the users table exists after running all migrations."""
    from sqlalchemy import inspect
    inspector = inspect(alembic_engine)
    tables = inspector.get_table_names()
    assert "users" in tables


def test_schema_at_head(migrated_db, alembic_engine):
    """Verify full schema after migrations."""
    from sqlalchemy import inspect, text
    inspector = inspect(alembic_engine)

    # Check tables
    assert "users" in inspector.get_table_names()
    assert "orders" in inspector.get_table_names()
    assert "products" in inspector.get_table_names()

    # Check columns on users table
    user_columns = {c["name"] for c in inspector.get_columns("users")}
    assert "id" in user_columns
    assert "email" in user_columns
    assert "created_at" in user_columns

    # Check index exists
    indexes = {idx["name"] for idx in inspector.get_indexes("users")}
    assert "ix_users_email" in indexes
```

---

## Testing Specific Migrations

### Migrating Up/Down One Step at a Time

```python
# tests/test_specific_migrations.py
import pytest
from sqlalchemy import inspect, text


def test_migration_002_adds_email_index(alembic_runner, alembic_engine):
    """
    Test that migration 002 adds an index on users.email.
    """
    # Start from beginning
    alembic_runner.migrate_down_to("base")

    # Apply only up to migration 001 (before the index migration)
    alembic_runner.migrate_up_to("001_create_users")

    inspector = inspect(alembic_engine)
    indexes_before = {idx["name"] for idx in inspector.get_indexes("users")}
    assert "ix_users_email" not in indexes_before

    # Apply migration 002
    alembic_runner.migrate_up_one()

    inspector = inspect(alembic_engine)
    indexes_after = {idx["name"] for idx in inspector.get_indexes("users")}
    assert "ix_users_email" in indexes_after


def test_migration_002_downgrade_removes_index(alembic_runner, alembic_engine):
    """Test that downgrading migration 002 removes the email index."""
    alembic_runner.migrate_up_to("002_add_email_index")

    # Verify index exists
    inspector = inspect(alembic_engine)
    indexes = {idx["name"] for idx in inspector.get_indexes("users")}
    assert "ix_users_email" in indexes

    # Downgrade
    alembic_runner.migrate_down_one()

    inspector = inspect(alembic_engine)
    indexes = {idx["name"] for idx in inspector.get_indexes("users")}
    assert "ix_users_email" not in indexes
```

### Testing Data Migrations

Data migrations are the most critical to test — they transform existing data and can cause irreversible data loss if wrong.

```python
# migrations/versions/005_backfill_user_status.py
"""
Data migration: backfill users.status from users.is_active.
- is_active=True  → status='active'
- is_active=False → status='inactive'
"""
from alembic import op
import sqlalchemy as sa


def upgrade():
    # Step 1: Add the new column (nullable first for backfill)
    op.add_column("users", sa.Column("status", sa.String(20), nullable=True))

    # Step 2: Backfill from existing data
    op.execute("""
        UPDATE users
        SET status = CASE
            WHEN is_active = TRUE THEN 'active'
            ELSE 'inactive'
        END
    """)

    # Step 3: Make non-nullable after backfill
    op.alter_column("users", "status", nullable=False)


def downgrade():
    op.drop_column("users", "status")
```

```python
# tests/test_data_migrations.py
import pytest
from sqlalchemy import text


@pytest.fixture
def db_before_status_migration(alembic_runner, alembic_engine):
    """Set up DB at state before migration 005, with test data."""
    alembic_runner.migrate_up_to("004_add_is_active")  # Previous migration

    # Insert test data in the pre-005 schema
    with alembic_engine.connect() as conn:
        conn.execute(text("""
            INSERT INTO users (id, email, is_active, created_at)
            VALUES
                (1, 'active@example.com', TRUE, NOW()),
                (2, 'inactive@example.com', FALSE, NOW()),
                (3, 'also_active@example.com', TRUE, NOW())
        """))
        conn.commit()

    yield alembic_engine

    # Clean up
    alembic_runner.migrate_down_to("base")


def test_backfill_status_from_is_active(db_before_status_migration, alembic_runner):
    """Verify that migration 005 correctly sets status from is_active."""
    engine = db_before_status_migration

    # Run the data migration
    alembic_runner.migrate_up_one()  # Apply migration 005

    with engine.connect() as conn:
        result = conn.execute(
            text("SELECT id, is_active, status FROM users ORDER BY id")
        ).fetchall()

    assert result[0] == (1, True, "active")
    assert result[1] == (2, False, "inactive")
    assert result[2] == (3, True, "active")


def test_backfill_handles_null_is_active(db_before_status_migration, alembic_runner):
    """Edge case: users with NULL is_active should get 'inactive' status."""
    engine = db_before_status_migration

    with engine.connect() as conn:
        conn.execute(text("""
            INSERT INTO users (id, email, is_active, created_at)
            VALUES (4, 'null@example.com', NULL, NOW())
        """))
        conn.commit()

    alembic_runner.migrate_up_one()

    with engine.connect() as conn:
        row = conn.execute(
            text("SELECT status FROM users WHERE id = 4")
        ).fetchone()

    # NULL is_active should be treated as inactive
    assert row[0] == "inactive"


def test_backfill_migration_downgrade_removes_column(
    db_before_status_migration, alembic_runner, alembic_engine
):
    """Verify downgrade removes the status column."""
    from sqlalchemy import inspect

    alembic_runner.migrate_up_one()  # Apply 005

    inspector = inspect(alembic_engine)
    columns_before = {c["name"] for c in inspector.get_columns("users")}
    assert "status" in columns_before

    alembic_runner.migrate_down_one()  # Revert 005

    inspector = inspect(alembic_engine)
    columns_after = {c["name"] for c in inspector.get_columns("users")}
    assert "status" not in columns_after
```

### migrate_up_to(revision) — Named Revisions

pytest-alembic accepts both revision IDs and branch labels:

```python
# By revision ID (hex)
alembic_runner.migrate_up_to("a1b2c3d4e5f6")

# By relative label
alembic_runner.migrate_up_to("head")
alembic_runner.migrate_down_to("base")

# By custom label (if defined in the migration file)
# In migration file: revision = "add_users_table", down_revision = "base"
alembic_runner.migrate_up_to("add_users_table")

# One step at a time
alembic_runner.migrate_up_one()    # Upgrade by 1
alembic_runner.migrate_down_one()  # Downgrade by 1
```

---

## Integration with Testcontainers

### Session-Scoped Container + Function-Scoped Migrations

The recommended approach: one PostgreSQL container for the session (fast), but migration state reset per test that needs it (isolated).

```python
# tests/conftest.py
import pytest
from alembic.config import Config
from sqlalchemy import create_engine
from sqlalchemy.pool import NullPool
from testcontainers.postgres import PostgresContainer


@pytest.fixture(scope="session")
def postgres_container():
    """Single container for the entire test session."""
    with PostgresContainer("postgres:16-alpine") as pg:
        yield pg


@pytest.fixture(scope="session")
def alembic_engine(postgres_container):
    """Session-scoped engine for alembic_runner to use."""
    engine = create_engine(
        postgres_container.get_connection_url(),
        poolclass=NullPool,
    )
    yield engine
    engine.dispose()


@pytest.fixture
def alembic_config(alembic_engine):
    """Function-scoped Alembic config pointing at the test DB."""
    cfg = Config("alembic.ini")
    cfg.set_main_option("sqlalchemy.url", str(alembic_engine.url))
    return cfg


@pytest.fixture(scope="session", autouse=True)
def run_migrations_once(alembic_engine):
    """
    Run migrations to head once at session start for all non-migration tests.
    Migration-specific tests use alembic_runner to control state themselves.
    """
    from alembic import command
    cfg = Config("alembic.ini")
    cfg.set_main_option("sqlalchemy.url", str(alembic_engine.url))
    command.upgrade(cfg, "head")
    yield
    # Optional session teardown
    command.downgrade(cfg, "base")
```

### Using Multiple Schemas for Isolation

When running migration tests alongside regular integration tests, use PostgreSQL schemas to avoid conflicts:

```python
@pytest.fixture
def isolated_migration_schema(postgres_container):
    """
    Create an isolated schema for a single migration test.
    The migration runs against this schema, leaving the main schema untouched.
    """
    import uuid
    from sqlalchemy import create_engine, text
    from sqlalchemy.pool import NullPool

    schema = f"migration_test_{uuid.uuid4().hex[:8]}"
    base_url = postgres_container.get_connection_url()

    # Create the schema
    admin_engine = create_engine(base_url, poolclass=NullPool)
    with admin_engine.connect() as conn:
        conn.execute(text(f"CREATE SCHEMA {schema}"))
        conn.commit()
    admin_engine.dispose()

    # Engine pointing at isolated schema
    schema_url = f"{base_url}?options=-csearch_path={schema}"
    engine = create_engine(schema_url, poolclass=NullPool)

    yield engine

    engine.dispose()
    admin_engine = create_engine(base_url, poolclass=NullPool)
    with admin_engine.connect() as conn:
        conn.execute(text(f"DROP SCHEMA {schema} CASCADE"))
        conn.commit()
    admin_engine.dispose()


@pytest.fixture
def alembic_config_isolated(isolated_migration_schema):
    cfg = Config("alembic.ini")
    cfg.set_main_option("sqlalchemy.url", str(isolated_migration_schema.url))
    return cfg
```

---

## Migration State Inspection

### Checking Table Existence Before/After Migration

```python
# tests/test_migration_003_orders.py
import pytest
from sqlalchemy import inspect as sa_inspect, text


def test_orders_table_created_by_migration_003(alembic_runner, alembic_engine):
    """Migration 003 creates the orders table with correct schema."""
    alembic_runner.migrate_up_to("002_add_email_index")  # State before orders

    inspector = sa_inspect(alembic_engine)
    assert "orders" not in inspector.get_table_names()

    alembic_runner.migrate_up_to("003_add_orders_table")

    inspector = sa_inspect(alembic_engine)
    assert "orders" in inspector.get_table_names()

    # Verify column structure
    columns = {c["name"]: c for c in inspector.get_columns("orders")}
    assert "id" in columns
    assert "user_id" in columns
    assert "total" in columns
    assert "status" in columns

    # Verify nullable constraints
    assert columns["user_id"]["nullable"] is False
    assert columns["status"]["nullable"] is False

    # Verify foreign key
    fks = inspector.get_foreign_keys("orders")
    fk_targets = {fk["referred_table"] for fk in fks}
    assert "users" in fk_targets


def test_orders_table_removed_by_downgrade(alembic_runner, alembic_engine):
    """Downgrading migration 003 drops the orders table."""
    alembic_runner.migrate_up_to("003_add_orders_table")

    inspector = sa_inspect(alembic_engine)
    assert "orders" in inspector.get_table_names()

    alembic_runner.migrate_down_one()

    inspector = sa_inspect(alembic_engine)
    assert "orders" not in inspector.get_table_names()
```

### Testing Index Creation in Migrations

```python
def test_migration_004_adds_composite_index(alembic_runner, alembic_engine):
    """Migration 004 adds a composite index on (orders.user_id, orders.created_at)."""
    alembic_runner.migrate_up_to("003_add_orders_table")

    inspector = sa_inspect(alembic_engine)
    indexes_before = {idx["name"] for idx in inspector.get_indexes("orders")}
    assert "ix_orders_user_created" not in indexes_before

    alembic_runner.migrate_up_one()  # Apply 004

    inspector = sa_inspect(alembic_engine)
    indexes = inspector.get_indexes("orders")
    index_map = {idx["name"]: idx for idx in indexes}

    assert "ix_orders_user_created" in index_map
    idx = index_map["ix_orders_user_created"]
    assert set(idx["column_names"]) == {"user_id", "created_at"}
    assert idx["unique"] is False


def test_migration_004_partial_index(alembic_runner, alembic_engine):
    """Migration 004 adds a partial index for pending orders only."""
    alembic_runner.migrate_up_to("004_add_composite_index")

    # Inspect using raw SQL for partial indexes (SQLAlchemy Inspector
    # doesn't expose partial index predicates for all backends)
    with alembic_engine.connect() as conn:
        result = conn.execute(text("""
            SELECT indexname, indexdef
            FROM pg_indexes
            WHERE tablename = 'orders'
              AND indexname = 'ix_orders_pending'
        """)).fetchone()

    assert result is not None
    assert "WHERE" in result.indexdef
    assert "pending" in result.indexdef
```

### Testing Constraint Creation

```python
def test_unique_constraint_added(alembic_runner, alembic_engine):
    """Migration adds UNIQUE constraint on users.email."""
    alembic_runner.migrate_up_to("head")

    inspector = sa_inspect(alembic_engine)
    unique_constraints = inspector.get_unique_constraints("users")
    constraint_columns = [
        set(uc["column_names"]) for uc in unique_constraints
    ]
    assert {"email"} in constraint_columns


def test_check_constraint_added(alembic_runner, alembic_engine):
    """Migration adds CHECK constraint on orders.status."""
    alembic_runner.migrate_up_to("head")

    # Check constraints via raw SQL (pg_constraint)
    with alembic_engine.connect() as conn:
        result = conn.execute(text("""
            SELECT conname, consrc
            FROM pg_constraint
            WHERE conrelid = 'orders'::regclass
              AND contype = 'c'
        """)).fetchall()

    constraint_names = {r.conname for r in result}
    assert "ck_orders_status_valid" in constraint_names
```

---

## Common Anti-Patterns

### Anti-Pattern 1: Using Base.metadata.create_all() Instead of Alembic

```python
# BAD: Bypasses migrations entirely
@pytest.fixture(scope="session")
def engine():
    engine = create_engine(test_url)
    Base.metadata.create_all(engine)  # ← Doesn't test migration scripts!
    yield engine

# GOOD: Use Alembic migrations
@pytest.fixture(scope="session")
def engine(alembic_config):
    engine = create_engine(test_url)
    command.upgrade(alembic_config, "head")  # ← Actually runs migration scripts
    yield engine
```

### Anti-Pattern 2: Leaving the DB in a Migrated State Between Tests

```python
# BAD: Tests share mutation state across runs
def test_migration_a(alembic_runner):
    alembic_runner.migrate_up_to("head")
    # No teardown — next test starts from head, not base

def test_migration_b(alembic_runner):
    # This test assumes base, but DB is still at head from previous test
    alembic_runner.migrate_up_to("001")  # May fail or behave incorrectly

# GOOD: Reset state in fixtures
@pytest.fixture
def clean_migration_state(alembic_runner):
    alembic_runner.migrate_down_to("base")
    yield alembic_runner
    alembic_runner.migrate_down_to("base")
```

### Anti-Pattern 3: Not Testing the Downgrade Path

```python
# BAD: Migration file with empty downgrade
def upgrade():
    op.add_column("users", sa.Column("bio", sa.Text()))

def downgrade():
    pass  # ← Will break test_up_down_consistency

# GOOD: Always implement downgrade
def downgrade():
    op.drop_column("users", "bio")
```

### Anti-Pattern 4: Hardcoded Revision IDs in Tests

```python
# BAD: Revision IDs are auto-generated hex strings that change
alembic_runner.migrate_up_to("a1b2c3d4e5f6")  # Breaks when revision is recreated

# GOOD: Use the file slug or relative navigation
alembic_runner.migrate_up_to("add_users_table")  # Human-readable slug
alembic_runner.migrate_up_one()                   # Relative: +1 step
```

### Anti-Pattern 5: Not Running Migrations Before Unit Tests

When unit tests import models, SQLAlchemy may auto-generate schema in memory. This masks migration bugs:

```python
# BAD in unit tests: model imports work even if migrations are wrong
from myapp.models import User  # This works regardless of migration state

# GOOD: At least run test_model_definitions_match_ddl in CI
# to catch schema drift between models and migrations
```

### Anti-Pattern 6: One Shared Test Database for All Test Types

```python
# BAD: Migration tests and integration tests share the same DB
# Migration test runs migrate_down_to("base") → integration tests break

# GOOD: Use separate databases or schemas
# - Regular tests: shared DB, savepoint rollback
# - Migration tests: isolated schema per test
```
