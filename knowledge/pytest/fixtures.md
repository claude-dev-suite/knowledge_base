# Pytest Fixtures Deep Dive

Comprehensive guide to pytest fixtures.

**Official Documentation:** https://docs.pytest.org/en/stable/how-to/fixtures.html

---

## Table of Contents

1. [Fixture Basics](#fixture-basics)
2. [Fixture Scopes](#fixture-scopes)
3. [Fixture Finalization](#fixture-finalization)
4. [Fixture Factories](#fixture-factories)
5. [Parametrized Fixtures](#parametrized-fixtures)
6. [conftest.py](#conftestpy)
7. [Built-in Fixtures](#built-in-fixtures)
8. [Advanced Patterns](#advanced-patterns)

---

## Fixture Basics

### Creating Fixtures

```python
import pytest

@pytest.fixture
def sample_user():
    """Returns a sample user dict"""
    return {
        "id": 1,
        "name": "John Doe",
        "email": "john@example.com"
    }

def test_user_name(sample_user):
    assert sample_user["name"] == "John Doe"

def test_user_email(sample_user):
    assert "@" in sample_user["email"]
```

### Fixture with Dependencies

```python
@pytest.fixture
def database():
    return Database()

@pytest.fixture
def user_repository(database):
    return UserRepository(database)

@pytest.fixture
def user_service(user_repository):
    return UserService(user_repository)

def test_create_user(user_service):
    user = user_service.create(name="John")
    assert user.name == "John"
```

### Using Multiple Fixtures

```python
@pytest.fixture
def user():
    return User(name="John")

@pytest.fixture
def product():
    return Product(name="Widget", price=10.00)

def test_user_can_purchase(user, product):
    order = user.purchase(product)
    assert order.total == 10.00
```

---

## Fixture Scopes

### Function Scope (Default)

```python
@pytest.fixture  # scope="function" is default
def fresh_data():
    """New instance for each test"""
    return {"counter": 0}

def test_increment(fresh_data):
    fresh_data["counter"] += 1
    assert fresh_data["counter"] == 1

def test_decrement(fresh_data):
    # fresh_data is new, counter is 0
    fresh_data["counter"] -= 1
    assert fresh_data["counter"] == -1
```

### Class Scope

```python
@pytest.fixture(scope="class")
def shared_resource():
    """Shared across all tests in a class"""
    print("Creating resource")
    return {"value": 0}

class TestGroup:
    def test_one(self, shared_resource):
        shared_resource["value"] = 1
        assert shared_resource["value"] == 1

    def test_two(self, shared_resource):
        # Same instance from test_one
        assert shared_resource["value"] == 1
```

### Module Scope

```python
@pytest.fixture(scope="module")
def database_connection():
    """Created once per module"""
    print("Connecting to database")
    conn = create_connection()
    yield conn
    print("Closing connection")
    conn.close()
```

### Session Scope

```python
@pytest.fixture(scope="session")
def expensive_resource():
    """Created once for entire test session"""
    print("Creating expensive resource")
    resource = create_expensive_thing()
    yield resource
    print("Cleaning up")
    resource.cleanup()
```

### Package Scope

```python
# In tests/package/__init__.py or conftest.py
@pytest.fixture(scope="package")
def package_resource():
    """Shared across all tests in package"""
    return create_resource()
```

---

## Fixture Finalization

### Using yield

```python
@pytest.fixture
def database():
    # Setup
    db = Database()
    db.connect()
    db.begin_transaction()

    yield db  # Test runs here

    # Teardown (always runs)
    db.rollback_transaction()
    db.disconnect()

def test_insert(database):
    database.insert({"id": 1})
    # After test, rollback happens automatically
```

### Using request.addfinalizer

```python
@pytest.fixture
def files(request):
    created_files = []

    def create_file(name, content):
        path = f"/tmp/{name}"
        with open(path, "w") as f:
            f.write(content)
        created_files.append(path)
        return path

    def cleanup():
        for path in created_files:
            os.remove(path)

    request.addfinalizer(cleanup)
    return create_file
```

### Safe Teardown

```python
@pytest.fixture
def resource():
    resource = None
    try:
        resource = acquire_resource()
        yield resource
    finally:
        if resource:
            resource.release()
```

---

## Fixture Factories

### Basic Factory

```python
@pytest.fixture
def make_user():
    """Factory for creating users"""
    def _make_user(name="John", email=None):
        if email is None:
            email = f"{name.lower()}@example.com"
        return User(name=name, email=email)
    return _make_user

def test_multiple_users(make_user):
    user1 = make_user("Alice")
    user2 = make_user("Bob")
    assert user1.email != user2.email
```

### Factory with Cleanup

```python
@pytest.fixture
def make_temp_file():
    created_files = []

    def _make_temp_file(content=""):
        import tempfile
        fd, path = tempfile.mkstemp()
        os.write(fd, content.encode())
        os.close(fd)
        created_files.append(path)
        return path

    yield _make_temp_file

    # Cleanup all created files
    for path in created_files:
        if os.path.exists(path):
            os.remove(path)
```

### Factory Pattern with Database

```python
@pytest.fixture
def create_user(db_session):
    """Factory for creating users with auto-cleanup"""
    created_users = []

    def _create_user(**kwargs):
        defaults = {
            "name": "Test User",
            "email": f"test{len(created_users)}@example.com",
        }
        defaults.update(kwargs)
        user = User(**defaults)
        db_session.add(user)
        db_session.commit()
        created_users.append(user)
        return user

    yield _create_user

    # Cleanup
    for user in created_users:
        db_session.delete(user)
    db_session.commit()

def test_user_creation(create_user):
    user1 = create_user(name="Alice")
    user2 = create_user(name="Bob")
    assert User.query.count() == 2
```

---

## Parametrized Fixtures

### Basic Parametrization

```python
@pytest.fixture(params=["mysql", "postgresql", "sqlite"])
def database(request):
    """Test runs 3 times with different databases"""
    db = create_database(request.param)
    yield db
    db.cleanup()

def test_insert(database):
    database.insert({"id": 1})
    assert database.count() == 1
```

### With IDs

```python
@pytest.fixture(params=[
    pytest.param("mysql", id="mysql-db"),
    pytest.param("postgresql", id="postgres-db"),
    pytest.param("sqlite", id="sqlite-db"),
])
def database(request):
    return create_database(request.param)
```

### Complex Parameters

```python
@pytest.fixture(params=[
    {"name": "small", "size": 100},
    {"name": "medium", "size": 1000},
    {"name": "large", "size": 10000},
], ids=lambda x: x["name"])
def dataset(request):
    return generate_data(request.param["size"])
```

### Combining Parametrized Fixtures

```python
@pytest.fixture(params=["json", "xml"])
def format_type(request):
    return request.param

@pytest.fixture(params=["utf-8", "latin-1"])
def encoding(request):
    return request.param

def test_export(format_type, encoding):
    # Runs 4 times: json/utf-8, json/latin-1, xml/utf-8, xml/latin-1
    result = export(format=format_type, encoding=encoding)
    assert result is not None
```

---

## conftest.py

### Fixture Sharing

```python
# tests/conftest.py - Available to all tests

import pytest

@pytest.fixture(scope="session")
def app():
    """Application instance"""
    from myapp import create_app
    return create_app(testing=True)

@pytest.fixture
def client(app):
    """Test client"""
    return app.test_client()

@pytest.fixture
def db(app):
    """Database with transaction rollback"""
    from myapp.database import db as _db
    with app.app_context():
        _db.create_all()
        yield _db
        _db.session.rollback()
        _db.drop_all()
```

### Hierarchical conftest.py

```
tests/
├── conftest.py          # Root fixtures (session, app)
├── unit/
│   ├── conftest.py      # Unit test specific fixtures
│   └── test_models.py
├── integration/
│   ├── conftest.py      # Integration test fixtures
│   └── test_api.py
└── e2e/
    ├── conftest.py      # E2E test fixtures
    └── test_flows.py
```

### Fixture Override

```python
# tests/conftest.py
@pytest.fixture
def user():
    return User(name="Default User")

# tests/admin/conftest.py
@pytest.fixture
def user():
    """Override for admin tests"""
    return User(name="Admin User", is_admin=True)
```

---

## Built-in Fixtures

### tmp_path and tmp_path_factory

```python
def test_create_file(tmp_path):
    """tmp_path provides a temporary directory unique to the test"""
    file = tmp_path / "test.txt"
    file.write_text("hello")
    assert file.read_text() == "hello"

@pytest.fixture(scope="session")
def shared_temp_dir(tmp_path_factory):
    """tmp_path_factory for session-scoped temp dirs"""
    return tmp_path_factory.mktemp("shared")
```

### capsys and capfd

```python
def test_print_output(capsys):
    """Capture stdout/stderr"""
    print("hello")
    captured = capsys.readouterr()
    assert captured.out == "hello\n"
    assert captured.err == ""

def test_with_fd(capfd):
    """Capture at file descriptor level"""
    os.system("echo hello")
    captured = capfd.readouterr()
    assert "hello" in captured.out
```

### caplog

```python
import logging

def test_logging(caplog):
    """Capture log messages"""
    logger = logging.getLogger(__name__)

    with caplog.at_level(logging.INFO):
        logger.info("Test message")

    assert "Test message" in caplog.text
    assert len(caplog.records) == 1
    assert caplog.records[0].levelname == "INFO"
```

### monkeypatch

```python
def test_env_var(monkeypatch):
    """Modify environment"""
    monkeypatch.setenv("API_KEY", "test-key")
    assert os.environ["API_KEY"] == "test-key"

def test_dict_item(monkeypatch):
    """Modify dict"""
    d = {"key": "original"}
    monkeypatch.setitem(d, "key", "modified")
    assert d["key"] == "modified"

def test_attribute(monkeypatch):
    """Modify attribute"""
    class Config:
        debug = False

    monkeypatch.setattr(Config, "debug", True)
    assert Config.debug is True

def test_function(monkeypatch):
    """Replace function"""
    def mock_fetch():
        return {"mocked": True}

    monkeypatch.setattr("myapp.api.fetch", mock_fetch)
```

### request

```python
@pytest.fixture
def data(request):
    """Access test metadata"""
    print(f"Test name: {request.node.name}")
    print(f"Fixture name: {request.fixturename}")
    print(f"Scope: {request.scope}")

    # Access markers
    marker = request.node.get_closest_marker("slow")
    if marker:
        print("This is a slow test")

    return {}
```

### cache

```python
def test_with_cache(cache):
    """Access pytest cache"""
    # Store value
    cache.set("my_key", {"data": [1, 2, 3]})

    # Retrieve value
    value = cache.get("my_key", default=None)
    assert value == {"data": [1, 2, 3]}
```

---

## Advanced Patterns

### Fixture Composition

```python
@pytest.fixture
def base_user():
    return {"name": "User", "role": "user"}

@pytest.fixture
def admin_user(base_user):
    return {**base_user, "role": "admin", "permissions": ["all"]}

@pytest.fixture
def authenticated_admin(admin_user, auth_service):
    token = auth_service.generate_token(admin_user)
    return {**admin_user, "token": token}
```

### Lazy Fixtures (pytest-lazy-fixture)

```python
# pip install pytest-lazy-fixture
import pytest
from pytest_lazyfixture import lazy_fixture

@pytest.fixture
def user_a():
    return User(name="A")

@pytest.fixture
def user_b():
    return User(name="B")

@pytest.mark.parametrize("user", [
    lazy_fixture("user_a"),
    lazy_fixture("user_b"),
])
def test_user(user):
    assert user.name in ["A", "B"]
```

### Context Managers as Fixtures

```python
from contextlib import contextmanager

@pytest.fixture
def db_transaction(db):
    @contextmanager
    def _transaction():
        db.begin()
        try:
            yield db
            db.commit()
        except Exception:
            db.rollback()
            raise

    return _transaction

def test_with_transaction(db_transaction):
    with db_transaction() as db:
        db.insert(data)
```

### Async Fixtures

```python
import pytest
import asyncio

@pytest.fixture
async def async_client():
    """Async fixture with cleanup"""
    client = await create_async_client()
    yield client
    await client.close()

@pytest.mark.asyncio
async def test_async_operation(async_client):
    result = await async_client.fetch("/data")
    assert result["status"] == "ok"
```

### Dynamic Fixture Injection

```python
@pytest.fixture
def dynamic_fixture(request):
    """Inject fixtures dynamically"""
    fixture_name = request.param
    return request.getfixturevalue(fixture_name)

@pytest.mark.parametrize("dynamic_fixture", ["user", "admin"], indirect=True)
def test_with_dynamic(dynamic_fixture):
    assert dynamic_fixture is not None
```

---

## Best Practices

### Do

```python
# Use meaningful names
@pytest.fixture
def authenticated_api_client():
    pass

# Document fixtures
@pytest.fixture
def database():
    """
    Provides a clean database connection with automatic rollback.

    Usage:
        def test_something(database):
            database.insert(data)
    """
    pass

# Use appropriate scope
@pytest.fixture(scope="session")
def expensive_setup():
    pass  # Only created once
```

### Don't

```python
# Don't use global state
global_data = []  # Bad!

@pytest.fixture
def data():
    global_data.append(1)  # Causes test pollution
    return global_data

# Don't forget cleanup
@pytest.fixture
def file():
    f = open("test.txt", "w")
    yield f
    # Missing f.close()!  # Bad!
```

---

## Quick Reference

| Fixture | Description |
|---------|-------------|
| `@pytest.fixture` | Define fixture |
| `scope="function"` | Per test (default) |
| `scope="class"` | Per class |
| `scope="module"` | Per module |
| `scope="session"` | Per session |
| `autouse=True` | Auto-apply |
| `params=[...]` | Parametrize |
| `yield` | Setup/teardown |

| Built-in | Description |
|----------|-------------|
| `tmp_path` | Temp directory |
| `capsys` | Capture stdout |
| `caplog` | Capture logging |
| `monkeypatch` | Modify state |
| `request` | Test metadata |
| `cache` | Persistent cache |
