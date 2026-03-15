# testcontainers-python — Database Patterns

**Official Documentation:** https://testcontainers-python.readthedocs.io/

---

## Table of Contents

1. [SQLAlchemy 2.0 Integration](#sqlalchemy-20-integration)
2. [Alembic Migrations with Testcontainers](#alembic-migrations-with-testcontainers)
3. [Async SQLAlchemy (asyncpg)](#async-sqlalchemy-asyncpg)
4. [Django with Testcontainers](#django-with-testcontainers)
5. [MongoDB Patterns](#mongodb-patterns)
6. [Multiple Database Containers](#multiple-database-containers)

---

## SQLAlchemy 2.0 Integration

### Session-Scoped Container + Per-Test Savepoint Isolation

This is the gold-standard pattern for fast, isolated integration tests with SQLAlchemy 2.0.
It starts the container once and uses savepoints (nested transactions) to roll back after each test.

```python
# tests/conftest.py
import pytest
from sqlalchemy import create_engine, event, text
from sqlalchemy.orm import sessionmaker, Session
from testcontainers.postgres import PostgresContainer

from myapp.database import Base  # your SQLAlchemy declarative base


@pytest.fixture(scope="session")
def postgres_container():
    """Single PostgreSQL container for the entire test session."""
    with PostgresContainer("postgres:16-alpine") as pg:
        yield pg


@pytest.fixture(scope="session")
def engine(postgres_container):
    """Session-scoped engine. Tables created once and torn down at end."""
    engine = create_engine(
        postgres_container.get_connection_url(),
        echo=False,
        pool_pre_ping=True,
    )
    Base.metadata.create_all(engine)
    yield engine
    Base.metadata.drop_all(engine)
    engine.dispose()


@pytest.fixture
def db_session(engine) -> Session:
    """
    Per-test database session using the savepoint (nested transaction) pattern.

    Each test runs inside a SAVEPOINT. When the test finishes, the savepoint
    is rolled back — the outer transaction is never committed, so the database
    remains clean for the next test.

    This is equivalent to SQLAlchemy's recommended
    'joining-a-session-into-an-external-transaction' pattern.
    See: https://docs.sqlalchemy.org/en/20/orm/session_transaction.html
    """
    connection = engine.connect()
    transaction = connection.begin()

    # Create session bound to the outer connection
    session = Session(bind=connection, join_transaction_mode="create_savepoint")

    yield session

    session.close()
    transaction.rollback()
    connection.close()
```

**Usage:**

```python
import pytest
from myapp.models import User, Post
from myapp.repositories import UserRepository


@pytest.mark.integration
def test_create_user(db_session):
    user = User(username="alice", email="alice@example.com")
    db_session.add(user)
    db_session.flush()  # write to DB within the savepoint

    assert user.id is not None
    assert db_session.query(User).count() == 1


@pytest.mark.integration
def test_user_isolation(db_session):
    """This test sees a clean DB — previous test's user was rolled back."""
    assert db_session.query(User).count() == 0


@pytest.mark.integration
def test_repository(db_session):
    repo = UserRepository(db_session)
    user = repo.create(username="bob", email="bob@example.com")
    found = repo.get_by_email("bob@example.com")
    assert found.username == "bob"
```

### Alternative: SQLAlchemy 2.0 Style (Connection + begin_nested)

```python
@pytest.fixture
def db_session(engine):
    """Pure 2.0-style using begin_nested()."""
    with engine.connect() as connection:
        with connection.begin() as transaction:
            with Session(bind=connection) as session:
                with connection.begin_nested():  # SAVEPOINT
                    yield session
                # SAVEPOINT rolled back automatically
            # Outer transaction rolled back automatically
```

### Factory Boy Integration

```python
# tests/factories.py
from factory.alchemy import SQLAlchemyModelFactory
import factory
from myapp.models import User, Post

class UserFactory(SQLAlchemyModelFactory):
    class Meta:
        model = User
        sqlalchemy_session_persistence = "flush"

    username = factory.Sequence(lambda n: f"user_{n}")
    email    = factory.LazyAttribute(lambda o: f"{o.username}@example.com")
    is_active = True

class PostFactory(SQLAlchemyModelFactory):
    class Meta:
        model = Post
        sqlalchemy_session_persistence = "flush"

    title   = factory.Faker("sentence", nb_words=5)
    body    = factory.Faker("paragraph")
    author  = factory.SubFactory(UserFactory)

# conftest.py — inject session into factory
@pytest.fixture(autouse=True)
def _set_factory_session(db_session):
    UserFactory._meta.sqlalchemy_session = db_session
    PostFactory._meta.sqlalchemy_session = db_session
    yield
    UserFactory._meta.sqlalchemy_session = None
    PostFactory._meta.sqlalchemy_session = None
```

---

## Alembic Migrations with Testcontainers

Run full Alembic migration stack in integration tests to catch migration issues early.

```python
# tests/conftest.py (Alembic variant)
import pytest
from alembic.config import Config
from alembic import command
from sqlalchemy import create_engine
from testcontainers.postgres import PostgresContainer


@pytest.fixture(scope="session")
def postgres_container():
    with PostgresContainer("postgres:16-alpine") as pg:
        yield pg


@pytest.fixture(scope="session")
def migrated_engine(postgres_container):
    """Run all migrations on a fresh database."""
    url = postgres_container.get_connection_url()
    engine = create_engine(url)

    alembic_cfg = Config("alembic.ini")
    alembic_cfg.set_main_option("sqlalchemy.url", url)
    command.upgrade(alembic_cfg, "head")

    yield engine

    command.downgrade(alembic_cfg, "base")
    engine.dispose()


@pytest.fixture
def db_session(migrated_engine):
    """Per-test session with rollback — on top of a fully migrated DB."""
    from sqlalchemy.orm import Session
    connection = migrated_engine.connect()
    transaction = connection.begin()
    session = Session(bind=connection, join_transaction_mode="create_savepoint")
    yield session
    session.close()
    transaction.rollback()
    connection.close()
```

**Verify migration integrity:**

```python
@pytest.mark.integration
def test_migration_consistency(migrated_engine):
    """Verify that ORM metadata matches the migrated schema."""
    from sqlalchemy import inspect
    from myapp.models import Base

    inspector = inspect(migrated_engine)
    for table_name in Base.metadata.tables:
        assert inspector.has_table(table_name), \
            f"Table {table_name!r} missing from migrated DB"
```

---

## Async SQLAlchemy (asyncpg)

```python
# tests/conftest.py — async pattern
import pytest
import pytest_asyncio
from sqlalchemy.ext.asyncio import (
    AsyncEngine,
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)
from testcontainers.postgres import PostgresContainer

from myapp.models import Base


@pytest.fixture(scope="session")
def postgres_container():
    with PostgresContainer("postgres:16-alpine") as pg:
        yield pg


@pytest.fixture(scope="session")
def async_db_url(postgres_container):
    """Convert psycopg2 URL to asyncpg URL."""
    return postgres_container.get_connection_url().replace(
        "postgresql+psycopg2://", "postgresql+asyncpg://"
    )


@pytest_asyncio.fixture(scope="session")
async def async_engine(async_db_url) -> AsyncEngine:
    engine = create_async_engine(async_db_url, echo=False)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine.dispose()


@pytest_asyncio.fixture
async def async_session(async_engine) -> AsyncSession:
    """Per-test async session with savepoint rollback."""
    async with async_engine.connect() as conn:
        await conn.begin()
        async with AsyncSession(
            bind=conn,
            join_transaction_mode="create_savepoint",
        ) as session:
            yield session
        await conn.rollback()
```

**Usage:**

```python
import pytest

@pytest.mark.anyio
async def test_async_create_user(async_session):
    from myapp.models import User
    user = User(username="async_alice", email="alice@example.com")
    async_session.add(user)
    await async_session.flush()
    assert user.id is not None
```

---

## Django with Testcontainers

```python
# tests/conftest.py
import pytest
from testcontainers.postgres import PostgresContainer


@pytest.fixture(scope="session")
def postgres_container():
    with PostgresContainer("postgres:16-alpine") as pg:
        yield pg


@pytest.fixture(scope="session")
def django_db_setup(postgres_container, django_db_blocker):
    """Override Django's test DB setup to use the testcontainers PostgreSQL."""
    from django.conf import settings
    settings.DATABASES["default"] = {
        "ENGINE":   "django.db.backends.postgresql",
        "NAME":     postgres_container.dbname,
        "USER":     postgres_container.username,
        "PASSWORD": postgres_container.password,
        "HOST":     postgres_container.get_container_host_ip(),
        "PORT":     postgres_container.get_exposed_port(5432),
    }
    with django_db_blocker.unblock():
        from django.core.management import call_command
        call_command("migrate", "--run-syncdb", verbosity=0)
```

---

## MongoDB Patterns

```python
# tests/conftest.py
import pytest
from pymongo import MongoClient
from testcontainers.mongodb import MongoDbContainer


@pytest.fixture(scope="session")
def mongo_container():
    with MongoDbContainer("mongo:7") as mongo:
        yield mongo


@pytest.fixture(scope="session")
def mongo_client(mongo_container):
    client = MongoClient(mongo_container.get_connection_url())
    yield client
    client.close()


@pytest.fixture
def mongo_db(mongo_client):
    """Per-test: use a fresh database name, drop after test."""
    import uuid
    db_name = f"test_{uuid.uuid4().hex}"
    db = mongo_client[db_name]
    yield db
    mongo_client.drop_database(db_name)


# Usage
@pytest.mark.integration
def test_insert_document(mongo_db):
    users = mongo_db["users"]
    result = users.insert_one({"name": "Alice", "age": 30})
    assert result.inserted_id is not None

    found = users.find_one({"name": "Alice"})
    assert found["age"] == 30
```

---

## Multiple Database Containers

```python
# tests/conftest.py
import pytest
from testcontainers.postgres import PostgresContainer
from testcontainers.redis import RedisContainer
from testcontainers.mongodb import MongoDbContainer


@pytest.fixture(scope="session")
def postgres_container():
    with PostgresContainer("postgres:16-alpine") as pg:
        yield pg


@pytest.fixture(scope="session")
def redis_container():
    with RedisContainer("redis:7-alpine") as redis:
        yield redis


@pytest.fixture(scope="session")
def mongo_container():
    with MongoDbContainer("mongo:7") as mongo:
        yield mongo


# Usage: combine as needed
@pytest.mark.integration
def test_multi_store(db_session, redis_client, mongo_db):
    # PostgreSQL: persistent data
    user = User(username="alice")
    db_session.add(user)
    db_session.flush()

    # Redis: cache
    redis_client.setex(f"user:{user.id}", 3600, "alice")

    # MongoDB: event log
    mongo_db["events"].insert_one({
        "user_id": str(user.id),
        "action": "created",
    })

    # Verify
    assert redis_client.get(f"user:{user.id}") == b"alice"
    assert mongo_db["events"].count_documents({"user_id": str(user.id)}) == 1
```
