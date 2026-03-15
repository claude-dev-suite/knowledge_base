# Python Integration Testing — SQLAlchemy 2.0 Patterns

Deep reference for testing applications built with SQLAlchemy 2.0: session management, async engines, testcontainers, Alembic, factory_boy, and connection pool configuration.

**Official Documentation:**
- SQLAlchemy 2.0: https://docs.sqlalchemy.org/en/20/
- SQLAlchemy testing guide: https://docs.sqlalchemy.org/en/20/orm/session_transaction.html#joining-a-session-into-an-external-transaction-the-join-external-transaction-recipe
- factory_boy: https://factoryboy.readthedocs.io/
- testcontainers-python: https://testcontainers-python.readthedocs.io/

---

## Table of Contents

1. [SQLAlchemy 2.0 Savepoint Pattern](#sqlalchemy-20-savepoint-pattern)
2. [Async SQLAlchemy Pattern](#async-sqlalchemy-pattern)
3. [Testcontainers + SQLAlchemy](#testcontainers--sqlalchemy)
4. [Alembic Integration with Tests](#alembic-integration-with-tests)
5. [factory_boy + SQLAlchemy](#factory_boy--sqlalchemy)
6. [Connection Pool Configuration for Tests](#connection-pool-configuration-for-tests)
7. [Multiple Database Testing](#multiple-database-testing)
8. [Complete Production-Ready conftest.py](#complete-production-ready-conftestpy)

---

## SQLAlchemy 2.0 Savepoint Pattern

The canonical approach from the SQLAlchemy docs: wrap every test in an outer transaction, create a `SAVEPOINT` for the session to commit against, then roll back the outer transaction after the test. The database is restored to its pre-test state with zero truncation cost.

### How It Works

```
BEGIN                          ← outer transaction (never committed)
  SAVEPOINT pytest_savepoint   ← created by join_transaction_mode="create_savepoint"
    INSERT INTO users ...      ← application code
    UPDATE orders ...          ← application code
  RELEASE SAVEPOINT            ← session.commit() → releases savepoint, not real commit
  SAVEPOINT pytest_savepoint   ← new savepoint for next commit
ROLLBACK                       ← test teardown: all inserts/updates vanish
```

### Engine Fixture (Session Scope)

```python
# tests/conftest.py
import pytest
from sqlalchemy import create_engine, event
from sqlalchemy.orm import Session, sessionmaker
from sqlalchemy.pool import NullPool

from myapp.database import Base


@pytest.fixture(scope="session")
def engine():
    """
    Create the SQLAlchemy engine once for the entire test session.

    NullPool: each connection is opened and closed for each checkout.
    This prevents connections from being reused across tests, avoiding
    cross-test state contamination. Slightly slower but safe.
    """
    _engine = create_engine(
        "postgresql+psycopg2://testuser:testpassword@localhost:5432/testdb",
        poolclass=NullPool,
        echo=False,  # Set True to log all SQL statements during debugging
    )
    Base.metadata.create_all(_engine)
    yield _engine
    Base.metadata.drop_all(_engine)
    _engine.dispose()
```

### Session Fixture (Function Scope) — The Canonical Pattern

```python
@pytest.fixture
def db_session(engine):
    """
    Provide a transactional SQLAlchemy Session for each test.

    join_transaction_mode="create_savepoint":
        When the Session joins the external connection's transaction,
        it creates a SAVEPOINT instead of assuming control of the connection.
        This means session.commit() → RELEASE SAVEPOINT (not real COMMIT).

    After the test: rollback() erases everything the test wrote.
    """
    connection = engine.connect()
    transaction = connection.begin()

    session = Session(
        bind=connection,
        join_transaction_mode="create_savepoint",
    )

    yield session

    session.close()
    transaction.rollback()
    connection.close()
```

### Application Code That Calls commit()

The savepoint pattern is specifically designed to handle application code that calls `session.commit()`. This is the scenario that trips up many developers:

```python
# myapp/repositories/user.py (application code)
class UserRepository:
    def __init__(self, session: Session):
        self.session = session

    def create(self, email: str, name: str) -> User:
        user = User(email=email, name=name)
        self.session.add(user)
        self.session.commit()   # ← This would normally commit to DB
                                #   With savepoint pattern: creates SAVEPOINT instead
        self.session.refresh(user)
        return user
```

```python
# tests/integration/test_user_repository.py
def test_create_user(db_session):
    repo = UserRepository(db_session)

    user = repo.create(email="alice@example.com", name="Alice")

    # commit() was called inside create(), but it only released the savepoint
    # The outer transaction is still open
    assert user.id is not None
    assert user.email == "alice@example.com"

    # Verify in DB (within same connection)
    found = db_session.get(User, user.id)
    assert found is not None

    # After this test: transaction.rollback() removes the user from DB
    # The next test starts with a clean DB
```

### Handling Nested Transactions in Application Code

If application code uses `session.begin_nested()` (explicit SAVEPOINTs), the pattern still works because SQLAlchemy manages the SAVEPOINT stack:

```python
# Application code using explicit savepoints
def transfer_funds(session, from_account_id, to_account_id, amount):
    with session.begin_nested():  # SAVEPOINT transfer_savepoint
        from_acct = session.get(Account, from_account_id, with_for_update=True)
        to_acct = session.get(Account, to_account_id, with_for_update=True)

        if from_acct.balance < amount:
            raise InsufficientFundsError()

        from_acct.balance -= amount
        to_acct.balance += amount
    # RELEASE SAVEPOINT transfer_savepoint (not COMMIT)
    session.commit()  # Becomes RELEASE of outer pytest savepoint
```

```python
# Test still works correctly
def test_transfer_funds(db_session):
    alice = AccountFactory(balance=Decimal("1000.00"), _session=db_session)
    bob = AccountFactory(balance=Decimal("0.00"), _session=db_session)
    db_session.flush()

    transfer_funds(db_session, alice.id, bob.id, Decimal("200.00"))

    db_session.expire_all()
    assert db_session.get(Account, alice.id).balance == Decimal("800.00")
    assert db_session.get(Account, bob.id).balance == Decimal("200.00")
```

### SQLAlchemy 2.0 Style with `with Session()` context manager

```python
# Modern SQLAlchemy 2.0 style using context manager
@pytest.fixture
def db_session(engine):
    connection = engine.connect()
    transaction = connection.begin()

    # SQLAlchemy 2.0 Session construction
    with Session(
        bind=connection,
        join_transaction_mode="create_savepoint",
    ) as session:
        yield session

    transaction.rollback()
    connection.close()
```

---

## Async SQLAlchemy Pattern

For applications using `AsyncSession` and `create_async_engine` (FastAPI with async handlers, Starlette, etc.).

### Dependencies

```bash
pip install sqlalchemy[asyncio] asyncpg pytest-asyncio
```

### Async Engine and Session Fixtures

```python
# tests/conftest.py
import pytest
import pytest_asyncio
from sqlalchemy.ext.asyncio import (
    AsyncEngine,
    AsyncSession,
    create_async_engine,
    async_sessionmaker,
)
from sqlalchemy.pool import NullPool

from myapp.database import Base


@pytest_asyncio.fixture(scope="session")
async def async_engine():
    """Session-scoped async engine using asyncpg driver."""
    engine = create_async_engine(
        "postgresql+asyncpg://testuser:testpassword@localhost:5432/testdb",
        poolclass=NullPool,
        echo=False,
    )
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    yield engine

    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine.dispose()


@pytest_asyncio.fixture
async def async_session(async_engine):
    """
    Async savepoint pattern for function-scoped isolation.

    Note: AsyncSession does not directly support join_transaction_mode
    in the same way. Use begin_nested() explicitly.
    """
    async with async_engine.connect() as connection:
        async with connection.begin() as transaction:
            # Create a nested transaction (SAVEPOINT)
            async with connection.begin_nested() as savepoint:
                async_session_factory = async_sessionmaker(
                    bind=connection,
                    class_=AsyncSession,
                    expire_on_commit=False,
                )
                async with async_session_factory() as session:
                    yield session

                # Roll back to savepoint after each test
                await savepoint.rollback()

            await transaction.rollback()
```

### Alternative Async Pattern — Direct Connection Binding

```python
@pytest_asyncio.fixture
async def async_session(async_engine):
    """
    Simpler async isolation: open connection, begin transaction,
    bind session to connection, rollback after test.
    """
    conn = await async_engine.connect()
    await conn.begin()

    # bind session to this specific connection
    session = AsyncSession(bind=conn, join_transaction_mode="create_savepoint")

    yield session

    await session.close()
    await conn.rollback()
    await conn.close()
```

### pytest-asyncio Configuration

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"  # All async tests/fixtures are auto-detected
# Or: asyncio_mode = "strict" — requires explicit @pytest.mark.asyncio
```

### Async Test Example

```python
# tests/integration/test_async_user_service.py
import pytest
from myapp.services.user import AsyncUserService


@pytest.mark.asyncio
async def test_create_user_async(async_session):
    service = AsyncUserService(async_session)

    user = await service.create(email="async@example.com", name="Async Alice")

    assert user.id is not None
    # Verify via session query
    result = await async_session.get(User, user.id)
    assert result.email == "async@example.com"


@pytest.mark.asyncio
async def test_concurrent_creates(async_session):
    import asyncio
    service = AsyncUserService(async_session)

    users = await asyncio.gather(
        service.create(email="user1@example.com", name="User 1"),
        service.create(email="user2@example.com", name="User 2"),
        service.create(email="user3@example.com", name="User 3"),
    )

    assert len(users) == 3
    assert all(u.id is not None for u in users)
```

---

## Testcontainers + SQLAlchemy

### Session-Scoped PostgresContainer

```python
# tests/conftest.py
import pytest
from testcontainers.postgres import PostgresContainer
from sqlalchemy import create_engine
from sqlalchemy.pool import NullPool

from myapp.database import Base


@pytest.fixture(scope="session")
def postgres_container():
    """
    Start PostgreSQL once for all tests.
    The container uses the testcontainers default credentials
    unless overridden via environment variables.
    """
    with PostgresContainer(
        image="postgres:16-alpine",
        username="testuser",
        password="testpassword",
        dbname="testdb",
        # driver="psycopg2",  # Default; or "asyncpg" for async
    ) as pg:
        yield pg


@pytest.fixture(scope="session")
def engine(postgres_container):
    """Engine bound to the testcontainers PostgreSQL instance."""
    url = postgres_container.get_connection_url()
    # get_connection_url() returns: postgresql+psycopg2://testuser:testpassword@localhost:PORT/testdb

    _engine = create_engine(url, poolclass=NullPool)
    Base.metadata.create_all(_engine)
    yield _engine
    Base.metadata.drop_all(_engine)
    _engine.dispose()


@pytest.fixture
def db_session(engine):
    """Standard savepoint rollback fixture."""
    connection = engine.connect()
    transaction = connection.begin()
    session = Session(bind=connection, join_transaction_mode="create_savepoint")
    yield session
    session.close()
    transaction.rollback()
    connection.close()
```

### Async Testcontainer with asyncpg

```python
@pytest.fixture(scope="session")
def postgres_container():
    with PostgresContainer("postgres:16-alpine") as pg:
        yield pg


@pytest_asyncio.fixture(scope="session")
async def async_engine(postgres_container):
    """Convert sync URL to asyncpg URL."""
    sync_url = postgres_container.get_connection_url()
    # Convert: postgresql+psycopg2://... → postgresql+asyncpg://...
    async_url = sync_url.replace("postgresql+psycopg2", "postgresql+asyncpg")

    engine = create_async_engine(async_url, poolclass=NullPool)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine.dispose()
```

### Custom Wait Strategy

```python
from testcontainers.core.waiting_utils import wait_for_logs
from testcontainers.postgres import PostgresContainer


@pytest.fixture(scope="session")
def postgres_container():
    container = PostgresContainer("postgres:16-alpine")
    container.start()
    # Wait until PostgreSQL is ready to accept connections
    wait_for_logs(container, "database system is ready to accept connections", timeout=30)
    yield container
    container.stop()
```

---

## Alembic Integration with Tests

### Running Migrations Before Test Session

Instead of `Base.metadata.create_all()`, use Alembic to create the schema. This ensures your tests run against the actual migration state, catching migration errors early.

```python
# tests/conftest.py
import pytest
from alembic import command
from alembic.config import Config
from sqlalchemy import create_engine
from sqlalchemy.pool import NullPool


@pytest.fixture(scope="session")
def engine(postgres_container):
    url = postgres_container.get_connection_url()
    _engine = create_engine(url, poolclass=NullPool)
    yield _engine
    _engine.dispose()


@pytest.fixture(scope="session", autouse=True)
def run_migrations(engine):
    """
    Apply all Alembic migrations before the test session starts.
    Uses alembic.command.upgrade() so actual migration scripts run.
    """
    alembic_cfg = Config("alembic.ini")
    # Override the database URL to point to the test DB
    alembic_cfg.set_main_option("sqlalchemy.url", str(engine.url))

    command.upgrade(alembic_cfg, "head")
    yield
    # Optional: downgrade after session
    # command.downgrade(alembic_cfg, "base")
```

### Schema Isolation Per Test with Alembic

For expensive test suites, use PostgreSQL schemas to isolate migration state:

```python
# tests/conftest.py
import pytest
from sqlalchemy import text


@pytest.fixture
def isolated_schema(engine, worker_id):
    """
    Create a per-test PostgreSQL schema, run migrations into it,
    then drop it after the test.
    """
    schema_name = f"test_{worker_id}_{uuid.uuid4().hex[:8]}"

    with engine.connect() as conn:
        conn.execute(text(f"CREATE SCHEMA {schema_name}"))
        conn.execute(text(f"SET search_path TO {schema_name}"))
        conn.commit()

    alembic_cfg = Config("alembic.ini")
    alembic_cfg.set_main_option("sqlalchemy.url", str(engine.url))
    alembic_cfg.set_section_option("alembic", "version_locations", "migrations/versions")

    # Point alembic at our isolated schema
    with engine.connect() as conn:
        conn.execute(text(f"SET search_path TO {schema_name}"))
        conn.commit()

    command.upgrade(alembic_cfg, "head")

    yield schema_name

    with engine.connect() as conn:
        conn.execute(text(f"DROP SCHEMA {schema_name} CASCADE"))
        conn.commit()
```

---

## factory_boy + SQLAlchemy

### SQLAlchemyModelFactory Setup

```bash
pip install factory-boy
```

```python
# tests/factories.py
import factory
import factory.fuzzy
from decimal import Decimal
from factory.alchemy import SQLAlchemyModelFactory

from myapp.models import User, Order, Product, OrderItem


class UserFactory(SQLAlchemyModelFactory):
    class Meta:
        model = User
        sqlalchemy_session = None       # Set via conftest injection
        sqlalchemy_session_persistence = "flush"  # flush after create, not commit

    id = factory.Sequence(lambda n: n + 1)
    email = factory.LazyAttribute(lambda o: f"{o.username}@example.com")
    username = factory.Sequence(lambda n: f"user_{n:04d}")
    first_name = factory.Faker("first_name")
    last_name = factory.Faker("last_name")
    is_active = True
    created_at = factory.Faker("date_time_this_year", tzinfo=None)


class ProductFactory(SQLAlchemyModelFactory):
    class Meta:
        model = Product
        sqlalchemy_session = None
        sqlalchemy_session_persistence = "flush"

    name = factory.Sequence(lambda n: f"Product {n}")
    sku = factory.Sequence(lambda n: f"SKU-{n:06d}")
    price = factory.fuzzy.FuzzyDecimal(1.00, 999.99, precision=2)
    stock_quantity = factory.fuzzy.FuzzyInteger(0, 1000)


class OrderFactory(SQLAlchemyModelFactory):
    class Meta:
        model = Order
        sqlalchemy_session = None
        sqlalchemy_session_persistence = "flush"

    user = factory.SubFactory(UserFactory)
    status = factory.fuzzy.FuzzyChoice(["pending", "confirmed", "shipped", "delivered"])
    total = factory.LazyAttribute(
        lambda o: sum(item.subtotal for item in o.items)
        if hasattr(o, "items") and o.items
        else Decimal("0.00")
    )
```

### Session Injection via conftest.py

```python
# tests/integration/conftest.py
import pytest
from tests.factories import UserFactory, ProductFactory, OrderFactory


@pytest.fixture(autouse=True)
def inject_session_into_factories(db_session):
    """
    Inject the function-scoped test session into all factories.
    autouse=True ensures this runs before any test in this directory.
    """
    factories = [UserFactory, ProductFactory, OrderFactory]
    for factory_class in factories:
        factory_class._meta.sqlalchemy_session = db_session

    yield

    # Clear session reference to avoid stale session use
    for factory_class in factories:
        factory_class._meta.sqlalchemy_session = None
```

### Using Factories in Tests

```python
# tests/integration/test_order_service.py
import pytest
from tests.factories import UserFactory, ProductFactory, OrderFactory

pytestmark = [pytest.mark.integration, pytest.mark.database]


def test_calculate_order_total(db_session):
    user = UserFactory()
    product1 = ProductFactory(price=Decimal("10.00"))
    product2 = ProductFactory(price=Decimal("25.50"))

    order = OrderFactory(user=user)
    OrderItemFactory(order=order, product=product1, quantity=2)
    OrderItemFactory(order=order, product=product2, quantity=1)

    db_session.flush()  # Assign IDs without committing

    service = OrderService(db_session)
    total = service.recalculate_total(order.id)

    assert total == Decimal("45.50")  # 2 * 10.00 + 1 * 25.50


def test_batch_create_products(db_session):
    """Create 50 products at once using create_batch."""
    products = ProductFactory.create_batch(50)
    db_session.flush()

    count = db_session.query(Product).count()
    assert count == 50


def test_factory_with_overrides(db_session):
    """Override specific fields on factory creation."""
    admin_user = UserFactory(
        email="admin@example.com",
        username="admin",
        is_active=True,
    )
    inactive_user = UserFactory(is_active=False)

    db_session.flush()

    assert admin_user.email == "admin@example.com"
    assert not inactive_user.is_active
```

### SQLAlchemyOptions for Dynamic Session

For more complex scenarios where the session changes per test:

```python
# tests/factories.py
from factory.alchemy import SQLAlchemyModelFactory, SQLAlchemyOptions


class BaseFactory(SQLAlchemyModelFactory):
    """Base factory that reads session from a thread-local or context var."""

    class Meta:
        abstract = True
        sqlalchemy_session_persistence = "flush"

    @classmethod
    def _create(cls, model_class, *args, **kwargs):
        """Override to support dynamic session injection."""
        session = cls._meta.sqlalchemy_session
        if session is None:
            raise RuntimeError(
                f"No SQLAlchemy session set for {cls.__name__}. "
                "Call inject_session_into_factories fixture first."
            )
        return super()._create(model_class, *args, **kwargs)
```

---

## Connection Pool Configuration for Tests

### NullPool — The Testing Standard

`NullPool` disables connection pooling entirely. Each `engine.connect()` opens a new physical connection, and closing it immediately releases it back to the database. This is the safest choice for tests:

```python
from sqlalchemy.pool import NullPool, StaticPool

# NullPool: fresh connection for each checkout
engine = create_engine(database_url, poolclass=NullPool)

# StaticPool: single connection reused (useful for SQLite in-memory)
engine = create_engine("sqlite:///:memory:", poolclass=StaticPool)
```

**Why NullPool for tests:**
- Prevents one test's uncommitted data from being visible in another test's connection
- Avoids "connection already in transaction" errors with xdist workers
- Eliminates connection lease timeouts during long test sessions
- Each test gets a fresh connection with no cached statement state

### pool_pre_ping — Avoiding Stale Connections

When using a persistent pool (not NullPool), `pool_pre_ping=True` issues a `SELECT 1` before each checkout to detect stale connections:

```python
# For integration test environments where the DB might restart
engine = create_engine(
    database_url,
    pool_pre_ping=True,       # Issue SELECT 1 before using pooled connection
    pool_recycle=300,          # Recycle connections older than 5 minutes
    pool_size=2,               # Small pool for tests
    max_overflow=2,            # At most 4 total connections
)
```

### pool_recycle — Long-Running Test Suites

MySQL and some proxies (PgBouncer) close idle connections after a timeout. `pool_recycle` ensures connections are replaced before they're dropped:

```python
# MySQL: default wait_timeout is 8 hours, but managed instances may be lower
mysql_engine = create_engine(
    "mysql+pymysql://user:pass@localhost/testdb",
    pool_recycle=1800,  # Recycle after 30 minutes
    pool_pre_ping=True,
)
```

### Configuration Matrix for Test Types

```python
# tests/conftest.py
import os
from sqlalchemy.pool import NullPool, QueuePool

def get_test_engine_kwargs():
    """Return engine kwargs based on the test environment."""
    if os.getenv("CI"):
        # CI: NullPool, no timeout issues, fast startup
        return {"poolclass": NullPool}
    elif os.getenv("PYTEST_XDIST_WORKER"):
        # xdist worker: NullPool to avoid cross-worker connection sharing
        return {"poolclass": NullPool}
    else:
        # Local dev: small pool, pre-ping, recycle
        return {
            "pool_size": 2,
            "max_overflow": 2,
            "pool_pre_ping": True,
            "pool_recycle": 600,
        }


@pytest.fixture(scope="session")
def engine(database_url):
    _engine = create_engine(database_url, **get_test_engine_kwargs())
    Base.metadata.create_all(_engine)
    yield _engine
    _engine.dispose()
```

---

## Multiple Database Testing

### Testing Cross-Database Operations

```python
# tests/conftest.py
import pytest
from testcontainers.postgres import PostgresContainer
from testcontainers.mysql import MySqlContainer
from sqlalchemy import create_engine
from sqlalchemy.pool import NullPool

from myapp.database.primary import PrimaryBase
from myapp.database.analytics import AnalyticsBase


@pytest.fixture(scope="session")
def primary_db():
    with PostgresContainer("postgres:16-alpine") as pg:
        yield pg


@pytest.fixture(scope="session")
def analytics_db():
    with MySqlContainer("mysql:8.0") as mysql:
        yield mysql


@pytest.fixture(scope="session")
def primary_engine(primary_db):
    engine = create_engine(primary_db.get_connection_url(), poolclass=NullPool)
    PrimaryBase.metadata.create_all(engine)
    yield engine
    PrimaryBase.metadata.drop_all(engine)
    engine.dispose()


@pytest.fixture(scope="session")
def analytics_engine(analytics_db):
    url = analytics_db.get_connection_url()
    engine = create_engine(url, poolclass=NullPool)
    AnalyticsBase.metadata.create_all(engine)
    yield engine
    AnalyticsBase.metadata.drop_all(engine)
    engine.dispose()


@pytest.fixture
def primary_session(primary_engine):
    conn = primary_engine.connect()
    txn = conn.begin()
    session = Session(bind=conn, join_transaction_mode="create_savepoint")
    yield session
    session.close()
    txn.rollback()
    conn.close()


@pytest.fixture
def analytics_session(analytics_engine):
    conn = analytics_engine.connect()
    txn = conn.begin()
    session = Session(bind=conn, join_transaction_mode="create_savepoint")
    yield session
    session.close()
    txn.rollback()
    conn.close()
```

```python
# tests/integration/test_sync_service.py
def test_sync_user_to_analytics(primary_session, analytics_session):
    """Test that the sync service copies users to analytics DB."""
    from myapp.services.sync import UserSyncService

    # Create user in primary DB
    user = UserFactory(_session=primary_session, email="sync@example.com")
    primary_session.flush()

    # Run sync
    service = UserSyncService(primary_session, analytics_session)
    service.sync_user(user.id)

    # Verify in analytics DB
    analytics_session.flush()
    analytics_user = analytics_session.query(AnalyticsUser).filter_by(
        source_id=user.id
    ).first()
    assert analytics_user is not None
    assert analytics_user.email == "sync@example.com"
```

---

## Complete Production-Ready conftest.py

Full `conftest.py` for a FastAPI + SQLAlchemy 2.0 + testcontainers project with all patterns combined.

```python
# tests/conftest.py
"""
Root conftest.py for integration tests.

Provides:
- Session-scoped PostgreSQL and Redis containers
- Session-scoped SQLAlchemy engine with Alembic migrations
- Function-scoped database sessions with savepoint rollback
- FastAPI test client with overridden dependencies
- Redis client fixture
- Automatic factory session injection
"""
import json
import os
import pytest
import pytest_asyncio
from typing import Generator, AsyncGenerator

from alembic import command
from alembic.config import Config
from fastapi.testclient import TestClient
from httpx import AsyncClient, ASGITransport
from sqlalchemy import create_engine, event
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy.orm import Session, sessionmaker
from sqlalchemy.pool import NullPool
from testcontainers.postgres import PostgresContainer
from testcontainers.redis import RedisContainer

from myapp.main import app
from myapp.database import Base, get_db, get_async_db
from tests.factories import UserFactory, ProductFactory, OrderFactory


# ---------------------------------------------------------------------------
# Infrastructure Containers (session-scoped)
# ---------------------------------------------------------------------------

@pytest.fixture(scope="session")
def postgres_container():
    """PostgreSQL 16 container, started once for the entire test session."""
    with PostgresContainer(
        image="postgres:16-alpine",
        username="testuser",
        password="testpassword",
        dbname="testdb",
    ) as pg:
        yield pg


@pytest.fixture(scope="session")
def redis_container():
    """Redis 7 container, started once for the entire test session."""
    with RedisContainer("redis:7-alpine") as redis:
        yield redis


# ---------------------------------------------------------------------------
# SQLAlchemy Sync Engine (session-scoped)
# ---------------------------------------------------------------------------

@pytest.fixture(scope="session")
def database_url(postgres_container) -> str:
    return postgres_container.get_connection_url()


@pytest.fixture(scope="session")
def engine(database_url):
    """
    Session-scoped engine. Runs Alembic migrations (not create_all)
    so migration scripts are actually tested.
    """
    _engine = create_engine(database_url, poolclass=NullPool, echo=False)

    alembic_cfg = Config("alembic.ini")
    alembic_cfg.set_main_option("sqlalchemy.url", database_url)
    command.upgrade(alembic_cfg, "head")

    yield _engine
    _engine.dispose()


# ---------------------------------------------------------------------------
# SQLAlchemy Async Engine (session-scoped)
# ---------------------------------------------------------------------------

@pytest_asyncio.fixture(scope="session")
async def async_engine(database_url):
    """Session-scoped async engine using asyncpg."""
    async_url = database_url.replace("postgresql+psycopg2", "postgresql+asyncpg")
    _engine = create_async_engine(async_url, poolclass=NullPool, echo=False)
    yield _engine
    await _engine.dispose()


# ---------------------------------------------------------------------------
# Database Sessions (function-scoped)
# ---------------------------------------------------------------------------

@pytest.fixture
def db_session(engine) -> Generator[Session, None, None]:
    """
    Function-scoped sync session with savepoint rollback.
    All writes are rolled back after the test — no manual cleanup needed.
    """
    connection = engine.connect()
    transaction = connection.begin()
    session = Session(bind=connection, join_transaction_mode="create_savepoint")

    yield session

    session.close()
    transaction.rollback()
    connection.close()


@pytest_asyncio.fixture
async def async_session(async_engine) -> AsyncGenerator[AsyncSession, None]:
    """Function-scoped async session with rollback."""
    async with async_engine.connect() as connection:
        async with connection.begin() as transaction:
            async_session_factory = async_sessionmaker(
                bind=connection,
                class_=AsyncSession,
                expire_on_commit=False,
            )
            async with async_session_factory() as session:
                yield session
            await transaction.rollback()


# ---------------------------------------------------------------------------
# Redis Client (session-scoped)
# ---------------------------------------------------------------------------

@pytest.fixture(scope="session")
def redis_client(redis_container):
    """Session-scoped Redis client connected to the test container."""
    import redis as redis_lib
    client = redis_lib.Redis(
        host=redis_container.get_container_host_ip(),
        port=int(redis_container.get_exposed_port(6379)),
        decode_responses=True,
    )
    yield client
    client.flushall()  # Clean up after session
    client.close()


@pytest.fixture(autouse=True)
def flush_redis(redis_client):
    """Flush Redis before each test for clean state."""
    redis_client.flushall()
    yield


# ---------------------------------------------------------------------------
# FastAPI Test Clients
# ---------------------------------------------------------------------------

@pytest.fixture
def client(db_session) -> Generator[TestClient, None, None]:
    """Sync FastAPI test client with overridden DB dependency."""
    def override_get_db():
        yield db_session

    app.dependency_overrides[get_db] = override_get_db
    with TestClient(app, raise_server_exceptions=True) as c:
        yield c
    app.dependency_overrides.clear()


@pytest_asyncio.fixture
async def async_client(async_session) -> AsyncGenerator[AsyncClient, None]:
    """Async httpx client for testing async FastAPI endpoints."""
    async def override_get_async_db():
        yield async_session

    app.dependency_overrides[get_async_db] = override_get_async_db
    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://testserver",
    ) as c:
        yield c
    app.dependency_overrides.clear()


# ---------------------------------------------------------------------------
# Factory Injection
# ---------------------------------------------------------------------------

@pytest.fixture(autouse=True)
def inject_factories(db_session):
    """Inject the test DB session into all factories. Runs before every test."""
    factories = [UserFactory, ProductFactory, OrderFactory]
    for f in factories:
        f._meta.sqlalchemy_session = db_session
    yield
    for f in factories:
        f._meta.sqlalchemy_session = None


# ---------------------------------------------------------------------------
# xdist Worker Identification
# ---------------------------------------------------------------------------

@pytest.fixture(scope="session")
def worker_id(request):
    """Identify the xdist worker, or 'master' for non-parallel runs."""
    if hasattr(request.config, "workerinput"):
        return request.config.workerinput["workerid"]
    return "master"
```
