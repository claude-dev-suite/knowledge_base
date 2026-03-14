# Python Testing

Complete guide to Python testing with pytest and hypothesis.

## pytest Fundamentals

### Installation

```bash
uv add --dev pytest pytest-asyncio pytest-cov
```

### Basic Tests

```python
# tests/test_math.py
def test_addition() -> None:
    assert 1 + 1 == 2

def test_list_contains() -> None:
    items = [1, 2, 3]
    assert 2 in items
    assert 4 not in items

def test_raises_exception() -> None:
    import pytest
    with pytest.raises(ValueError):
        int("not a number")

def test_raises_with_match() -> None:
    import pytest
    with pytest.raises(ValueError, match="invalid literal"):
        int("abc")
```

### Test Classes

```python
class TestUserService:
    def test_create_user(self) -> None:
        user = create_user(name="Alice")
        assert user.name == "Alice"

    def test_user_email_lowercase(self) -> None:
        user = create_user(email="ALICE@EXAMPLE.COM")
        assert user.email == "alice@example.com"
```

## Fixtures

### Basic Fixtures

```python
import pytest

@pytest.fixture
def user() -> User:
    return User(name="Alice", email="alice@example.com")

@pytest.fixture
def db() -> Generator[Database, None, None]:
    connection = create_db_connection()
    yield connection
    connection.close()

def test_user_name(user: User) -> None:
    assert user.name == "Alice"

def test_user_in_db(db: Database, user: User) -> None:
    db.save(user)
    assert db.find(user.id) is not None
```

### Fixture Scopes

```python
@pytest.fixture(scope="function")  # Default - per test
def temp_file() -> Generator[Path, None, None]:
    path = Path("temp.txt")
    path.touch()
    yield path
    path.unlink()

@pytest.fixture(scope="class")  # Per test class
def class_resource() -> Resource:
    return Resource()

@pytest.fixture(scope="module")  # Per test module
def module_db() -> Database:
    return Database()

@pytest.fixture(scope="session")  # Per test session
def session_config() -> Config:
    return load_config()
```

### conftest.py

```python
# tests/conftest.py - Shared fixtures
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import Session, sessionmaker

@pytest.fixture(scope="session")
def engine() -> Engine:
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)
    yield engine
    engine.dispose()

@pytest.fixture(scope="function")
def db_session(engine: Engine) -> Generator[Session, None, None]:
    SessionLocal = sessionmaker(bind=engine)
    session = SessionLocal()
    yield session
    session.rollback()
    session.close()

@pytest.fixture
def client(db_session: Session) -> TestClient:
    def override_get_db() -> Generator[Session, None, None]:
        yield db_session

    app.dependency_overrides[get_db] = override_get_db
    with TestClient(app) as client:
        yield client
    app.dependency_overrides.clear()
```

## Parametrize

### Basic Parametrize

```python
import pytest

@pytest.mark.parametrize("input,expected", [
    (1, 2),
    (2, 4),
    (3, 6),
    (0, 0),
    (-1, -2),
])
def test_double(input: int, expected: int) -> None:
    assert double(input) == expected

@pytest.mark.parametrize("email,valid", [
    ("test@example.com", True),
    ("user.name@domain.co.uk", True),
    ("invalid-email", False),
    ("@nodomain.com", False),
    ("", False),
])
def test_validate_email(email: str, valid: bool) -> None:
    assert validate_email(email) == valid
```

### Multiple Parameters

```python
@pytest.mark.parametrize("x", [1, 2, 3])
@pytest.mark.parametrize("y", [10, 20])
def test_multiply(x: int, y: int) -> None:
    # Runs 6 times: (1,10), (1,20), (2,10), (2,20), (3,10), (3,20)
    assert multiply(x, y) == x * y
```

### IDs for Readability

```python
@pytest.mark.parametrize(
    "input,expected",
    [
        pytest.param(1, 2, id="positive"),
        pytest.param(0, 0, id="zero"),
        pytest.param(-1, -2, id="negative"),
    ]
)
def test_double(input: int, expected: int) -> None:
    assert double(input) == expected
```

## Mocking

### unittest.mock

