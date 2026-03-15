# Python Integration Testing Patterns

Comprehensive guide to structuring, organizing, and running integration tests in Python projects using pytest.

**Official Documentation:**
- pytest: https://docs.pytest.org/en/stable/
- pytest-xdist: https://pytest-xdist.readthedocs.io/
- testcontainers-python: https://testcontainers-python.readthedocs.io/
- GitHub Actions: https://docs.github.com/en/actions

---

## Table of Contents

1. [Test Pyramid for Python](#test-pyramid-for-python)
2. [conftest.py Architecture](#conftestpy-architecture)
3. [pytest Markers for Integration Tests](#pytest-markers-for-integration-tests)
4. [GitHub Actions CI/CD](#github-actions-cicd)
5. [pytest-xdist Parallel Execution](#pytest-xdist-parallel-execution)
6. [Test Isolation Strategies](#test-isolation-strategies)

---

## Test Pyramid for Python

The test pyramid defines the recommended distribution of test types. Integration tests sit at the 20% middle tier — expensive to run but essential for verifying real component interactions (database, external APIs, message queues).

```
         /\
        /  \    E2E (10%)
       / E2E\   Playwright, Selenium, httpx against live app
      /------\
     /        \  Integration (20%)
    / Integr.  \ testcontainers, real DB, real Redis, real HTTP
   /------------\
  /              \ Unit (70%)
 /  Unit Tests    \ mocks, fakes, pure functions
/------------------\
```

**Practical implications for Python:**

| Level | Tool | Speed | Scope |
|-------|------|-------|-------|
| Unit | pytest + unittest.mock | < 1 ms/test | Single class/function |
| Integration | pytest + testcontainers | 100 ms–10 s/test | Service + DB, service + queue |
| E2E | pytest + httpx/playwright | 1–60 s/test | Full stack, running app |

**Rule of thumb:**
- Unit tests run on every save (< 5 s total suite)
- Integration tests run on every push (< 5 min)
- E2E tests run on PR to main (< 30 min)

### Project Layout

```
myproject/
├── src/
│   └── myapp/
│       ├── models.py
│       ├── services.py
│       └── api.py
├── tests/
│   ├── conftest.py            # Root: session-scoped containers, app factory
│   ├── unit/
│   │   ├── conftest.py        # Unit-specific fixtures (pure mocks)
│   │   ├── test_models.py
│   │   └── test_services.py
│   ├── integration/
│   │   ├── conftest.py        # Integration: DB session, Redis client
│   │   ├── test_repositories.py
│   │   ├── test_api.py
│   │   └── test_workers.py
│   └── e2e/
│       ├── conftest.py        # E2E: running app URL, browser fixture
│       └── test_checkout_flow.py
├── pyproject.toml
└── alembic.ini
```

---

## conftest.py Architecture

`conftest.py` is pytest's dependency injection mechanism. Fixtures defined there are automatically available to all tests in the same directory and subdirectories without explicit imports.

### Fixture Scopes

| Scope | Lifetime | Use when |
|-------|----------|----------|
| `session` | Entire test run | Starting containers, creating DB schema, loading fixtures that never change |
| `package` | All tests in a package | Per-package DB state that tests in the same package share |
| `module` | All tests in a file | Shared state per test file (e.g., a single service instance) |
| `class` | All methods in a `Test*` class | Class-level shared setup |
| `function` | Single test (default) | Database transactions, per-test clean state |

### Root conftest.py — Session-scoped Infrastructure

```python
# tests/conftest.py
import pytest
from testcontainers.postgres import PostgresContainer
from testcontainers.redis import RedisContainer
from sqlalchemy import create_engine, text
from sqlalchemy.orm import sessionmaker

from myapp.database import Base


@pytest.fixture(scope="session")
def postgres_container():
    """Start PostgreSQL once for the entire test session."""
    with PostgresContainer("postgres:16-alpine") as pg:
        yield pg


@pytest.fixture(scope="session")
def redis_container():
    """Start Redis once for the entire test session."""
    with RedisContainer("redis:7-alpine") as redis:
        yield redis


@pytest.fixture(scope="session")
def db_engine(postgres_container):
    """Create the SQLAlchemy engine and schema once per session."""
    engine = create_engine(
        postgres_container.get_connection_url(),
        # NullPool avoids connection pool issues with pytest-xdist
        poolclass=NullPool,
    )
    Base.metadata.create_all(engine)
    yield engine
    Base.metadata.drop_all(engine)
    engine.dispose()


@pytest.fixture(scope="session")
def redis_client(redis_container):
    """Session-scoped Redis client."""
    import redis as redis_lib
    client = redis_lib.Redis(
        host=redis_container.get_container_host_ip(),
        port=redis_container.get_exposed_port(6379),
    )
    yield client
    client.close()
```

### Integration conftest.py — Function-scoped DB Sessions

```python
# tests/integration/conftest.py
import pytest
from sqlalchemy.orm import Session

from myapp.database import Base


@pytest.fixture
def db_session(db_engine):
    """
    Provide a transactional database session that rolls back after each test.
    Uses the SQLAlchemy 2.0 savepoint pattern to allow nested transactions
    inside application code.
    """
    connection = db_engine.connect()
    transaction = connection.begin()

    session = Session(bind=connection, join_transaction_mode="create_savepoint")

    yield session

    session.close()
    transaction.rollback()
    connection.close()


@pytest.fixture
def app_client(db_session):
    """FastAPI test client with the test DB session injected."""
    from fastapi.testclient import TestClient
    from myapp.main import app
    from myapp.database import get_db

    def override_get_db():
        yield db_session

    app.dependency_overrides[get_db] = override_get_db
    with TestClient(app) as client:
        yield client
    app.dependency_overrides.clear()
```

### Module-scoped Example — Shared Service Instance

```python
# tests/integration/conftest.py (continued)

@pytest.fixture(scope="module")
def email_service(smtp_container):
    """One EmailService instance shared by all tests in this module."""
    from myapp.services.email import EmailService
    return EmailService(
        host=smtp_container.get_container_host_ip(),
        port=smtp_container.get_exposed_port(1025),
    )
```

### Fixture Dependency Chain

Fixtures can depend on fixtures of wider scope, but never narrower scope. The hierarchy is enforced by pytest:

```
session-scoped: postgres_container → db_engine
                                          ↓
function-scoped:                    db_session → app_client
```

A `session`-scoped fixture can depend on another `session` fixture. A `function`-scoped fixture can depend on `session`, `module`, `class`, or `function` fixtures.

---

## pytest Markers for Integration Tests

Markers classify tests so you can run targeted subsets. They are declared in `pytest.ini` or `pyproject.toml` to avoid `PytestUnknownMarkWarning`.

### Registering Markers

```toml
# pyproject.toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-v --tb=short"
markers = [
    "unit: fast unit tests with no I/O",
    "integration: tests requiring real infrastructure (DB, Redis, etc.)",
    "slow: tests taking longer than 5 seconds",
    "database: tests requiring a real database connection",
    "redis: tests requiring a real Redis connection",
    "kafka: tests requiring a real Kafka broker",
    "external: tests calling external HTTP APIs",
    "smoke: minimal smoke test subset for quick validation",
]
```

Equivalent `pytest.ini`:

```ini
[pytest]
testpaths = tests
addopts = -v --tb=short
markers =
    unit: fast unit tests with no I/O
    integration: tests requiring real infrastructure
    slow: tests taking longer than 5 seconds
    database: tests requiring a real database connection
    redis: tests requiring a real Redis connection
    kafka: tests requiring a real Kafka broker
    external: tests calling external HTTP APIs
    smoke: minimal smoke test subset
```

### Applying Markers

**Function level:**
```python
import pytest

@pytest.mark.integration
@pytest.mark.database
def test_user_repository_create(db_session):
    from myapp.repositories.user import UserRepository
    repo = UserRepository(db_session)
    user = repo.create(email="test@example.com", name="Alice")
    assert user.id is not None
    assert user.email == "test@example.com"
```

**Class level (applies to all methods):**
```python
@pytest.mark.integration
@pytest.mark.database
class TestOrderRepository:
    def test_create_order(self, db_session):
        ...

    def test_find_by_user(self, db_session):
        ...
```

**Module level (applies to entire file):**
```python
# tests/integration/test_repositories.py
import pytest

pytestmark = [pytest.mark.integration, pytest.mark.database]


def test_user_created(db_session):
    ...


def test_order_created(db_session):
    ...
```

### Running Subsets

```bash
# Run all integration tests
pytest -m integration

# Run integration but not slow ones
pytest -m "integration and not slow"

# Run database tests only
pytest -m database

# Run integration OR smoke
pytest -m "integration or smoke"

# Exclude external API tests (useful for offline development)
pytest -m "not external"

# Run only unit tests
pytest -m unit

# Run integration + database, excluding kafka
pytest -m "integration and database and not kafka"

# Combine with directory selection
pytest tests/integration/ -m "not slow"

# Dry-run: list collected tests without running
pytest -m integration --collect-only
```

### Automatic Marker Application via conftest.py

Automatically mark tests in specific directories without decorating each file:

```python
# tests/conftest.py
import pytest


def pytest_collection_modifyitems(config, items):
    """Automatically apply markers based on test location."""
    for item in items:
        # Mark by directory
        if "integration" in str(item.fspath):
            item.add_marker(pytest.mark.integration)
        if "e2e" in str(item.fspath):
            item.add_marker(pytest.mark.slow)
            item.add_marker(pytest.mark.integration)

        # Mark slow tests by keyword in name
        if "slow" in item.name.lower():
            item.add_marker(pytest.mark.slow)
```

### Skipping Integration Tests in Unit Mode

```python
# tests/conftest.py
import os
import pytest


def pytest_configure(config):
    """Register custom markers."""
    config.addinivalue_line(
        "markers", "integration: mark test as requiring real infrastructure"
    )


def pytest_runtest_setup(item):
    """Skip integration tests when SKIP_INTEGRATION env var is set."""
    if "integration" in item.keywords and os.getenv("SKIP_INTEGRATION"):
        pytest.skip("Integration tests disabled via SKIP_INTEGRATION")
```

---

## GitHub Actions CI/CD

### Service Containers vs Testcontainers

GitHub Actions offers two approaches to running dependent services:

**Service containers** — GitHub manages the container lifecycle via the `services:` block. Simple, zero-code setup. Containers start before the job and are torn down after.

**Testcontainers** — Your test code manages containers via Docker-in-Docker. More flexible (per-test control), but requires Docker access in the runner.

| Aspect | Service Containers | Testcontainers |
|--------|-------------------|----------------|
| Setup | YAML only, no code change | Python code in conftest.py |
| Per-test isolation | No (shared for job) | Yes (can start/stop per test) |
| Multiple DBs | One per service block | Unlimited |
| Dynamic config | Fixed env vars | `get_connection_url()` |
| Container version | Fixed in YAML | Parameterizable at runtime |
| Startup detection | `health check` | `wait_for_logs` / port poll |

### Testcontainers in GitHub Actions

GitHub Actions runners run as root with Docker available. Key environment variable considerations:

```yaml
# .github/workflows/integration.yml
name: Integration Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  integration-tests:
    runs-on: ubuntu-latest

    env:
      # Disable Ryuk (reaper container) — avoids permission issues on some runners
      TESTCONTAINERS_RYUK_DISABLED: "true"
      # Override Docker host for testcontainers on GitHub Actions
      # Not usually needed on ubuntu-latest but required in some DinD setups
      # DOCKER_HOST: "unix:///var/run/docker.sock"

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Cache pip dependencies
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements*.txt', '**/pyproject.toml') }}
          restore-keys: |
            ${{ runner.os }}-pip-

      - name: Cache Docker images
        uses: actions/cache@v4
        with:
          path: /tmp/docker-cache
          key: docker-${{ hashFiles('.github/docker-image-list.txt') }}
          restore-keys: docker-

      - name: Restore Docker image cache
        run: |
          if [ -d /tmp/docker-cache ]; then
            for f in /tmp/docker-cache/*.tar; do
              [ -f "$f" ] && docker load -i "$f"
            done
          fi

      - name: Install dependencies
        run: |
          pip install -e ".[test]"

      - name: Run unit tests
        run: pytest tests/unit/ -m unit --tb=short -q

      - name: Run integration tests
        run: |
          pytest tests/integration/ \
            -m "integration and not slow" \
            --tb=short \
            -v \
            --junitxml=reports/integration-junit.xml \
            --cov=src/myapp \
            --cov-report=xml:reports/coverage.xml

      - name: Save Docker image cache
        if: always()
        run: |
          mkdir -p /tmp/docker-cache
          docker save postgres:16-alpine | gzip > /tmp/docker-cache/postgres.tar
          docker save redis:7-alpine | gzip > /tmp/docker-cache/redis.tar

      - name: Upload test reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-reports
          path: reports/

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          file: reports/coverage.xml
```

### Service Containers Approach (Alternative)

```yaml
# .github/workflows/integration-services.yml
name: Integration Tests (Service Containers)

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpassword
          POSTGRES_DB: testdb
        options: >-
          --health-cmd "pg_isready -U testuser"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 5s
          --health-timeout 3s
          --health-retries 5
        ports:
          - 6379:6379

    env:
      DATABASE_URL: "postgresql://testuser:testpassword@localhost:5432/testdb"
      REDIS_URL: "redis://localhost:6379"

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -e ".[test]"
      - name: Run migrations
        run: alembic upgrade head
      - name: Run integration tests
        run: pytest tests/integration/ -v --tb=short
```

### Docker-in-Docker for Self-Hosted Runners

When running on self-hosted runners or Kubernetes pods, Docker-in-Docker may be needed:

```yaml
  test-k8s:
    runs-on: self-hosted
    container:
      image: python:3.12-slim
      options: --privileged  # Required for DinD

    env:
      DOCKER_HOST: "tcp://docker:2376"
      DOCKER_TLS_VERIFY: "1"
      DOCKER_CERT_PATH: "/certs/client"
      # Tell testcontainers which hostname to use for exposed ports
      TESTCONTAINERS_HOST_OVERRIDE: "docker"
      TESTCONTAINERS_RYUK_DISABLED: "true"

    services:
      docker:
        image: docker:24-dind
        options: --privileged
        env:
          DOCKER_TLS_CERTDIR: "/certs"
        volumes:
          - docker-certs-ca:/certs/ca
          - docker-certs-client:/certs/client

    steps:
      - uses: actions/checkout@v4
      - run: pip install -e ".[test]"
      - run: pytest tests/integration/ -v
```

### Matrix Strategy — Multiple Python/DB Versions

```yaml
  integration-matrix:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        python-version: ["3.11", "3.12", "3.13"]
        postgres-version: ["14", "15", "16"]

    env:
      POSTGRES_IMAGE: postgres:${{ matrix.postgres-version }}-alpine
      TESTCONTAINERS_RYUK_DISABLED: "true"

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: pip install -e ".[test]"
      - run: pytest tests/integration/ -v
        env:
          POSTGRES_IMAGE: postgres:${{ matrix.postgres-version }}-alpine
```

---

## pytest-xdist Parallel Execution

`pytest-xdist` distributes test execution across multiple worker processes, dramatically reducing total test time.

```bash
pip install pytest-xdist
```

### Basic Usage

```bash
# Use all available CPUs (auto-detect)
pytest -n auto

# Use exactly 4 workers
pytest -n 4

# Use half available CPUs
pytest -n logical

# Disable parallelism (useful for debugging)
pytest -n 0

# Combine with markers
pytest -m integration -n auto

# With verbose output per worker
pytest -n auto -v
```

### Distribution Modes

```bash
# loadscope: group tests by module (default with -n auto in some versions)
# All tests in one file run on the same worker → safe for module-scoped fixtures
pytest -n auto --dist loadscope

# loadfile: group by file path (similar to loadscope)
pytest -n auto --dist loadfile

# load: distribute as tasks complete (fastest, least isolation-friendly)
pytest -n auto --dist load

# no: all tests on one worker (effectively -n 1)
pytest --dist no
```

**Recommendation:** Use `--dist loadscope` when you have module-scoped fixtures that maintain state. Each file's tests run on a single worker, so module fixtures are created and destroyed predictably.

### Database Isolation with xdist — Per-Worker Databases

The core challenge with xdist and databases: multiple workers may write to the same tables simultaneously, causing test interference. The solution is to give each worker its own database.

```python
# tests/conftest.py
import pytest
from sqlalchemy import create_engine, text
from sqlalchemy.orm import sessionmaker
from sqlalchemy.pool import NullPool

from myapp.database import Base


@pytest.fixture(scope="session")
def worker_id(request):
    """Return the xdist worker ID, or 'master' if not running in parallel."""
    if hasattr(request.config, "workerinput"):
        return request.config.workerinput["workerid"]
    return "master"


@pytest.fixture(scope="session")
def database_name(worker_id):
    """Each xdist worker gets its own isolated database."""
    return f"testdb_{worker_id}"


@pytest.fixture(scope="session")
def db_engine(postgres_container, database_name):
    """
    Create a per-worker database so parallel test runs don't interfere.
    Uses the 'postgres' maintenance database to CREATE/DROP.
    """
    admin_url = postgres_container.get_connection_url().replace(
        "/test", "/postgres"
    )
    admin_engine = create_engine(admin_url, isolation_level="AUTOCOMMIT", poolclass=NullPool)

    with admin_engine.connect() as conn:
        conn.execute(text(f'CREATE DATABASE "{database_name}"'))

    worker_url = postgres_container.get_connection_url().replace(
        "/test", f"/{database_name}"
    )
    engine = create_engine(worker_url, poolclass=NullPool)
    Base.metadata.create_all(engine)

    yield engine

    engine.dispose()

    with admin_engine.connect() as conn:
        conn.execute(
            text(f'DROP DATABASE IF EXISTS "{database_name}" WITH (FORCE)')
        )
    admin_engine.dispose()


@pytest.fixture
def db_session(db_engine):
    """Function-scoped session with savepoint rollback."""
    connection = db_engine.connect()
    transaction = connection.begin()
    session_factory = sessionmaker(bind=connection)
    session = session_factory()
    session.begin_nested()  # Create savepoint

    yield session

    session.close()
    transaction.rollback()
    connection.close()
```

### Testcontainers + xdist — Session-Scoped Container Sharing

When using `scope="session"` with xdist, each worker spawns its own session and would create its own container — causing N containers for N workers. Use `testcontainers`'s built-in `DockerContainer` locking or start containers outside of pytest via fixtures scoped to the worker:

**Option A: One container per worker (simple, wasteful)**

The default behavior: each worker starts its own container. Works but uses more resources.

```python
@pytest.fixture(scope="session")
def postgres_container():
    """Each xdist worker gets its own container."""
    with PostgresContainer("postgres:16-alpine") as pg:
        yield pg
```

**Option B: Shared container via `tmp_path_factory` locking (xdist-safe)**

```python
# tests/conftest.py
import json
import pytest
from filelock import FileLock
from testcontainers.postgres import PostgresContainer


@pytest.fixture(scope="session")
def postgres_container(tmp_path_factory, worker_id):
    """
    Share a single Postgres container across all xdist workers.
    Uses filelock so only the first worker starts it.
    """
    root_tmp = tmp_path_factory.getbasetemp().parent  # Shared across workers

    if worker_id == "master":
        # Not running xdist — just start container normally
        with PostgresContainer("postgres:16-alpine") as pg:
            yield pg
        return

    lock_path = root_tmp / "postgres_container.lock"
    info_path = root_tmp / "postgres_container.json"

    with FileLock(str(lock_path)):
        if not info_path.exists():
            # First worker: start container and write connection info
            container = PostgresContainer("postgres:16-alpine")
            container.start()
            info = {
                "url": container.get_connection_url(),
                "host": container.get_container_host_ip(),
                "port": container.get_exposed_port(5432),
            }
            info_path.write_text(json.dumps(info))
        else:
            container = None
            info = json.loads(info_path.read_text())

    yield info  # Workers receive connection info dict, not container object

    # Last worker cleans up (approximation — use a counter for precise cleanup)
    if container is not None:
        container.stop()
```

**Option C: Use `pytest-xdist-filelock` plugin** (third-party)

A cleaner solution using `pytest-xdist`'s `xdist_group` marker:

```python
# Mark tests to run on the same worker
@pytest.mark.xdist_group("database")
def test_that_needs_shared_db():
    ...
```

### `--dist loadscope` for Fixture Safety

```bash
# All tests in the same module run on the same xdist worker
# This makes module-scoped fixtures behave correctly
pytest -n auto --dist loadscope tests/integration/

# Output shows worker assignment:
# [gw0] PASSED tests/integration/test_users.py::test_create
# [gw0] PASSED tests/integration/test_users.py::test_delete
# [gw1] PASSED tests/integration/test_orders.py::test_create
```

### xdist Configuration in pyproject.toml

```toml
[tool.pytest.ini_options]
addopts = [
    "-n", "auto",
    "--dist", "loadscope",
    "-v",
    "--tb", "short",
]
```

---

## Test Isolation Strategies

### SQLAlchemy Savepoint Rollback Pattern

The gold standard for database integration tests: wrap each test in a savepoint (nested transaction), roll back after. The application code can commit its own transactions (they become savepoints within your outer transaction).

```python
# tests/integration/conftest.py
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import Session
from sqlalchemy.pool import NullPool

from myapp.database import Base, engine as app_engine


@pytest.fixture(scope="session")
def test_engine():
    """Create a test engine once. Use NullPool to avoid cross-test state."""
    engine = create_engine(
        "postgresql://user:password@localhost/testdb",
        poolclass=NullPool,
    )
    Base.metadata.create_all(engine)
    yield engine
    Base.metadata.drop_all(engine)
    engine.dispose()


@pytest.fixture
def db_session(test_engine):
    """
    Savepoint-based session rollback.

    Flow:
      1. Open connection
      2. Begin outer transaction (never committed)
      3. Create Session bound to this connection
      4. Set join_transaction_mode="create_savepoint" so that any
         session.commit() inside application code creates a SAVEPOINT
         instead of a real COMMIT
      5. After test: rollback the outer transaction → all changes vanish
    """
    connection = test_engine.connect()
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

**Why `join_transaction_mode="create_savepoint"`?**

Without it, calling `session.commit()` in application code would commit to the outer transaction, making rollback ineffective. With `create_savepoint`, each `session.commit()` becomes `SAVEPOINT`, and the outer `ROLLBACK` erases everything.

### Django: TestCase vs `@pytest.mark.django_db(transaction=True)`

```python
# Using pytest-django

# Option 1: pytest.mark.django_db (default — uses TestCase wrapping)
# Each test runs in a transaction that is rolled back.
# Fast, but cannot test code that uses transaction.atomic() with on_commit hooks.
@pytest.mark.django_db
def test_user_created(db):
    from myapp.models import User
    user = User.objects.create(username="alice", email="alice@example.com")
    assert user.pk is not None


# Option 2: transaction=True — real transactions, manual cleanup
# Slower (truncates tables between tests).
# Required when testing signals, on_commit hooks, or select_for_update().
@pytest.mark.django_db(transaction=True)
def test_on_commit_hook_fires(db):
    from myapp.models import User
    from myapp.signals import post_save_handler

    with patch.object(post_save_handler, 'send') as mock_send:
        user = User.objects.create(username="bob")
        # on_commit only fires after a real COMMIT
        assert mock_send.called


# Option 3: Use Django TestCase class (equivalent to mark.django_db)
from django.test import TestCase

class UserTests(TestCase):
    def test_user_creation(self):
        from myapp.models import User
        user = User.objects.create(username="charlie")
        self.assertIsNotNone(user.pk)
```

### Truncation Strategy

For tests that need multiple committed transactions (e.g., testing aggregation queries), the savepoint pattern won't work. Use table truncation after each test instead:

```python
# tests/integration/conftest.py
import pytest
from sqlalchemy import text, inspect


@pytest.fixture(autouse=True)
def truncate_tables(db_engine):
    """Truncate all tables after each test (slower but complete isolation)."""
    yield

    with db_engine.connect() as conn:
        # Disable constraints during truncation
        conn.execute(text("SET session_replication_role = replica"))

        inspector = inspect(db_engine)
        table_names = inspector.get_table_names(schema="public")
        # Exclude alembic version table
        table_names = [t for t in table_names if t != "alembic_version"]

        for table_name in table_names:
            conn.execute(text(f'TRUNCATE TABLE "{table_name}" CASCADE'))

        conn.execute(text("SET session_replication_role = DEFAULT"))
        conn.commit()


# For SQLite (no session_replication_role):
@pytest.fixture(autouse=True)
def truncate_tables_sqlite(db_engine):
    yield
    with db_engine.connect() as conn:
        inspector = inspect(db_engine)
        conn.execute(text("PRAGMA foreign_keys = OFF"))
        for table_name in inspector.get_table_names():
            conn.execute(text(f"DELETE FROM {table_name}"))
        conn.execute(text("PRAGMA foreign_keys = ON"))
        conn.commit()
```

### pytest-factoryboy with xdist

`factory_boy` creates test data. With `pytest-factoryboy`, factories become fixtures. The challenge with xdist is injecting the correct per-worker session.

```python
# tests/factories.py
import factory
from factory.alchemy import SQLAlchemyModelFactory
from myapp.models import User, Order


class UserFactory(SQLAlchemyModelFactory):
    class Meta:
        model = User
        sqlalchemy_session = None  # Injected via conftest

    username = factory.Sequence(lambda n: f"user_{n}")
    email = factory.LazyAttribute(lambda o: f"{o.username}@example.com")
    is_active = True


class OrderFactory(SQLAlchemyModelFactory):
    class Meta:
        model = Order
        sqlalchemy_session = None

    user = factory.SubFactory(UserFactory)
    total = factory.Faker("pydecimal", left_digits=4, right_digits=2, positive=True)
    status = "pending"
```

```python
# tests/integration/conftest.py
import pytest
from pytest_factoryboy import register
from tests.factories import UserFactory, OrderFactory

# Register factories as pytest fixtures
register(UserFactory)
register(OrderFactory)


@pytest.fixture(autouse=True)
def inject_session_into_factories(db_session):
    """
    Inject the function-scoped db_session into all factories.
    This must run before any factory fixture is used.
    """
    UserFactory._meta.sqlalchemy_session = db_session
    OrderFactory._meta.sqlalchemy_session = db_session
    yield
    # Clean up to avoid session leaking between tests
    UserFactory._meta.sqlalchemy_session = None
    OrderFactory._meta.sqlalchemy_session = None
```

```python
# tests/integration/test_orders.py
import pytest

pytestmark = [pytest.mark.integration, pytest.mark.database]


def test_order_belongs_to_user(user_factory, order_factory, db_session):
    """user_factory and order_factory are auto-registered pytest fixtures."""
    user = user_factory()
    order = order_factory(user=user)

    db_session.flush()  # Assign PKs without committing

    assert order.user_id == user.id
    assert order.status == "pending"


def test_multiple_orders(user_factory, order_factory, db_session):
    user = user_factory()
    orders = order_factory.create_batch(5, user=user)

    db_session.flush()

    result = db_session.query(Order).filter_by(user_id=user.id).count()
    assert result == 5
```

### Complete Reference — Isolation Strategy Selection

```
Need to test...                        → Use
─────────────────────────────────────────────────────
SELECT / INSERT / UPDATE queries       → savepoint rollback
Django on_commit / transaction signals → @django_db(transaction=True)
Aggregations with multiple commits     → truncation
Code that uses TRUNCATE internally     → per-test database (xdist worker_id)
High parallelism (xdist)               → per-worker database OR savepoint+xdist
Data that must persist across tests    → module-scoped fixture + explicit cleanup
```
