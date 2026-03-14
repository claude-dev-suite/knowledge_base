# Pytest Testing Framework Reference

Comprehensive documentation for pytest testing framework.

**Official Documentation:** https://docs.pytest.org/en/stable/

---

## Table of Contents

1. [Getting Started](#getting-started)
2. [Test Functions](#test-functions)
3. [Assertions](#assertions)
4. [Fixtures](#fixtures)
5. [Markers](#markers)
6. [Parametrization](#parametrization)
7. [Mocking](#mocking)
8. [Configuration](#configuration)
9. [CLI Options](#cli-options)
10. [Best Practices](#best-practices)

---

## Getting Started

### Installation

```bash
pip install pytest
pip install pytest-cov  # Coverage
pip install pytest-asyncio  # Async support
pip install pytest-mock  # Better mocking
```

### Test Discovery

Pytest automatically discovers tests matching:
- Files: `test_*.py` or `*_test.py`
- Classes: `Test*` (no `__init__` method)
- Functions: `test_*`

```
project/
├── tests/
│   ├── __init__.py
│   ├── test_models.py
│   ├── test_services.py
│   └── conftest.py      # Shared fixtures
├── src/
│   └── myapp/
└── pyproject.toml
```

### Basic Configuration

```toml
# pyproject.toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = "-v --tb=short"
```

---

## Test Functions

### Basic Tests

```python
def test_addition():
    assert 1 + 1 == 2

def test_string_operations():
    assert "hello".upper() == "HELLO"
    assert "HELLO".lower() == "hello"

def test_list_operations():
    items = [1, 2, 3]
    assert len(items) == 3
    assert 2 in items
```

### Test Classes

```python
class TestCalculator:
    def test_add(self):
        assert add(1, 2) == 3

    def test_subtract(self):
        assert subtract(5, 3) == 2

    def test_multiply(self):
        assert multiply(3, 4) == 12
```

### Setup and Teardown

```python
class TestDatabase:
    def setup_method(self):
        """Run before each test method"""
        self.db = create_test_database()

    def teardown_method(self):
        """Run after each test method"""
        self.db.close()

    def setup_class(cls):
        """Run once before all tests in class"""
        cls.connection = connect_to_server()

    def teardown_class(cls):
        """Run once after all tests in class"""
        cls.connection.close()
```

### Module-Level Setup

```python
def setup_module(module):
    """Run once before any test in module"""
    initialize_test_environment()

def teardown_module(module):
    """Run once after all tests in module"""
    cleanup_test_environment()
```

---

## Assertions

### Basic Assertions

```python
def test_assertions():
    # Equality
    assert value == expected
    assert value != other

    # Identity
    assert value is None
    assert value is not None

    # Truthiness
    assert value  # Truthy
    assert not value  # Falsy

    # Membership
    assert item in collection
    assert item not in collection

    # Comparison
    assert value > 5
    assert value >= 5
    assert value < 10
    assert value <= 10
```

### Approximate Comparisons

```python
import pytest

def test_float_comparison():
    # Using pytest.approx
    assert 0.1 + 0.2 == pytest.approx(0.3)
    assert 0.3 == pytest.approx(0.3, rel=1e-3)  # Relative tolerance
    assert 0.3 == pytest.approx(0.3, abs=1e-3)  # Absolute tolerance

    # For lists/dicts
    assert [0.1, 0.2] == pytest.approx([0.1, 0.2])
    assert {"a": 0.1} == pytest.approx({"a": 0.1})
```

### Exception Testing

```python
import pytest

def test_raises_exception():
    with pytest.raises(ValueError):
        raise ValueError("invalid value")

def test_exception_message():
    with pytest.raises(ValueError, match="invalid"):
        raise ValueError("invalid value")

def test_exception_attributes():
    with pytest.raises(ValueError) as exc_info:
        raise ValueError("test message")

    assert "test" in str(exc_info.value)
    assert exc_info.type == ValueError
```

### Warning Testing

```python
import warnings
import pytest

def test_warning():
    with pytest.warns(UserWarning):
        warnings.warn("my warning", UserWarning)

def test_warning_message():
    with pytest.warns(UserWarning, match="specific"):
        warnings.warn("specific warning", UserWarning)
```

### Assertion Rewriting

```python
# Pytest automatically rewrites assertions for better messages
def test_list_comparison():
    actual = [1, 2, 3]
    expected = [1, 2, 4]
    assert actual == expected
    # Output shows exact difference:
    # AssertionError: assert [1, 2, 3] == [1, 2, 4]
    #   At index 2 diff: 3 != 4
```

---

## Fixtures

### Basic Fixtures

```python
import pytest

@pytest.fixture
def sample_data():
    return {"name": "John", "age": 30}

def test_with_fixture(sample_data):
    assert sample_data["name"] == "John"
    assert sample_data["age"] == 30
```

### Fixture Scopes

```python
@pytest.fixture(scope="function")  # Default: new for each test
def function_fixture():
    return create_resource()

@pytest.fixture(scope="class")  # Once per test class
def class_fixture():
    return create_resource()

@pytest.fixture(scope="module")  # Once per module
def module_fixture():
    return create_resource()

@pytest.fixture(scope="session")  # Once per test session
def session_fixture():
    return create_expensive_resource()
```

### Setup and Teardown in Fixtures

```python
@pytest.fixture
def database():
    # Setup
    db = create_database()
    db.connect()

    yield db  # Test runs here

    # Teardown
    db.disconnect()
    db.cleanup()

def test_with_database(database):
    database.insert({"id": 1})
    assert database.count() == 1
```

### Fixture Dependencies

```python
@pytest.fixture
def database():
    return create_database()

@pytest.fixture
def user(database):
    user = User(name="John")
    database.insert(user)
    return user

@pytest.fixture
def authenticated_client(user):
    client = TestClient()
    client.authenticate(user)
    return client

def test_authenticated_request(authenticated_client):
    response = authenticated_client.get("/profile")
    assert response.status_code == 200
```

### Autouse Fixtures

```python
@pytest.fixture(autouse=True)
def setup_logging():
    """Automatically used by all tests"""
    configure_test_logging()
    yield
    reset_logging()
```

### Fixture Factories

```python
@pytest.fixture
def make_user():
    created_users = []

    def _make_user(name, email):
        user = User(name=name, email=email)
        created_users.append(user)
        return user

    yield _make_user

    # Cleanup
    for user in created_users:
        user.delete()

def test_multiple_users(make_user):
    user1 = make_user("John", "john@example.com")
    user2 = make_user("Jane", "jane@example.com")
    assert user1.name != user2.name
```

### conftest.py

```python
# tests/conftest.py - Shared fixtures
import pytest

@pytest.fixture(scope="session")
def app():
    """Create application instance for testing"""
    from myapp import create_app
    app = create_app(testing=True)
    return app

@pytest.fixture
def client(app):
    """Test client for making requests"""
    return app.test_client()

@pytest.fixture
def db(app):
    """Database fixture with transaction rollback"""
    from myapp.database import db as _db
    _db.create_all()
    yield _db
    _db.drop_all()
```

---

## Markers

### Built-in Markers

```python
import pytest

# Skip test
@pytest.mark.skip(reason="Not implemented yet")
def test_not_ready():
    pass

# Skip conditionally
@pytest.mark.skipif(
    sys.platform == "win32",
    reason="Does not run on Windows"
)
def test_unix_only():
    pass

# Expected failure
@pytest.mark.xfail(reason="Known bug")
def test_known_issue():
    assert False  # Won't cause test failure

# Strict expected failure
@pytest.mark.xfail(strict=True)
def test_strict_xfail():
    pass  # Fails if test passes!
```

### Custom Markers

```python
# pytest.ini or pyproject.toml
[tool.pytest.ini_options]
markers = [
    "slow: marks tests as slow",
    "integration: integration tests",
    "unit: unit tests",
]

# In tests
@pytest.mark.slow
def test_slow_operation():
    time.sleep(5)

@pytest.mark.integration
def test_database_connection():
    pass

# Run specific markers
# pytest -m slow
# pytest -m "not slow"
# pytest -m "slow or integration"
```

### Filtering with Markers

```bash
# Run only slow tests
pytest -m slow

# Run everything except slow tests
pytest -m "not slow"

# Combine markers
pytest -m "slow and integration"
pytest -m "slow or integration"
```

---

## Parametrization

### Basic Parametrization

```python
import pytest

@pytest.mark.parametrize("input,expected", [
    (1, 2),
    (2, 4),
    (3, 6),
])
def test_double(input, expected):
    assert input * 2 == expected
```

### Multiple Parameters

```python
@pytest.mark.parametrize("a,b,expected", [
    (1, 1, 2),
    (2, 3, 5),
    (10, 20, 30),
])
def test_addition(a, b, expected):
    assert a + b == expected
```

### Parameter IDs

```python
@pytest.mark.parametrize("user,expected", [
    pytest.param(
        {"name": "John", "admin": True},
        True,
        id="admin-user"
    ),
    pytest.param(
        {"name": "Jane", "admin": False},
        False,
        id="regular-user"
    ),
])
def test_is_admin(user, expected):
    assert user["admin"] == expected
```

### Combining Parametrize

```python
@pytest.mark.parametrize("x", [0, 1])
@pytest.mark.parametrize("y", [2, 3])
def test_combinations(x, y):
    # Runs 4 times: (0,2), (0,3), (1,2), (1,3)
    pass
```

### Parametrized Fixtures

```python
@pytest.fixture(params=["mysql", "postgresql", "sqlite"])
def database(request):
    return create_database(request.param)

def test_database_operations(database):
    # Runs 3 times with different databases
    database.insert({"id": 1})
    assert database.count() == 1
```

---

## Mocking

### Using pytest-mock

```python
# pip install pytest-mock

def test_with_mock(mocker):
    # Mock a function
    mock_fetch = mocker.patch("myapp.api.fetch_data")
    mock_fetch.return_value = {"data": "mocked"}

    result = fetch_data()
    assert result == {"data": "mocked"}
    mock_fetch.assert_called_once()

def test_mock_method(mocker):
    # Mock object method
    user = User()
    mocker.patch.object(user, "save", return_value=True)

    assert user.save() == True
```

### Using unittest.mock

```python
from unittest.mock import Mock, patch, MagicMock

def test_with_mock():
    mock = Mock()
    mock.return_value = 42
    assert mock() == 42
    mock.assert_called_once()

def test_with_patch():
    with patch("myapp.api.fetch") as mock_fetch:
        mock_fetch.return_value = {"data": []}
        result = fetch_data()
        assert result == {"data": []}

@patch("myapp.api.fetch")
def test_with_decorator(mock_fetch):
    mock_fetch.return_value = {"data": []}
    result = fetch_data()
    assert result == {"data": []}
```

### Async Mocking

```python
from unittest.mock import AsyncMock

@pytest.mark.asyncio
async def test_async_mock(mocker):
    mock_fetch = mocker.patch(
        "myapp.api.async_fetch",
        new_callable=AsyncMock
    )
    mock_fetch.return_value = {"data": "mocked"}

    result = await async_fetch()
    assert result == {"data": "mocked"}
```

### Spying

```python
def test_spy(mocker):
    spy = mocker.spy(MyClass, "method")

    obj = MyClass()
    obj.method("arg")

    spy.assert_called_once_with("arg")
```

---

## Configuration

### pyproject.toml

```toml
[tool.pytest.ini_options]
minversion = "7.0"
testpaths = ["tests"]
python_files = ["test_*.py", "*_test.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]

# CLI options
addopts = [
    "-v",
    "--tb=short",
    "--strict-markers",
    "-ra",  # Show extra test summary
]

# Markers
markers = [
    "slow: marks tests as slow",
    "integration: integration tests",
]

# Async mode
asyncio_mode = "auto"

# Coverage
[tool.coverage.run]
source = ["src"]
branch = true

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "if TYPE_CHECKING:",
]
```

### pytest.ini

```ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts = -v --tb=short
filterwarnings =
    error
    ignore::DeprecationWarning
```

---

## CLI Options

```bash
# Run all tests
pytest

# Run specific file
pytest tests/test_api.py

# Run specific test
pytest tests/test_api.py::test_function
pytest tests/test_api.py::TestClass::test_method

# Run tests matching pattern
pytest -k "test_user"
pytest -k "test_user and not slow"

# Run with markers
pytest -m slow
pytest -m "not slow"

# Verbose output
pytest -v
pytest -vv  # More verbose

# Stop on first failure
pytest -x
pytest --maxfail=3

# Show local variables in tracebacks
pytest -l

# Show print statements
pytest -s

# Parallel execution (requires pytest-xdist)
pytest -n auto
pytest -n 4

# Coverage
pytest --cov=myapp
pytest --cov=myapp --cov-report=html

# Run last failed
pytest --lf
pytest --ff  # Failed first

# Collect only (don't run)
pytest --collect-only

# Show slowest tests
pytest --durations=10

# Debug
pytest --pdb  # Drop into debugger on failure
pytest --trace  # Drop into debugger at start
```

---

## Best Practices

### Test Organization

```python
# Group related tests
class TestUserCreation:
    def test_creates_user_with_valid_data(self):
        pass

    def test_raises_on_invalid_email(self):
        pass

    def test_hashes_password(self):
        pass
```

### Arrange-Act-Assert Pattern

```python
def test_user_login():
    # Arrange
    user = create_user(email="test@example.com", password="secret")

    # Act
    result = login(email="test@example.com", password="secret")

    # Assert
    assert result.success is True
    assert result.user.email == "test@example.com"
```

### Test Naming

```python
# Use descriptive names
def test_user_can_change_password():
    pass

def test_returns_error_when_email_is_invalid():
    pass

def test_creates_audit_log_on_deletion():
    pass
```

### Fixture Best Practices

```python
# Use fixtures for shared setup
@pytest.fixture
def authenticated_user(db, make_user):
    user = make_user(email="test@example.com")
    user.activate()
    return user

# Use scope appropriately
@pytest.fixture(scope="session")
def expensive_resource():
    # Only created once
    return create_expensive_thing()
```

---

## Quick Reference

### Test Functions

| Function | Description |
|----------|-------------|
| `def test_*()` | Define a test |
| `class Test*` | Group tests |
| `@pytest.fixture` | Create fixture |
| `@pytest.mark.*` | Apply marker |
| `@pytest.mark.parametrize` | Parametrize test |

### Common Markers

| Marker | Description |
|--------|-------------|
| `@pytest.mark.skip` | Skip test |
| `@pytest.mark.skipif` | Skip conditionally |
| `@pytest.mark.xfail` | Expected failure |
| `@pytest.mark.parametrize` | Parametrize |
| `@pytest.mark.asyncio` | Async test |

### Assertions

| Assertion | Description |
|-----------|-------------|
| `assert x == y` | Equality |
| `assert x is None` | Identity |
| `assert x in y` | Membership |
| `pytest.raises(E)` | Exception |
| `pytest.warns(W)` | Warning |
| `pytest.approx(x)` | Float comparison |

### Fixture Scopes

| Scope | Description |
|-------|-------------|
| `function` | New for each test (default) |
| `class` | Once per test class |
| `module` | Once per module |
| `session` | Once per test run |