```python
from unittest.mock import Mock, patch, AsyncMock, MagicMock

def test_with_mock() -> None:
    mock_api = Mock()
    mock_api.get_users.return_value = [{"name": "Alice"}]

    result = process_users(mock_api)

    mock_api.get_users.assert_called_once()
    assert len(result) == 1

@patch("myapp.services.external_api.fetch")
def test_with_patch(mock_fetch: Mock) -> None:
    mock_fetch.return_value = {"data": []}

    result = get_data()

    mock_fetch.assert_called_once_with("https://api.example.com")
    assert result == {"data": []}

def test_patch_object() -> None:
    with patch.object(MyClass, "method", return_value=42):
        obj = MyClass()
        assert obj.method() == 42
```

### pytest-mock

```python
def test_with_mocker(mocker) -> None:
    mock_fetch = mocker.patch("myapp.api.fetch")
    mock_fetch.return_value = {"data": []}

    result = get_data()

    mock_fetch.assert_called_once()
```

### Async Mocking

```python
from unittest.mock import AsyncMock

@pytest.mark.asyncio
async def test_async_function() -> None:
    mock_api = AsyncMock()
    mock_api.fetch.return_value = {"id": 1}

    result = await process_async(mock_api)

    mock_api.fetch.assert_awaited_once()
```

## Async Tests

```python
import pytest

@pytest.mark.asyncio
async def test_async_fetch() -> None:
    result = await fetch_data()
    assert result is not None

@pytest.mark.asyncio
async def test_async_with_fixture(async_client: AsyncClient) -> None:
    response = await async_client.get("/api/users")
    assert response.status_code == 200

# Fixture for async
@pytest.fixture
async def async_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSession(engine) as session:
        yield session
        await session.rollback()
```

### pytest-asyncio Configuration

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"  # Automatically mark async tests
```

## Hypothesis (Property-Based Testing)

### Installation

```bash
uv add --dev hypothesis
```

### Basic Strategies

```python
from hypothesis import given, strategies as st

@given(st.integers(), st.integers())
def test_addition_commutative(a: int, b: int) -> None:
    assert a + b == b + a

@given(st.lists(st.integers()))
def test_sort_idempotent(items: list[int]) -> None:
    assert sorted(sorted(items)) == sorted(items)

@given(st.text(min_size=1, max_size=100))
def test_string_nonempty(s: str) -> None:
    assert len(s) >= 1

@given(st.floats(allow_nan=False, allow_infinity=False))
def test_float_roundtrip(x: float) -> None:
    assert float(str(x)) == pytest.approx(x, rel=1e-10)
```

### Common Strategies

```python
from hypothesis import strategies as st

# Numbers
st.integers()
st.integers(min_value=0, max_value=100)
st.floats(min_value=-1.0, max_value=1.0)

# Text
st.text()
st.text(min_size=1, max_size=50)
st.from_regex(r"[a-z]+@[a-z]+\.[a-z]{2,}")

# Collections
st.lists(st.integers())
st.lists(st.integers(), min_size=1, max_size=10)
st.dictionaries(st.text(), st.integers())
st.sets(st.integers())

# Optional/None
st.none()
st.one_of(st.integers(), st.none())
st.integers() | st.none()

# Choices
st.sampled_from(["a", "b", "c"])
st.sampled_from(MyEnum)

# Dates/Times
st.dates()
st.datetimes()
st.timedeltas()
```

### Custom Strategies

```python
from hypothesis import strategies as st
from dataclasses import dataclass

@dataclass
class User:
    name: str
    age: int
    email: str

# Build from dataclass
user_strategy = st.builds(
    User,
    name=st.text(min_size=1, max_size=50),
    age=st.integers(min_value=0, max_value=150),
    email=st.emails(),
)

@given(user_strategy)
def test_user_valid(user: User) -> None:
    assert user.name
    assert 0 <= user.age <= 150
    assert "@" in user.email

# Composite strategy
@st.composite
def valid_user(draw: st.DrawFn) -> User:
    name = draw(st.text(min_size=1, max_size=50))
    age = draw(st.integers(min_value=18, max_value=100))
    email = draw(st.emails())
    return User(name=name, age=age, email=email)

