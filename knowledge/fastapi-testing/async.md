# FastAPI Testing — Async

> **Source:** FastAPI advanced docs (https://fastapi.tiangolo.com/advanced/async-tests/), pytest-asyncio docs, anyio docs, httpx `AsyncClient` docs.

---

## 1. Why Async Tests?

`TestClient` drives your ASGI app synchronously — it runs an event loop internally to handle each request. This works for most tests, but fails when:

- Your fixtures themselves are async (e.g., async DB session setup)
- You need to test streaming responses and consume them incrementally
- You test WebSocket behaviour with async read/write interleaving
- You want to test `SSE` or long-poll endpoints without blocking
- You use Motor (async MongoDB), aioredis, or `asyncpg` directly in fixtures

The solution is `httpx.AsyncClient` with `ASGITransport`, paired with `pytest-asyncio`.

---

## 2. Required Packages

```bash
pip install httpx pytest-asyncio anyio
# For async SQLAlchemy:
pip install sqlalchemy[asyncio] aiosqlite
# For MongoDB:
pip install motor
```

---

## 3. `AsyncClient` with `ASGITransport`

```python
# tests/test_async_basic.py
import pytest
import httpx
from app.main import app

@pytest.mark.anyio
async def test_read_root():
    async with httpx.AsyncClient(
        transport=httpx.ASGITransport(app=app),
        base_url="http://test",
    ) as client:
        response = await client.get("/")
    assert response.status_code == 200
    assert response.json() == {"message": "Hello World"}
```

`ASGITransport` routes HTTP calls directly to the ASGI app — no network sockets are opened.

---

## 4. `pytest-asyncio` Configuration

### asyncio_mode = "auto"

Setting `asyncio_mode = "auto"` removes the need to decorate every async test with `@pytest.mark.asyncio`:

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

```ini
# pytest.ini
[pytest]
asyncio_mode = auto
```

With `auto` mode, any `async def test_*` function is automatically treated as an async test.

### asyncio_mode = "strict" (explicit)

```toml
[tool.pytest.ini_options]
asyncio_mode = "strict"
```

With `strict`, you must explicitly mark each async test:

```python
@pytest.mark.asyncio
async def test_something():
    ...
```

---

## 5. anyio Backend Configuration

`pytest-asyncio` >= 0.21 uses anyio internally. You can select the backend at the session or test level.

### Selecting backend globally

```python
# conftest.py
import pytest

@pytest.fixture(scope="session")
def anyio_backend():
    return "asyncio"  # or "trio"
```

### Selecting backend per test

```python
@pytest.mark.anyio(backend="asyncio")
async def test_with_asyncio():
    ...

@pytest.mark.anyio(backend="trio")
async def test_with_trio():
    ...
```

### Running tests on both backends

```python
@pytest.fixture(params=["asyncio", "trio"])
def anyio_backend(request):
    return request.param

@pytest.mark.anyio
async def test_runs_on_both_backends():
    async with httpx.AsyncClient(
        transport=httpx.ASGITransport(app=app),
        base_url="http://test",
    ) as client:
        response = await client.get("/")
    assert response.status_code == 200
```

---

## 6. Async Fixtures and `@pytest_asyncio.fixture`

When your fixture itself is `async`, use `@pytest_asyncio.fixture`:

```python
# tests/conftest.py
import pytest
import pytest_asyncio
import httpx
from app.main import app

@pytest_asyncio.fixture
async def async_client():
    async with httpx.AsyncClient(
        transport=httpx.ASGITransport(app=app),
        base_url="http://test",
    ) as client:
        yield client

# With asyncio_mode = "auto", you can also use plain @pytest.fixture:
@pytest.fixture
async def async_client_auto():
    async with httpx.AsyncClient(
        transport=httpx.ASGITransport(app=app),
        base_url="http://test",
    ) as client:
        yield client
```

```python
# tests/test_with_fixture.py
async def test_get_items(async_client):
    response = await async_client.get("/items/")
    assert response.status_code == 200
```

### Scoped async fixtures

```python
@pytest_asyncio.fixture(scope="module")
async def module_client():
    async with httpx.AsyncClient(
        transport=httpx.ASGITransport(app=app),
        base_url="http://test",
    ) as client:
        yield client
```

Note: `scope="session"` async fixtures require `asyncio_mode = "auto"` or explicit session-scoped event loop configuration.

---

## 7. Lifespan Events in Async Tests

### Using `asynccontextmanager` lifespan

```python
# app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    # startup: initialize resources
    app.state.db_pool = await create_async_pool()
    app.state.cache = await create_redis_pool()
    yield
    # shutdown: release resources
    await app.state.db_pool.close()
    await app.state.cache.close()

app = FastAPI(lifespan=lifespan)
```

`AsyncClient` with `ASGITransport` triggers the lifespan **automatically** when entering the context manager:

```python
async def test_lifespan_runs(async_client):
    # lifespan startup ran when async_client was created
    response = await async_client.get("/health")
    assert response.status_code == 200
    # lifespan shutdown runs when async_client exits
```

### Overriding lifespan for tests

```python
# tests/conftest.py
from contextlib import asynccontextmanager
import pytest_asyncio
import httpx
from fastapi import FastAPI
from app.routers import items_router, users_router

@asynccontextmanager
async def test_lifespan(app: FastAPI):
    # Use test resources instead of production ones
    app.state.db_pool = FakeAsyncPool()
    app.state.cache = FakeRedis()
    yield

@pytest_asyncio.fixture
async def test_app_client():
    test_app = FastAPI(lifespan=test_lifespan)
    test_app.include_router(items_router)
    test_app.include_router(users_router)

    async with httpx.AsyncClient(
        transport=httpx.ASGITransport(app=test_app),
        base_url="http://test",
    ) as client:
        yield client
```

---

## 8. Async Database Connections

### SQLAlchemy Async (aiosqlite / asyncpg)

```python
# app/database.py
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker
from sqlalchemy.orm import DeclarativeBase

DATABASE_URL = "postgresql+asyncpg://user:pass@localhost/mydb"

engine = create_async_engine(DATABASE_URL, echo=True)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

class Base(DeclarativeBase):
    pass

async def get_async_db() -> AsyncSession:
    async with AsyncSessionLocal() as session:
        yield session
```

```python
# tests/conftest.py
import pytest_asyncio
from sqlalchemy.ext.asyncio import (
    AsyncSession,
    create_async_engine,
    async_sessionmaker,
)
import httpx
from app.main import app
from app.database import Base, get_async_db

TEST_DATABASE_URL = "sqlite+aiosqlite:///:memory:"

@pytest_asyncio.fixture(scope="session")
async def async_engine():
    engine = create_async_engine(TEST_DATABASE_URL)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine.dispose()


@pytest_asyncio.fixture
async def async_db(async_engine) -> AsyncSession:
    """Each test gets a rolled-back transaction."""
    async with async_engine.begin() as connection:
        async_session = async_sessionmaker(
            bind=connection, expire_on_commit=False
        )
        async with async_session() as session:
            yield session
            await session.rollback()


@pytest_asyncio.fixture
async def async_client(async_db: AsyncSession):
    async def override_get_db():
        yield async_db

    app.dependency_overrides[get_async_db] = override_get_db
    async with httpx.AsyncClient(
        transport=httpx.ASGITransport(app=app),
        base_url="http://test",
    ) as client:
        yield client
    app.dependency_overrides.clear()
```

```python
# tests/test_async_db.py
async def test_create_item(async_client):
    response = await async_client.post(
        "/items/",
        json={"name": "Async Widget", "price": 14.99},
    )
    assert response.status_code == 201
    assert response.json()["name"] == "Async Widget"

async def test_list_items(async_client):
    # Create first
    await async_client.post("/items/", json={"name": "Item A", "price": 1.00})
    # Then list
    response = await async_client.get("/items/")
    assert response.status_code == 200
    assert len(response.json()) >= 1
```

### Motor (async MongoDB)

```python
# app/database.py
import motor.motor_asyncio

MONGO_URL = "mongodb://localhost:27017"
client = motor.motor_asyncio.AsyncIOMotorClient(MONGO_URL)
db = client["mydb"]

async def get_db():
    return db
```

```python
# tests/conftest.py
import pytest_asyncio
from unittest.mock import AsyncMock, MagicMock
import httpx
from app.main import app
from app.database import get_db

@pytest_asyncio.fixture
async def mock_mongo_db():
    """Create a mock MongoDB database."""
    mock_db = MagicMock()

    # Mock collection with async methods
    mock_collection = AsyncMock()
    mock_collection.find_one.return_value = {"_id": "abc123", "name": "Test"}
    mock_collection.insert_one.return_value = MagicMock(inserted_id="new_id")
    mock_collection.find.return_value.__aiter__ = AsyncMock(
        return_value=iter([{"_id": "1", "name": "Item 1"}])
    )

    mock_db.__getitem__ = MagicMock(return_value=mock_collection)
    return mock_db


@pytest_asyncio.fixture
async def async_client_mongo(mock_mongo_db):
    async def override_get_db():
        return mock_mongo_db

    app.dependency_overrides[get_db] = override_get_db
    async with httpx.AsyncClient(
        transport=httpx.ASGITransport(app=app),
        base_url="http://test",
    ) as client:
        yield client
    app.dependency_overrides.clear()
```

---

## 9. Testing Async Streaming Responses

### `StreamingResponse`

```python
# app/main.py
import asyncio
from fastapi.responses import StreamingResponse

async def fake_video_stream():
    for i in range(5):
        await asyncio.sleep(0.01)
        yield f"chunk-{i}\n".encode()

@app.get("/stream/video")
async def stream_video():
    return StreamingResponse(
        fake_video_stream(),
        media_type="video/mp4",
    )

@app.get("/stream/text")
async def stream_text():
    async def generate():
        words = ["Hello", " ", "streaming", " ", "world"]
        for word in words:
            yield word.encode()
            await asyncio.sleep(0.001)

    return StreamingResponse(generate(), media_type="text/plain")
```

```python
# tests/test_streaming.py
async def test_stream_collects_all_chunks(async_client):
    response = await async_client.get("/stream/text")
    assert response.status_code == 200
    # AsyncClient collects entire streamed response
    assert response.text == "Hello streaming world"

async def test_stream_content_type(async_client):
    response = await async_client.get("/stream/video")
    assert response.headers["content-type"] == "video/mp4"
    assert len(response.content) > 0

async def test_stream_incremental():
    """Consume stream chunk by chunk."""
    chunks = []
    async with httpx.AsyncClient(
        transport=httpx.ASGITransport(app=app),
        base_url="http://test",
    ) as client:
        async with client.stream("GET", "/stream/text") as response:
            assert response.status_code == 200
            async for chunk in response.aiter_bytes():
                chunks.append(chunk)

    assert b"".join(chunks) == b"Hello streaming world"
```

---

## 10. Testing Server-Sent Events (SSE)

```python
# app/main.py
import asyncio
from fastapi.responses import StreamingResponse

@app.get("/events")
async def event_stream():
    async def generate():
        for i in range(3):
            yield f"data: event {i}\n\n"
            await asyncio.sleep(0.01)
        yield "data: done\n\n"

    return StreamingResponse(generate(), media_type="text/event-stream")
```

```python
# tests/test_sse.py
async def test_sse_events(async_client):
    chunks = []
    async with async_client.stream("GET", "/events") as response:
        assert response.status_code == 200
        assert response.headers["content-type"] == "text/event-stream; charset=utf-8"
        async for line in response.aiter_lines():
            if line.startswith("data:"):
                chunks.append(line[6:])  # strip "data: "

    assert chunks == ["event 0", "event 1", "event 2", "done"]

async def test_sse_raw_format(async_client):
    """Verify raw SSE message format."""
    full_body = b""
    async with async_client.stream("GET", "/events") as response:
        async for chunk in response.aiter_bytes():
            full_body += chunk

    events = [e for e in full_body.decode().split("\n\n") if e]
    assert len(events) == 4  # 3 + done
    assert all(e.startswith("data:") for e in events)
```

---

## 11. Async WebSocket Testing

```python
# app/main.py
from fastapi import WebSocket, WebSocketDisconnect

@app.websocket("/ws/async")
async def async_ws(websocket: WebSocket):
    await websocket.accept()
    try:
        while True:
            msg = await websocket.receive_json()
            await websocket.send_json({"echo": msg})
    except WebSocketDisconnect:
        pass
```

With `TestClient`, WebSocket tests are already synchronous. For async WebSocket testing, use the same `TestClient` in async fixtures or use `anyio` directly:

```python
# tests/test_ws_async.py
from fastapi.testclient import TestClient
from app.main import app
import pytest_asyncio
import pytest

# WebSocket testing is natively sync via TestClient
def test_websocket_echo_sync():
    with TestClient(app) as client:
        with client.websocket_connect("/ws/async") as ws:
            ws.send_json({"action": "ping"})
            data = ws.receive_json()
            assert data == {"echo": {"action": "ping"}}

# Async WebSocket test using anyio and websockets library
@pytest.mark.anyio
async def test_websocket_echo_async():
    """
    Use ASGITransport is only for HTTP.
    For WS, spawn an actual server with anyio or use TestClient.
    """
    # Approach: use TestClient in an anyio thread
    import anyio

    def sync_ws_test():
        with TestClient(app) as client:
            with client.websocket_connect("/ws/async") as ws:
                ws.send_json({"action": "hello"})
                return ws.receive_json()

    result = await anyio.to_thread.run_sync(sync_ws_test)
    assert result == {"echo": {"action": "hello"}}
```

---

## 12. `AsyncClient` with Authentication Headers

```python
# tests/conftest.py
import pytest_asyncio
import httpx
from app.main import app
from app.auth import create_access_token

@pytest_asyncio.fixture
async def authenticated_async_client():
    token = create_access_token({"sub": "testuser"})
    headers = {"Authorization": f"Bearer {token}"}
    async with httpx.AsyncClient(
        transport=httpx.ASGITransport(app=app),
        base_url="http://test",
        headers=headers,
    ) as client:
        yield client
```

```python
async def test_protected_endpoint(authenticated_async_client):
    response = await authenticated_async_client.get("/users/me")
    assert response.status_code == 200
    assert response.json()["username"] == "testuser"

async def test_override_auth_async():
    from app.auth import get_current_user, User

    mock_user = User(username="async_user", email="async@example.com")
    app.dependency_overrides[get_current_user] = lambda: mock_user

    async with httpx.AsyncClient(
        transport=httpx.ASGITransport(app=app),
        base_url="http://test",
    ) as client:
        response = await client.get("/users/me")

    assert response.status_code == 200
    assert response.json()["username"] == "async_user"
    app.dependency_overrides.clear()
```

---

## 13. Mixing Sync and Async Tests in the Same Suite

Both `TestClient` (sync) and `AsyncClient` (async) tests can coexist. pytest-asyncio handles the event loop per async test transparently.

```python
# tests/test_mixed.py
from fastapi.testclient import TestClient
import httpx
import pytest
from app.main import app

# Synchronous test — uses TestClient
def test_sync_root():
    with TestClient(app) as client:
        response = client.get("/")
        assert response.status_code == 200

# Asynchronous test — uses AsyncClient
@pytest.mark.anyio
async def test_async_root():
    async with httpx.AsyncClient(
        transport=httpx.ASGITransport(app=app),
        base_url="http://test",
    ) as client:
        response = await client.get("/")
    assert response.status_code == 200

# Both share the same fixtures that are sync (conftest's db, etc.)
def test_sync_create_item(client):  # sync client fixture
    response = client.post("/items/", json={"name": "Sync Item", "price": 1.00})
    assert response.status_code == 201

@pytest.mark.anyio
async def test_async_create_item(async_client):  # async client fixture
    response = await async_client.post("/items/", json={"name": "Async Item", "price": 2.00})
    assert response.status_code == 201
```

### Sharing fixtures between sync and async tests

```python
# conftest.py
import pytest
import pytest_asyncio
from fastapi.testclient import TestClient
import httpx
from app.main import app

# Sync fixture usable by both sync and async tests
@pytest.fixture
def base_headers():
    return {"X-Request-ID": "test-123"}

# Sync client fixture
@pytest.fixture
def client(base_headers):
    with TestClient(app, headers=base_headers) as c:
        yield c

# Async client fixture
@pytest_asyncio.fixture
async def async_client(base_headers):
    async with httpx.AsyncClient(
        transport=httpx.ASGITransport(app=app),
        base_url="http://test",
        headers=base_headers,
    ) as c:
        yield c
```

---

## 14. `trio` vs `asyncio` Backends for anyio

FastAPI/Starlette supports both asyncio and trio. Tests can run on either.

| Feature | asyncio | trio |
|---------|---------|------|
| Standard library | Yes (`asyncio`) | No (third-party) |
| Default in FastAPI | Yes | Optional |
| Strict task scoping | No | Yes (nurseries) |
| Cancellation | Manual | Automatic |
| Exception handling | TaskGroup (3.11+) | Nursery |

```python
# Run with trio backend
pip install trio

# conftest.py
@pytest.fixture(params=["asyncio", "trio"])
def anyio_backend(request):
    return request.param

@pytest.mark.anyio
async def test_on_both_backends(anyio_backend, async_client):
    response = await async_client.get("/")
    assert response.status_code == 200
```

Note: Trio backend requires all async libraries in your app to be anyio-compatible (most httpx/starlette internals are).

---

## 15. Complete Async Test Examples

### conftest.py (full async setup)

```python
# tests/conftest.py
"""
Full async test configuration:
- aiosqlite in-memory database
- AsyncSession per test with rollback
- AsyncClient fixtures (auth + unauth)
- anyio_backend set to asyncio
"""
import pytest
import pytest_asyncio
import httpx
from sqlalchemy.ext.asyncio import (
    AsyncSession,
    create_async_engine,
    async_sessionmaker,
)
from sqlalchemy.pool import StaticPool

from app.main import app
from app.database import Base, get_async_db
from app.auth import get_current_user, create_access_token, User
from app.config import get_settings, Settings

TEST_DATABASE_URL = "sqlite+aiosqlite://"

# Session-scoped engine
@pytest_asyncio.fixture(scope="session")
async def engine():
    _engine = create_async_engine(
        TEST_DATABASE_URL,
        connect_args={"check_same_thread": False},
        poolclass=StaticPool,
    )
    async with _engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield _engine
    async with _engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await _engine.dispose()


@pytest_asyncio.fixture
async def async_db(engine) -> AsyncSession:
    """Transactional test session — rolls back after each test."""
    async with engine.begin() as connection:
        session_factory = async_sessionmaker(
            bind=connection,
            expire_on_commit=False,
            class_=AsyncSession,
        )
        async with session_factory() as session:
            yield session
            await session.rollback()


@pytest.fixture(scope="session")
def anyio_backend():
    return "asyncio"


test_settings = Settings(
    database_url=TEST_DATABASE_URL,
    secret_key="test-secret",
    debug=True,
)


@pytest_asyncio.fixture
async def async_client(async_db: AsyncSession):
    """Unauthenticated async client with isolated DB session."""
    async def _override_db():
        yield async_db

    app.dependency_overrides[get_async_db] = _override_db
    app.dependency_overrides[get_settings] = lambda: test_settings

    async with httpx.AsyncClient(
        transport=httpx.ASGITransport(app=app),
        base_url="http://test",
    ) as client:
        yield client

    app.dependency_overrides.clear()


@pytest_asyncio.fixture
async def auth_async_client(async_db: AsyncSession):
    """Authenticated async client (regular user)."""
    mock_user = User(username="testuser", email="test@example.com")

    async def _override_db():
        yield async_db

    app.dependency_overrides[get_async_db] = _override_db
    app.dependency_overrides[get_current_user] = lambda: mock_user
    app.dependency_overrides[get_settings] = lambda: test_settings

    async with httpx.AsyncClient(
        transport=httpx.ASGITransport(app=app),
        base_url="http://test",
    ) as client:
        yield client

    app.dependency_overrides.clear()
```

### tests/test_async_full.py

```python
# tests/test_async_full.py
import pytest
import httpx

# ---- Basic CRUD ----
async def test_create_item(auth_async_client):
    response = await auth_async_client.post(
        "/items/",
        json={"name": "Async Widget", "price": 9.99},
    )
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "Async Widget"
    assert "id" in data

async def test_get_item(auth_async_client):
    # Create
    create_resp = await auth_async_client.post(
        "/items/",
        json={"name": "Target Item", "price": 5.00},
    )
    item_id = create_resp.json()["id"]
    # Read
    response = await auth_async_client.get(f"/items/{item_id}")
    assert response.status_code == 200
    assert response.json()["name"] == "Target Item"

async def test_item_not_found(async_client):
    response = await async_client.get("/items/99999")
    assert response.status_code == 404

# ---- Streaming ----
async def test_stream_events(async_client):
    events = []
    async with async_client.stream("GET", "/events") as response:
        assert response.status_code == 200
        async for line in response.aiter_lines():
            if line.startswith("data:"):
                events.append(line[6:].strip())
    assert len(events) > 0

# ---- Concurrent requests ----
async def test_concurrent_requests(async_client):
    import asyncio
    tasks = [async_client.get("/items/") for _ in range(10)]
    responses = await asyncio.gather(*tasks)
    assert all(r.status_code == 200 for r in responses)

# ---- Auth flow ----
async def test_full_auth_flow(async_client):
    # Register
    reg_response = await async_client.post(
        "/register",
        json={"username": "newuser", "email": "new@example.com", "password": "pass123"},
    )
    assert reg_response.status_code == 201

    # Login
    login_response = await async_client.post(
        "/token",
        data={"username": "newuser", "password": "pass123"},
    )
    assert login_response.status_code == 200
    token = login_response.json()["access_token"]

    # Authenticated request
    me_response = await async_client.get(
        "/users/me",
        headers={"Authorization": f"Bearer {token}"},
    )
    assert me_response.status_code == 200
    assert me_response.json()["username"] == "newuser"
```

---

## 16. Performance Testing with AsyncClient

```python
# tests/test_performance.py
import asyncio
import time
import pytest

async def test_response_time(async_client):
    start = time.perf_counter()
    response = await async_client.get("/items/")
    elapsed = time.perf_counter() - start
    assert response.status_code == 200
    assert elapsed < 0.1  # must respond within 100ms

async def test_throughput(async_client):
    n_requests = 50
    start = time.perf_counter()
    tasks = [async_client.get("/items/") for _ in range(n_requests)]
    responses = await asyncio.gather(*tasks)
    elapsed = time.perf_counter() - start

    successful = sum(1 for r in responses if r.status_code == 200)
    assert successful == n_requests
    rps = n_requests / elapsed
    assert rps > 10  # at least 10 requests/sec
```

---

## 17. Common Pitfalls

### Pitfall 1: Using `TestClient` when fixtures are async

```python
# WRONG — TestClient cannot accept an async override_get_db
async def override_get_db():
    yield fake_session  # this is async, TestClient won't await it

# CORRECT — use sync override with TestClient
def override_get_db():
    yield fake_session
```

### Pitfall 2: Event loop scope mismatch

```python
# WRONG — session-scoped async fixture without matching event_loop scope
@pytest_asyncio.fixture(scope="session")
async def engine():  # needs session-scoped event loop
    ...

# CORRECT — set asyncio_mode = "auto" and configure scope in pyproject.toml
# or use @pytest.mark.asyncio(loop_scope="session")
```

### Pitfall 3: Not clearing dependency_overrides

```python
# WRONG — state bleeds between tests
app.dependency_overrides[get_db] = fake

async def test_a(async_client): ...
async def test_b(async_client): ...  # still uses fake from test_a

# CORRECT — always clear in fixture teardown
@pytest_asyncio.fixture
async def async_client():
    app.dependency_overrides[get_db] = fake
    async with httpx.AsyncClient(...) as c:
        yield c
    app.dependency_overrides.clear()  # <- always runs, even if test fails
```

### Pitfall 4: Forgetting `base_url`

```python
# WRONG — ASGITransport requires an explicit base_url
async with httpx.AsyncClient(transport=httpx.ASGITransport(app=app)) as client:
    await client.get("/")  # raises ValueError: relative URL

# CORRECT
async with httpx.AsyncClient(
    transport=httpx.ASGITransport(app=app),
    base_url="http://test",
) as client:
    await client.get("/")
```

### Pitfall 5: Mixing `asyncio.run` inside tests

```python
# WRONG — creates a new event loop inside an already-running loop
async def test_something():
    result = asyncio.run(some_async_fn())  # RuntimeError

# CORRECT — just await directly
async def test_something():
    result = await some_async_fn()
```
