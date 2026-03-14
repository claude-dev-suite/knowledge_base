# Pytest Advanced Patterns

Advanced testing patterns, mocking strategies, and integration testing with pytest.

**Official Documentation:** https://docs.pytest.org/en/stable/

---

## Table of Contents

1. [Mocking and Patching](#mocking-and-patching)
2. [Parametrization Patterns](#parametrization-patterns)
3. [Async Testing](#async-testing)
4. [Integration Testing](#integration-testing)
5. [Test Organization](#test-organization)
6. [Custom Markers](#custom-markers)
7. [Plugins and Hooks](#plugins-and-hooks)
8. [Performance and Profiling](#performance-and-profiling)
9. [Configuration](#configuration)

---

## Mocking and Patching

### Basic Mocking with unittest.mock

```python
from unittest.mock import Mock, MagicMock, patch, AsyncMock

# Basic mock
def test_mock_basic():
    mock = Mock()
    mock.method.return_value = "result"

    assert mock.method() == "result"
    mock.method.assert_called_once()


# MagicMock for magic methods
def test_magic_mock():
    mock = MagicMock()
    mock.__len__.return_value = 5

    assert len(mock) == 5


# Mock with spec
def test_mock_with_spec():
    class MyClass:
        def method(self, x: int) -> str:
            return str(x)

    mock = Mock(spec=MyClass)
    mock.method.return_value = "mocked"

    assert mock.method(42) == "mocked"
    # mock.nonexistent_method()  # Would raise AttributeError
```

### Patching

```python
# Patch decorator
@patch("myapp.api.requests.get")
def test_api_call(mock_get):
    mock_get.return_value.json.return_value = {"data": "test"}

    result = api.fetch_data()

    assert result == {"data": "test"}
    mock_get.assert_called_once_with("https://api.example.com/data")


# Patch context manager
def test_patch_context():
    with patch("myapp.config.get_setting") as mock_setting:
        mock_setting.return_value = "test_value"
        result = app.process()

    assert result == "processed: test_value"


# Patch object attribute
def test_patch_object():
    with patch.object(MyClass, "method", return_value="mocked"):
        obj = MyClass()
        assert obj.method() == "mocked"


# Patch multiple targets
@patch("myapp.db.save")
@patch("myapp.cache.invalidate")
def test_multiple_patches(mock_invalidate, mock_save):
    # Note: decorators are applied bottom-up
    mock_save.return_value = True

    result = service.update(data)

    mock_save.assert_called_once()
    mock_invalidate.assert_called_once()
```

### Patching Best Practices

```python
# Patch where it's used, not where it's defined
# myapp/service.py
from myapp.utils import helper

def process():
    return helper()

# test_service.py
# CORRECT: patch where helper is used
@patch("myapp.service.helper")
def test_process(mock_helper):
    mock_helper.return_value = "mocked"
    assert process() == "mocked"

# WRONG: patching where helper is defined
@patch("myapp.utils.helper")  # Won't work as expected
def test_process_wrong(mock_helper):
    pass


# Patch with side_effect for complex behavior
def test_side_effect():
    with patch("myapp.api.fetch") as mock_fetch:
        # Return different values on successive calls
        mock_fetch.side_effect = [
            {"id": 1},
            {"id": 2},
            Exception("API Error"),
        ]

        assert api.fetch() == {"id": 1}
        assert api.fetch() == {"id": 2}
        with pytest.raises(Exception, match="API Error"):
            api.fetch()


# Side effect as function
def test_side_effect_function():
    def dynamic_response(url):
        if "users" in url:
            return {"users": []}
        return {"data": []}

    with patch("requests.get") as mock_get:
        mock_get.return_value.json.side_effect = dynamic_response
```

### pytest-mock Plugin

```python
# pip install pytest-mock

def test_with_mocker(mocker):
    """mocker fixture provides convenient patching"""
    mock_fetch = mocker.patch("myapp.api.fetch")
    mock_fetch.return_value = {"data": "test"}

    result = service.process()

    assert result == "processed"
    mock_fetch.assert_called_once()


def test_spy(mocker):
    """Spy on real implementation"""
    spy = mocker.spy(mymodule, "real_function")

    result = mymodule.real_function(42)

    spy.assert_called_once_with(42)
    assert result == real_result  # Original implementation ran


def test_mock_property(mocker):
    """Mock a property"""
    mocker.patch.object(
        MyClass,
        "property_name",
        new_callable=mocker.PropertyMock,
        return_value="mocked_value"
    )

    obj = MyClass()
    assert obj.property_name == "mocked_value"
```

### Mocking datetime

```python
from datetime import datetime
from unittest.mock import patch


def test_mock_datetime():
    with patch("myapp.service.datetime") as mock_datetime:
        mock_datetime.now.return_value = datetime(2024, 1, 1, 12, 0, 0)
        mock_datetime.side_effect = lambda *args, **kwargs: datetime(*args, **kwargs)

        result = service.get_timestamp()

        assert result == "2024-01-01 12:00:00"


# Using freezegun (recommended)
# pip install freezegun
from freezegun import freeze_time


@freeze_time("2024-01-01 12:00:00")
def test_frozen_time():
    assert datetime.now() == datetime(2024, 1, 1, 12, 0, 0)


@freeze_time("2024-01-01")
class TestWithFrozenTime:
    def test_one(self):
        assert datetime.now().year == 2024

    def test_two(self):
        assert datetime.now().month == 1
```

---

## Parametrization Patterns

### Basic Parametrization

```python
@pytest.mark.parametrize("input,expected", [
    (1, 2),
    (2, 4),
    (3, 6),
])
def test_double(input, expected):
    assert double(input) == expected


# With IDs for better output
@pytest.mark.parametrize("input,expected", [
    pytest.param(1, 2, id="one"),
    pytest.param(2, 4, id="two"),
    pytest.param(0, 0, id="zero"),
])
def test_double_with_ids(input, expected):
    assert double(input) == expected
```

### Multiple Parameters

```python
# Cartesian product of parameters
@pytest.mark.parametrize("x", [1, 2])
@pytest.mark.parametrize("y", [10, 20])
def test_multiply(x, y):
    # Runs 4 times: (1,10), (1,20), (2,10), (2,20)
    result = x * y
    assert result in [10, 20, 20, 40]


# Combined parameters
@pytest.mark.parametrize("x,y,expected", [
    (1, 10, 10),
    (2, 20, 40),
])
def test_multiply_combined(x, y, expected):
    assert x * y == expected
```

### Conditional Skip

```python
@pytest.mark.parametrize("db_type", [
    "postgresql",
    pytest.param("mysql", marks=pytest.mark.skipif(
        not MYSQL_AVAILABLE,
        reason="MySQL not available"
    )),
    pytest.param("oracle", marks=pytest.mark.skip(reason="Not implemented")),
])
def test_database(db_type):
    db = create_database(db_type)
    assert db.ping()
```

### Parametrize with Fixtures

```python
# indirect=True passes param to fixture
@pytest.fixture
def database(request):
    db_type = request.param
    db = create_database(db_type)
    yield db
    db.cleanup()


@pytest.mark.parametrize("database", ["postgresql", "mysql"], indirect=True)
def test_insert(database):
    database.insert({"id": 1})
    assert database.count() == 1
```

### Class Parametrization

```python
@pytest.mark.parametrize("config", [
    {"env": "dev"},
    {"env": "prod"},
])
class TestWithConfig:
    def test_one(self, config):
        assert config["env"] in ["dev", "prod"]

    def test_two(self, config):
        app = create_app(config)
        assert app.config["env"] == config["env"]
```

### Dynamic Parametrization

```python
def generate_test_cases():
    """Generate test cases dynamically"""
    cases = []
    for i in range(5):
        cases.append(pytest.param(i, i * 2, id=f"case_{i}"))
    return cases


@pytest.mark.parametrize("input,expected", generate_test_cases())
def test_dynamic(input, expected):
    assert input * 2 == expected


# From file
def load_test_cases():
    import json
    with open("test_cases.json") as f:
        return json.load(f)


@pytest.mark.parametrize("case", load_test_cases())
def test_from_file(case):
    assert process(case["input"]) == case["expected"]
```

---

## Async Testing

### pytest-asyncio

```python
# pip install pytest-asyncio
import pytest
import asyncio


@pytest.mark.asyncio
async def test_async_function():
    result = await async_fetch_data()
    assert result == {"data": "test"}


@pytest.mark.asyncio
async def test_async_with_timeout():
    with pytest.raises(asyncio.TimeoutError):
        await asyncio.wait_for(slow_operation(), timeout=1.0)


# Async fixture
@pytest.fixture
async def async_client():
    client = await create_async_client()
    yield client
    await client.close()


@pytest.mark.asyncio
async def test_with_async_fixture(async_client):
    result = await async_client.get("/data")
    assert result.status == 200
```

### Testing Async Context Managers

```python
@pytest.mark.asyncio
async def test_async_context_manager():
    async with AsyncSession() as session:
        result = await session.execute("SELECT 1")
        assert result is not None


@pytest.mark.asyncio
async def test_async_iterator():
    results = []
    async for item in async_generator():
        results.append(item)

    assert len(results) == 10
```

### Mocking Async Functions

```python
from unittest.mock import AsyncMock


@pytest.mark.asyncio
async def test_mock_async():
    mock_fetch = AsyncMock(return_value={"data": "test"})

    result = await mock_fetch()

    assert result == {"data": "test"}
    mock_fetch.assert_awaited_once()


@pytest.mark.asyncio
async def test_patch_async(mocker):
    mock_api = mocker.patch("myapp.api.fetch", new_callable=AsyncMock)
    mock_api.return_value = {"status": "ok"}

    result = await service.process()

    mock_api.assert_awaited_once()
    assert result["status"] == "ok"
```

### Async Fixture Scopes

```python
@pytest.fixture(scope="session")
async def database():
    """Session-scoped async fixture"""
    db = await create_database()
    yield db
    await db.close()


@pytest.fixture
async def transaction(database):
    """Function-scoped with session-scoped dependency"""
    async with database.transaction() as tx:
        yield tx
        await tx.rollback()
```

---

## Integration Testing

### Database Integration

```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker


@pytest.fixture(scope="session")
def engine():
    """Create test database engine"""
    return create_engine("postgresql://test:test@localhost/testdb")


@pytest.fixture(scope="session")
def tables(engine):
    """Create tables once per session"""
    Base.metadata.create_all(engine)
    yield
    Base.metadata.drop_all(engine)


@pytest.fixture
def db_session(engine, tables):
    """Provide transactional database session"""
    connection = engine.connect()
    transaction = connection.begin()
    Session = sessionmaker(bind=connection)
    session = Session()

    yield session

    session.close()
    transaction.rollback()
    connection.close()


def test_create_user(db_session):
    user = User(name="Test", email="test@example.com")
    db_session.add(user)
    db_session.commit()

    assert db_session.query(User).count() == 1
```

### API Integration with httpx/requests-mock

```python
# Using responses library
# pip install responses
import responses


@responses.activate
def test_api_integration():
    responses.add(
        responses.GET,
        "https://api.example.com/users",
        json={"users": [{"id": 1, "name": "John"}]},
        status=200,
    )

    result = api.get_users()

    assert len(result["users"]) == 1
    assert responses.calls[0].request.url == "https://api.example.com/users"


# Using httpx with respx
# pip install respx
import httpx
import respx


@respx.mock
@pytest.mark.asyncio
async def test_async_api():
    respx.get("https://api.example.com/data").mock(
        return_value=httpx.Response(200, json={"data": "test"})
    )

    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/data")

    assert response.json() == {"data": "test"}
```

### Docker Integration with testcontainers

```python
# pip install testcontainers
from testcontainers.postgres import PostgresContainer
from testcontainers.redis import RedisContainer


@pytest.fixture(scope="session")
def postgres_container():
    with PostgresContainer("postgres:15") as postgres:
        yield postgres


@pytest.fixture
def db_url(postgres_container):
    return postgres_container.get_connection_url()


def test_with_real_postgres(db_url):
    engine = create_engine(db_url)
    with engine.connect() as conn:
        result = conn.execute(text("SELECT 1"))
        assert result.scalar() == 1


@pytest.fixture(scope="session")
def redis_container():
    with RedisContainer() as redis:
        yield redis


def test_with_real_redis(redis_container):
    import redis as redis_client
    client = redis_client.Redis(
        host=redis_container.get_container_host_ip(),
        port=redis_container.get_exposed_port(6379),
    )
    client.set("key", "value")
    assert client.get("key") == b"value"
```

### FastAPI Testing

```python
from fastapi.testclient import TestClient
from httpx import AsyncClient
import pytest


@pytest.fixture
def client():
    return TestClient(app)


def test_read_root(client):
    response = client.get("/")
    assert response.status_code == 200
    assert response.json() == {"message": "Hello World"}


def test_create_item(client):
    response = client.post(
        "/items/",
        json={"name": "Test Item", "price": 10.5},
    )
    assert response.status_code == 201
    assert response.json()["name"] == "Test Item"


# Async testing
@pytest.fixture
async def async_client():
    async with AsyncClient(app=app, base_url="http://test") as client:
        yield client


@pytest.mark.asyncio
async def test_async_endpoint(async_client):
    response = await async_client.get("/async-data")
    assert response.status_code == 200
```

---

## Test Organization

### Test Structure

```
tests/
├── conftest.py              # Shared fixtures
├── unit/
│   ├── conftest.py          # Unit test specific fixtures
│   ├── test_models.py
│   └── test_services.py
├── integration/
│   ├── conftest.py
│   ├── test_api.py
│   └── test_database.py
├── e2e/
│   ├── conftest.py
│   └── test_flows.py
└── fixtures/
    ├── __init__.py
    └── factories.py         # Test data factories
```

### Test Factories with factory_boy

```python
# pip install factory_boy
import factory
from factory.alchemy import SQLAlchemyModelFactory


class UserFactory(SQLAlchemyModelFactory):
    class Meta:
        model = User
        sqlalchemy_session_persistence = "commit"

    name = factory.Faker("name")
    email = factory.LazyAttribute(lambda o: f"{o.name.lower().replace(' ', '.')}@example.com")
    is_active = True


class PostFactory(SQLAlchemyModelFactory):
    class Meta:
        model = Post

    title = factory.Faker("sentence")
    content = factory.Faker("paragraph")
    author = factory.SubFactory(UserFactory)


# Usage in tests
def test_user_factory(db_session):
    UserFactory._meta.sqlalchemy_session = db_session

    user = UserFactory()
    assert user.id is not None
    assert "@" in user.email


def test_with_traits(db_session):
    class UserFactory(SQLAlchemyModelFactory):
        class Meta:
            model = User

        name = factory.Faker("name")

        class Params:
            admin = factory.Trait(
                is_admin=True,
                role="admin",
            )

    admin = UserFactory(admin=True)
    assert admin.is_admin is True
```

### Test Classes Organization

```python
class TestUserService:
    """Group related tests"""

    @pytest.fixture(autouse=True)
    def setup(self, db_session):
        """Setup for all tests in class"""
        self.service = UserService(db_session)

    def test_create_user(self):
        user = self.service.create(name="Test")
        assert user.name == "Test"

    def test_get_user(self):
        user = self.service.create(name="Test")
        found = self.service.get(user.id)
        assert found.id == user.id

    class TestValidation:
        """Nested class for validation tests"""

        def test_invalid_email(self):
            with pytest.raises(ValidationError):
                self.service.create(email="invalid")

        def test_duplicate_email(self):
            self.service.create(email="test@example.com")
            with pytest.raises(DuplicateError):
                self.service.create(email="test@example.com")
```

---

## Custom Markers

### Defining Markers

```python
# conftest.py
import pytest


def pytest_configure(config):
    config.addinivalue_line("markers", "slow: marks tests as slow")
    config.addinivalue_line("markers", "integration: marks as integration test")
    config.addinivalue_line("markers", "e2e: marks as end-to-end test")


# Or in pytest.ini
# [pytest]
# markers =
#     slow: marks tests as slow
#     integration: marks as integration test
```

### Using Markers

```python
@pytest.mark.slow
def test_slow_operation():
    time.sleep(10)
    assert True


@pytest.mark.integration
def test_database_integration(db):
    assert db.ping()


@pytest.mark.skipif(sys.platform == "win32", reason="Unix only")
def test_unix_specific():
    pass


@pytest.mark.xfail(reason="Known bug #123")
def test_known_failure():
    assert buggy_function() == expected


# Custom marker with parameters
@pytest.mark.timeout(10)
def test_with_timeout():
    pass
```

### Running by Markers

```bash
# Run only slow tests
pytest -m slow

# Run non-slow tests
pytest -m "not slow"

# Combine markers
pytest -m "integration and not slow"

# Run all except e2e
pytest -m "not e2e"
```

### Custom Marker with Behavior

```python
# conftest.py
import pytest


def pytest_collection_modifyitems(config, items):
    """Add skip marker to slow tests unless --slow is passed"""
    if config.getoption("--slow"):
        return

    skip_slow = pytest.mark.skip(reason="need --slow option to run")
    for item in items:
        if "slow" in item.keywords:
            item.add_marker(skip_slow)


def pytest_addoption(parser):
    parser.addoption(
        "--slow",
        action="store_true",
        default=False,
        help="run slow tests",
    )
```

---

## Plugins and Hooks

### Common Plugins

```python
# pytest.ini or pyproject.toml
# [tool.pytest.ini_options]
# addopts = "-v --tb=short"

# Useful plugins:
# pytest-cov: Coverage reporting
# pytest-xdist: Parallel test execution
# pytest-mock: Convenient mocking
# pytest-asyncio: Async test support
# pytest-timeout: Test timeouts
# pytest-randomly: Randomize test order
# pytest-sugar: Better output
# pytest-env: Set environment variables
```

### Writing Custom Plugin

```python
# conftest.py or separate plugin file
import pytest
import time


class TestTimer:
    def __init__(self):
        self.start_times = {}
        self.durations = {}

    def pytest_runtest_setup(self, item):
        self.start_times[item.nodeid] = time.time()

    def pytest_runtest_teardown(self, item):
        duration = time.time() - self.start_times[item.nodeid]
        self.durations[item.nodeid] = duration

    def pytest_sessionfinish(self, session):
        print("\n\nTest Durations:")
        for nodeid, duration in sorted(
            self.durations.items(),
            key=lambda x: x[1],
            reverse=True
        )[:10]:
            print(f"  {duration:.3f}s - {nodeid}")


def pytest_configure(config):
    config.pluginmanager.register(TestTimer(), "timer")
```

### Useful Hooks

```python
# conftest.py

def pytest_runtest_setup(item):
    """Called before each test"""
    print(f"Setting up: {item.name}")


def pytest_runtest_teardown(item, nextitem):
    """Called after each test"""
    print(f"Tearing down: {item.name}")


def pytest_collection_modifyitems(session, config, items):
    """Modify collected tests"""
    # Reorder tests
    items.sort(key=lambda x: x.name)


def pytest_exception_interact(node, call, report):
    """Called on test failure for debugging"""
    if report.failed:
        print(f"Test failed: {node.name}")


@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    """Access test result"""
    outcome = yield
    report = outcome.get_result()

    if report.when == "call" and report.failed:
        # Screenshot on failure for selenium tests
        if hasattr(item, "funcargs") and "browser" in item.funcargs:
            browser = item.funcargs["browser"]
            browser.save_screenshot(f"failure_{item.name}.png")
```

---

## Performance and Profiling

### Parallel Testing with pytest-xdist

```bash
# Install
pip install pytest-xdist

# Run with 4 workers
pytest -n 4

# Run with auto-detected workers
pytest -n auto

# Distribute by file
pytest --dist loadfile -n 4

# Distribute by test
pytest --dist loadscope -n 4
```

### Test Profiling

```python
# Built-in duration reporting
pytest --durations=10  # Show 10 slowest tests

# With pytest-profiling
# pip install pytest-profiling
pytest --profile

# Custom profiling
import cProfile
import pstats


@pytest.fixture
def profiler():
    prof = cProfile.Profile()
    prof.enable()
    yield prof
    prof.disable()
    stats = pstats.Stats(prof)
    stats.sort_stats("cumulative")
    stats.print_stats(10)


def test_with_profiling(profiler):
    expensive_operation()
```

### Memory Testing

```python
# pip install pytest-memray
# pytest --memray

# Or manual memory tracking
import tracemalloc


@pytest.fixture
def memory_tracker():
    tracemalloc.start()
    yield
    current, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    print(f"Current memory: {current / 1024:.2f} KB")
    print(f"Peak memory: {peak / 1024:.2f} KB")


def test_memory_usage(memory_tracker):
    data = [i for i in range(1000000)]
    assert len(data) == 1000000
```

---

## Configuration

### pytest.ini

```ini
[pytest]
minversion = 7.0
addopts = -ra -q --strict-markers
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
markers =
    slow: marks tests as slow
    integration: integration tests
filterwarnings =
    error
    ignore::DeprecationWarning
```

### pyproject.toml

```toml
[tool.pytest.ini_options]
minversion = "7.0"
addopts = "-ra -q --strict-markers"
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
markers = [
    "slow: marks tests as slow",
    "integration: integration tests",
]
filterwarnings = [
    "error",
    "ignore::DeprecationWarning",
]

[tool.coverage.run]
source = ["src"]
branch = true

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "raise NotImplementedError",
]
```

### Environment Variables

```python
# conftest.py
import os


@pytest.fixture(scope="session", autouse=True)
def setup_test_env():
    """Set environment variables for all tests"""
    os.environ["TESTING"] = "true"
    os.environ["DATABASE_URL"] = "postgresql://test:test@localhost/test"
    yield
    del os.environ["TESTING"]


# Or use pytest-env plugin
# pytest.ini
# [pytest]
# env =
#     TESTING=true
#     DATABASE_URL=postgresql://test:test@localhost/test
```

---

## Quick Reference

### CLI Commands

```bash
# Basic
pytest                      # Run all tests
pytest test_file.py         # Run specific file
pytest test_file.py::test   # Run specific test
pytest -k "pattern"         # Run tests matching pattern
pytest -x                   # Stop on first failure
pytest -v                   # Verbose output
pytest -q                   # Quiet output

# Markers
pytest -m slow              # Run marked tests
pytest -m "not slow"        # Exclude marked tests

# Coverage
pytest --cov=src            # With coverage
pytest --cov-report=html    # HTML coverage report

# Debugging
pytest --pdb                # Drop into debugger on failure
pytest --tb=short           # Shorter traceback
pytest -l                   # Show locals in traceback

# Parallel
pytest -n 4                 # 4 parallel workers
pytest -n auto              # Auto-detect workers
```

### Assertion Patterns

```python
# Basic
assert x == y
assert x != y
assert x is None
assert x is not None

# Collections
assert item in collection
assert len(items) == expected
assert all(condition for item in items)

# Exceptions
with pytest.raises(ValueError):
    raise ValueError("error")

with pytest.raises(ValueError, match="pattern"):
    raise ValueError("error with pattern")

# Approximate
assert value == pytest.approx(expected, rel=1e-3)
```