@given(valid_user())
def test_adult_user(user: User) -> None:
    assert user.age >= 18
```

### Stateful Testing

```python
from hypothesis.stateful import RuleBasedStateMachine, rule, invariant

class SetMachine(RuleBasedStateMachine):
    def __init__(self) -> None:
        super().__init__()
        self.model: set[int] = set()
        self.impl = MySet()

    @rule(value=st.integers())
    def add(self, value: int) -> None:
        self.model.add(value)
        self.impl.add(value)

    @rule(value=st.integers())
    def remove(self, value: int) -> None:
        self.model.discard(value)
        self.impl.remove(value)

    @invariant()
    def size_matches(self) -> None:
        assert len(self.model) == len(self.impl)

    @invariant()
    def contents_match(self) -> None:
        for item in self.model:
            assert self.impl.contains(item)

TestSet = SetMachine.TestCase
```

### Configuration

```toml
# pyproject.toml
[tool.hypothesis]
deadline = 500
max_examples = 100
database = ".hypothesis"
verbosity = "normal"
```

## Coverage

### Configuration

```toml
# pyproject.toml
[tool.coverage.run]
source = ["src"]
branch = true
omit = [
    "*/tests/*",
    "*/__init__.py",
    "*/conftest.py",
]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "if TYPE_CHECKING:",
    "raise NotImplementedError",
    "@abstractmethod",
    "if __name__ == .__main__.:",
]
fail_under = 80
show_missing = true

[tool.coverage.html]
directory = "htmlcov"
```

### Commands

```bash
# Run with coverage
pytest --cov=src --cov-report=term-missing

# HTML report
pytest --cov=src --cov-report=html

# XML for CI
pytest --cov=src --cov-report=xml

# Fail if below threshold
pytest --cov=src --cov-fail-under=80
```

## pytest Configuration

```toml
# pyproject.toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py", "*_test.py"]
python_functions = ["test_*"]
python_classes = ["Test*"]
addopts = [
    "-v",
    "--strict-markers",
    "--strict-config",
    "-ra",
]
markers = [
    "slow: marks tests as slow",
    "integration: integration tests",
    "unit: unit tests",
]
filterwarnings = [
    "error",
    "ignore::DeprecationWarning",
]
asyncio_mode = "auto"
```

## Markers

```python
import pytest

@pytest.mark.slow
def test_slow_operation() -> None:
    # Long-running test
    pass

@pytest.mark.integration
def test_database_integration(db: Database) -> None:
    pass

@pytest.mark.skip(reason="Not implemented yet")
def test_future_feature() -> None:
    pass

@pytest.mark.skipif(sys.platform == "win32", reason="Unix only")
def test_unix_feature() -> None:
    pass

@pytest.mark.xfail(reason="Known bug #123")
def test_known_bug() -> None:
    pass
```

```bash
# Run only slow tests
pytest -m slow

# Skip slow tests
pytest -m "not slow"

# Run slow OR integration
pytest -m "slow or integration"
```

## Best Practices

1. **Use fixtures** for setup/teardown
2. **Use parametrize** for multiple test cases
3. **Use hypothesis** for edge cases
4. **Use conftest.py** for shared fixtures
5. **Isolate tests** - no shared state
6. **Mock external services**
7. **Test error cases** not just happy path
8. **Keep tests fast** - mock slow operations
9. **Use coverage** to find gaps
10. **Name tests descriptively**

## Directory Structure

```
tests/
├── conftest.py          # Shared fixtures
├── factories.py         # Test data factories
├── unit/
│   ├── test_models.py
│   └── test_services.py
├── integration/
│   ├── conftest.py      # Integration fixtures
│   ├── test_api.py
│   └── test_database.py
└── e2e/
    └── test_workflows.py
```

## References

- [pytest documentation](https://docs.pytest.org/)
- [hypothesis documentation](https://hypothesis.readthedocs.io/)
- [pytest-asyncio](https://pytest-asyncio.readthedocs.io/)
- [coverage.py](https://coverage.readthedocs.io/)
