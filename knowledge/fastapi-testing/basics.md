# FastAPI Testing — Basics

> **Source:** FastAPI official docs (https://fastapi.tiangolo.com/tutorial/testing/), Starlette test client, TestDriven.io FastAPI series.

---

## 1. Installation

```bash
pip install fastapi httpx pytest pytest-asyncio
# or with extras
pip install "fastapi[standard]" httpx pytest pytest-asyncio
```

`httpx` is required because Starlette's `TestClient` is built on top of it (replacing the older `requests`-based client). `pytest-asyncio` is needed when you add async fixtures or async tests later.

---

## 2. TestClient — Synchronous Testing

`TestClient` wraps your ASGI application and drives it synchronously. It starts the app's lifespan (startup/shutdown) when used as a context manager, or on first request when used directly.

```python
# app/main.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {"message": "Hello World"}

@app.get("/items/{item_id}")
def read_item(item_id: int, q: str | None = None):
    return {"item_id": item_id, "q": q}
```

```python
# tests/test_main.py
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_read_root():
    response = client.get("/")
    assert response.status_code == 200
    assert response.json() == {"message": "Hello World"}

def test_read_item():
    response = client.get("/items/42?q=somequery")
    assert response.status_code == 200
    data = response.json()
    assert data["item_id"] == 42
    assert data["q"] == "somequery"
```

### TestClient as a context manager

Using `with TestClient(app) as client:` triggers the app's startup and shutdown lifespan events:

```python
def test_with_lifespan():
    with TestClient(app) as client:
        response = client.get("/")
        assert response.status_code == 200
    # shutdown has run by this point
```

---

## 3. HTTP Method Tests

### GET

```python
def test_get_list():
    response = client.get("/items/")
    assert response.status_code == 200
    assert isinstance(response.json(), list)

def test_get_with_query_params():
    response = client.get("/items/", params={"skip": 0, "limit": 10})
    assert response.status_code == 200
```

### POST (JSON body)

```python
# app/main.py
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    price: float
    is_offer: bool = False

@app.post("/items/", status_code=201)
def create_item(item: Item):
    return item
```

```python
def test_create_item():
    payload = {"name": "Widget", "price": 9.99, "is_offer": True}
    response = client.post("/items/", json=payload)
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "Widget"
    assert data["price"] == 9.99
    assert data["is_offer"] is True

def test_create_item_invalid():
    response = client.post("/items/", json={"name": "BadItem"})  # missing price
    assert response.status_code == 422
    errors = response.json()["detail"]
    assert any(e["loc"] == ["body", "price"] for e in errors)
```

### PUT

```python
@app.put("/items/{item_id}")
def update_item(item_id: int, item: Item):
    return {"item_id": item_id, **item.model_dump()}
```

```python
def test_update_item():
    response = client.put("/items/1", json={"name": "Updated", "price": 19.99})
    assert response.status_code == 200
    assert response.json()["name"] == "Updated"
```

### DELETE

```python
@app.delete("/items/{item_id}", status_code=204)
def delete_item(item_id: int):
    return None
```

```python
def test_delete_item():
    response = client.delete("/items/1")
    assert response.status_code == 204
    assert response.content == b""
```

### PATCH

```python
class ItemUpdate(BaseModel):
    name: str | None = None
    price: float | None = None

@app.patch("/items/{item_id}")
def patch_item(item_id: int, item: ItemUpdate):
    stored = {"item_id": item_id, "name": "Old Name", "price": 5.00}
    update_data = item.model_dump(exclude_unset=True)
    stored.update(update_data)
    return stored
```

```python
def test_patch_item():
    response = client.patch("/items/1", json={"price": 12.50})
    assert response.status_code == 200
    data = response.json()
    assert data["price"] == 12.50
    assert data["name"] == "Old Name"  # unchanged
```

---

## 4. Request Headers and Cookies

```python
def test_with_headers():
    response = client.get("/protected", headers={"X-API-Key": "secret-key"})
    assert response.status_code == 200

def test_with_cookies():
    client.cookies.set("session", "abc123")
    response = client.get("/profile")
    assert response.status_code == 200
```

---

## 5. Dependency Injection Overrides

`app.dependency_overrides` is a dict mapping original dependency callables to replacement callables. FastAPI resolves overrides at test time without touching production code.

```python
# app/dependencies.py
from fastapi import Depends

def get_db():
    # yields a real DB session in production
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

def get_settings():
    return Settings()
```

```python
# tests/test_deps.py
from fastapi.testclient import TestClient
from app.main import app
from app.dependencies import get_db, get_settings

# --- fake DB session ---
class FakeDB:
    def __init__(self):
        self.store = {}

    def query(self, model):
        return list(self.store.get(model.__name__, {}).values())

    def add(self, obj):
        cls = type(obj).__name__
        self.store.setdefault(cls, {})[id(obj)] = obj

    def commit(self): pass
    def refresh(self, obj): pass
    def close(self): pass


fake_db = FakeDB()

def override_get_db():
    yield fake_db

app.dependency_overrides[get_db] = override_get_db

client = TestClient(app)

def test_create_user_with_fake_db():
    response = client.post("/users/", json={"username": "alice", "email": "alice@example.com"})
    assert response.status_code == 201
```

### Cleaning up overrides

Always restore overrides between tests to avoid state leakage:

```python
import pytest
from fastapi.testclient import TestClient
from app.main import app
from app.dependencies import get_db

@pytest.fixture
def client_with_fake_db():
    fake = FakeDB()
    app.dependency_overrides[get_db] = lambda: fake
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()
```

---

## 6. Overriding Database Dependencies

### SQLAlchemy (sync)

```python
# app/database.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, DeclarativeBase

SQLALCHEMY_DATABASE_URL = "postgresql://user:pass@localhost/mydb"
engine = create_engine(SQLALCHEMY_DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

class Base(DeclarativeBase):
    pass

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

```python
# tests/conftest.py
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from fastapi.testclient import TestClient

from app.main import app
from app.database import Base, get_db

SQLALCHEMY_TEST_URL = "sqlite:///./test.db"

engine = create_engine(
    SQLALCHEMY_TEST_URL,
    connect_args={"check_same_thread": False},  # SQLite only
)
TestingSessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)


@pytest.fixture(scope="session", autouse=True)
def create_tables():
    Base.metadata.create_all(bind=engine)
    yield
    Base.metadata.drop_all(bind=engine)


@pytest.fixture
def db_session():
    connection = engine.connect()
    transaction = connection.begin()
    session = TestingSessionLocal(bind=connection)
    yield session
    session.close()
    transaction.rollback()
    connection.close()


@pytest.fixture
def client(db_session):
    def override_get_db():
        yield db_session

    app.dependency_overrides[get_db] = override_get_db
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()
```

This pattern creates a savepoint per test and rolls back, giving each test a clean state without truncating tables.

### Using pytest-factoryboy or factory_boy for test data

```python
# tests/factories.py
import factory
from factory.alchemy import SQLAlchemyModelFactory
from app.models import User
from tests.conftest import TestingSessionLocal

class UserFactory(SQLAlchemyModelFactory):
    class Meta:
        model = User
        sqlalchemy_session = TestingSessionLocal()
        sqlalchemy_session_persistence = "commit"

    username = factory.Sequence(lambda n: f"user{n}")
    email = factory.LazyAttribute(lambda obj: f"{obj.username}@example.com")
    is_active = True
```

---

## 7. Authentication Testing

### 7.1 API Key Testing

```python
# app/auth.py
from fastapi import Security, HTTPException, status
from fastapi.security import APIKeyHeader

API_KEY_NAME = "X-API-Key"
API_KEY = "supersecret"

api_key_header = APIKeyHeader(name=API_KEY_NAME, auto_error=False)

async def get_api_key(api_key: str = Security(api_key_header)):
    if api_key == API_KEY:
        return api_key
    raise HTTPException(status_code=403, detail="Invalid API Key")
```

```python
# tests/test_api_key.py
from fastapi.testclient import TestClient
from app.main import app
from app.auth import get_api_key

client = TestClient(app)

def test_valid_api_key():
    response = client.get("/secure-endpoint", headers={"X-API-Key": "supersecret"})
    assert response.status_code == 200

def test_missing_api_key():
    response = client.get("/secure-endpoint")
    assert response.status_code == 403

def test_invalid_api_key():
    response = client.get("/secure-endpoint", headers={"X-API-Key": "wrong"})
    assert response.status_code == 403

# Override approach — bypass auth entirely in tests
def test_bypass_api_key():
    app.dependency_overrides[get_api_key] = lambda: "test-key"
    response = client.get("/secure-endpoint")
    assert response.status_code == 200
    app.dependency_overrides.clear()
```

### 7.2 JWT / OAuth2 Bearer Token Testing

```python
# app/auth.py
from datetime import datetime, timedelta, timezone
from typing import Annotated
import jwt
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from pydantic import BaseModel

SECRET_KEY = "test-secret-key"
ALGORITHM = "HS256"

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

class User(BaseModel):
    username: str
    email: str
    disabled: bool = False

def create_access_token(data: dict, expires_delta: timedelta | None = None) -> str:
    to_encode = data.copy()
    expire = datetime.now(timezone.utc) + (expires_delta or timedelta(minutes=15))
    to_encode["exp"] = expire
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

async def get_current_user(token: Annotated[str, Depends(oauth2_scheme)]) -> User:
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise credentials_exception
    except jwt.InvalidTokenError:
        raise credentials_exception
    # look up user in DB ...
    return User(username=username, email=f"{username}@example.com")
```

```python
# tests/test_jwt.py
from fastapi.testclient import TestClient
from app.main import app
from app.auth import create_access_token, get_current_user, User

client = TestClient(app)

def get_test_token(username: str = "testuser") -> str:
    return create_access_token({"sub": username})

def test_protected_route_with_valid_token():
    token = get_test_token()
    response = client.get("/me", headers={"Authorization": f"Bearer {token}"})
    assert response.status_code == 200
    assert response.json()["username"] == "testuser"

def test_protected_route_with_no_token():
    response = client.get("/me")
    assert response.status_code == 401

def test_protected_route_with_invalid_token():
    response = client.get("/me", headers={"Authorization": "Bearer invalid.token.here"})
    assert response.status_code == 401
```

### 7.3 Mocking `get_current_user`

This is the cleanest approach: override the dependency to return a known user object, bypassing all JWT parsing.

```python
# tests/test_auth_override.py
import pytest
from fastapi.testclient import TestClient
from app.main import app
from app.auth import get_current_user, User

MOCK_USER = User(username="alice", email="alice@example.com")

@pytest.fixture
def authenticated_client():
    app.dependency_overrides[get_current_user] = lambda: MOCK_USER
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()

def test_get_profile(authenticated_client):
    response = authenticated_client.get("/users/me")
    assert response.status_code == 200
    assert response.json()["username"] == "alice"

def test_admin_required(authenticated_client):
    # Test that a non-admin user gets 403
    response = authenticated_client.get("/admin/dashboard")
    assert response.status_code == 403
```

For tests requiring different user roles, parameterize:

```python
@pytest.fixture
def admin_client():
    admin = User(username="admin", email="admin@example.com", is_superuser=True)
    app.dependency_overrides[get_current_user] = lambda: admin
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()

def test_admin_endpoint(admin_client):
    response = admin_client.get("/admin/dashboard")
    assert response.status_code == 200
```

### 7.4 Testing the Token Endpoint Itself

```python
def test_login_for_access_token():
    response = client.post(
        "/token",
        data={"username": "alice", "password": "secret"},  # form data, not JSON
        headers={"Content-Type": "application/x-www-form-urlencoded"},
    )
    assert response.status_code == 200
    token_data = response.json()
    assert "access_token" in token_data
    assert token_data["token_type"] == "bearer"

def test_login_wrong_password():
    response = client.post(
        "/token",
        data={"username": "alice", "password": "wrong"},
    )
    assert response.status_code == 401
```

---

## 8. File Upload Testing

```python
# app/main.py
from fastapi import UploadFile, File

@app.post("/upload/")
async def upload_file(file: UploadFile = File(...)):
    content = await file.read()
    return {
        "filename": file.filename,
        "content_type": file.content_type,
        "size": len(content),
    }

@app.post("/upload-multiple/")
async def upload_multiple(files: list[UploadFile] = File(...)):
    return [{"filename": f.filename, "size": (await f.read()).__len__()} for f in files]
```

```python
# tests/test_upload.py
import io
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_upload_file():
    file_content = b"hello file content"
    response = client.post(
        "/upload/",
        files={"file": ("test.txt", io.BytesIO(file_content), "text/plain")},
    )
    assert response.status_code == 200
    data = response.json()
    assert data["filename"] == "test.txt"
    assert data["content_type"] == "text/plain"
    assert data["size"] == len(file_content)

def test_upload_image():
    # Minimal valid PNG (1x1 pixel)
    png_bytes = (
        b"\x89PNG\r\n\x1a\n\x00\x00\x00\rIHDR\x00\x00\x00\x01"
        b"\x00\x00\x00\x01\x08\x02\x00\x00\x00\x90wS\xde\x00\x00"
        b"\x00\x0cIDATx\x9cc\xf8\x0f\x00\x00\x01\x01\x00\x05\x18"
        b"\xd8N\x00\x00\x00\x00IEND\xaeB`\x82"
    )
    response = client.post(
        "/upload/",
        files={"file": ("photo.png", io.BytesIO(png_bytes), "image/png")},
    )
    assert response.status_code == 200
    assert response.json()["filename"] == "photo.png"

def test_upload_multiple_files():
    files = [
        ("files", ("a.txt", io.BytesIO(b"file a"), "text/plain")),
        ("files", ("b.txt", io.BytesIO(b"file b"), "text/plain")),
    ]
    response = client.post("/upload-multiple/", files=files)
    assert response.status_code == 200
    data = response.json()
    assert len(data) == 2
    assert data[0]["filename"] == "a.txt"

def test_upload_with_form_fields():
    # Mixed form data + file
    response = client.post(
        "/upload-with-meta/",
        data={"description": "A test file"},
        files={"file": ("doc.pdf", io.BytesIO(b"%PDF-1.4"), "application/pdf")},
    )
    assert response.status_code == 200
```

---

## 9. WebSocket Testing

```python
# app/main.py
from fastapi import WebSocket

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    while True:
        data = await websocket.receive_text()
        await websocket.send_text(f"Echo: {data}")

@app.websocket("/ws/json")
async def websocket_json(websocket: WebSocket):
    await websocket.accept()
    data = await websocket.receive_json()
    await websocket.send_json({"received": data, "processed": True})
```

```python
# tests/test_websocket.py
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_websocket_echo():
    with client.websocket_connect("/ws") as ws:
        ws.send_text("hello")
        data = ws.receive_text()
        assert data == "Echo: hello"

def test_websocket_json():
    with client.websocket_connect("/ws/json") as ws:
        ws.send_json({"action": "ping"})
        response = ws.receive_json()
        assert response["processed"] is True
        assert response["received"]["action"] == "ping"

def test_websocket_multiple_messages():
    with client.websocket_connect("/ws") as ws:
        for i in range(3):
            ws.send_text(f"message {i}")
            reply = ws.receive_text()
            assert reply == f"Echo: message {i}"

def test_websocket_auth():
    # Pass token as query param
    with client.websocket_connect("/ws/auth?token=valid-token") as ws:
        ws.send_text("hello")
        assert ws.receive_text() == "Echo: hello"

def test_websocket_close():
    with client.websocket_connect("/ws") as ws:
        ws.close()
        # WebSocketDisconnect is raised server-side, handled gracefully
```

---

## 10. Background Tasks Testing

Background tasks run in the same thread during tests (TestClient blocks until they complete).

```python
# app/main.py
from fastapi import BackgroundTasks

def write_log(message: str):
    with open("log.txt", "a") as f:
        f.write(f"{message}\n")

@app.post("/send-notification/")
async def send_notification(background_tasks: BackgroundTasks, email: str):
    background_tasks.add_task(write_log, f"Notification sent to {email}")
    return {"message": "Notification will be sent"}
```

```python
# tests/test_background.py
import os
from unittest.mock import MagicMock, patch
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_background_task_runs(tmp_path, monkeypatch):
    log_file = tmp_path / "log.txt"
    # Patch open() to write to tmp file
    original_open = open

    def patched_open(file, mode="r", **kwargs):
        if file == "log.txt":
            return original_open(str(log_file), mode, **kwargs)
        return original_open(file, mode, **kwargs)

    with patch("builtins.open", patched_open):
        response = client.post("/send-notification/?email=test@example.com")

    assert response.status_code == 200
    # Background task has already run by the time TestClient returns
    assert "test@example.com" in log_file.read_text()

def test_background_task_called():
    with patch("app.main.write_log") as mock_log:
        response = client.post("/send-notification/?email=user@example.com")
        assert response.status_code == 200
        mock_log.assert_called_once_with("Notification sent to user@example.com")
```

### Testing background tasks with dependency injection

```python
# app/main.py
from app.services import EmailService

@app.post("/email/")
async def send_email(
    background_tasks: BackgroundTasks,
    to: str,
    email_service: EmailService = Depends(get_email_service),
):
    background_tasks.add_task(email_service.send, to=to, subject="Hello")
    return {"queued": True}
```

```python
def test_email_background_task():
    mock_service = MagicMock(spec=EmailService)
    app.dependency_overrides[get_email_service] = lambda: mock_service

    response = client.post("/email/?to=alice@example.com")
    assert response.status_code == 200
    mock_service.send.assert_called_once_with(to="alice@example.com", subject="Hello")

    app.dependency_overrides.clear()
```

---

## 11. Startup and Shutdown Events in Tests

### Using lifespan (modern, recommended)

```python
# app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI

db_pool = {}

@asynccontextmanager
async def lifespan(app: FastAPI):
    # startup
    db_pool["conn"] = await create_db_connection()
    yield
    # shutdown
    await db_pool["conn"].close()
    db_pool.clear()

app = FastAPI(lifespan=lifespan)
```

```python
# tests/test_lifespan.py
from unittest.mock import AsyncMock, patch
from fastapi.testclient import TestClient
from app.main import app

def test_lifespan_startup_shutdown():
    with patch("app.main.create_db_connection", new_callable=AsyncMock) as mock_connect:
        mock_conn = AsyncMock()
        mock_connect.return_value = mock_conn

        with TestClient(app) as client:
            # startup has run
            response = client.get("/health")
            assert response.status_code == 200

        # shutdown has run
        mock_conn.close.assert_called_once()
```

### Overriding lifespan in tests

```python
# tests/conftest.py
from contextlib import asynccontextmanager
import pytest
from fastapi.testclient import TestClient
from app.main import app as real_app
from fastapi import FastAPI

@pytest.fixture
def test_app():
    # Replace lifespan with a no-op for unit tests
    @asynccontextmanager
    async def test_lifespan(app):
        app.state.db = FakeDB()
        yield
        pass

    # Mount a fresh FastAPI instance with the test lifespan
    test_application = FastAPI(lifespan=test_lifespan)
    # include routers from real app
    for route in real_app.routes:
        test_application.routes.append(route)

    with TestClient(test_application) as client:
        yield client
```

### Using deprecated @app.on_event (older pattern)

```python
# app/main.py
@app.on_event("startup")
async def startup_event():
    app.state.items = []

@app.on_event("shutdown")
async def shutdown_event():
    app.state.items.clear()
```

```python
def test_startup_state():
    with TestClient(app) as client:
        response = client.get("/items/")
        assert response.status_code == 200
```

---

## 12. Response Validation Patterns

```python
from pydantic import BaseModel

class ItemResponse(BaseModel):
    id: int
    name: str
    price: float

def test_response_schema():
    response = client.get("/items/1")
    assert response.status_code == 200
    # Validate response matches schema
    item = ItemResponse(**response.json())
    assert item.id == 1

def test_response_headers():
    response = client.get("/items/")
    assert response.headers["content-type"] == "application/json"
    assert "x-request-id" in response.headers

def test_response_list_structure():
    response = client.get("/items/")
    data = response.json()
    assert isinstance(data, list)
    assert all("id" in item and "name" in item for item in data)

def test_pagination_response():
    response = client.get("/items/?page=1&size=10")
    data = response.json()
    assert "items" in data
    assert "total" in data
    assert "page" in data
    assert "size" in data
    assert len(data["items"]) <= 10

def test_empty_list():
    response = client.get("/items/")
    assert response.status_code == 200
    assert response.json() == []
```

---

## 13. Error Handlers and Exception Handlers

```python
# app/main.py
from fastapi import Request
from fastapi.responses import JSONResponse

class ItemNotFoundError(Exception):
    def __init__(self, item_id: int):
        self.item_id = item_id

@app.exception_handler(ItemNotFoundError)
async def item_not_found_handler(request: Request, exc: ItemNotFoundError):
    return JSONResponse(
        status_code=404,
        content={"detail": f"Item {exc.item_id} not found"},
    )

@app.exception_handler(500)
async def internal_error_handler(request: Request, exc: Exception):
    return JSONResponse(
        status_code=500,
        content={"detail": "Internal server error"},
    )
```

```python
# tests/test_errors.py
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_item_not_found():
    response = client.get("/items/99999")
    assert response.status_code == 404
    assert "99999" in response.json()["detail"]

def test_validation_error_format():
    response = client.post("/items/", json={"name": 123, "price": "not-a-float"})
    assert response.status_code == 422
    errors = response.json()["detail"]
    assert isinstance(errors, list)
    # Each error has loc, msg, type
    for error in errors:
        assert "loc" in error
        assert "msg" in error
        assert "type" in error

def test_custom_exception_handler():
    # Trigger the custom exception handler
    response = client.get("/items/0")  # item_id=0 raises ItemNotFoundError
    assert response.status_code == 404
    assert response.json()["detail"] == "Item 0 not found"

def test_raise_http_exception():
    with TestClient(app, raise_server_exceptions=False) as c:
        response = c.get("/cause-500")
        assert response.status_code == 500
```

---

## 14. Environment Variable Override in Tests

### Using monkeypatch (pytest built-in)

```python
# app/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str = "sqlite:///./default.db"
    secret_key: str = "default-secret"
    debug: bool = False

    class Config:
        env_file = ".env"

def get_settings() -> Settings:
    return Settings()
```

```python
# tests/test_config.py
import pytest
from fastapi.testclient import TestClient
from app.main import app
from app.config import get_settings, Settings

@pytest.fixture
def override_settings(monkeypatch):
    monkeypatch.setenv("DATABASE_URL", "sqlite:///./test.db")
    monkeypatch.setenv("SECRET_KEY", "test-secret")
    monkeypatch.setenv("DEBUG", "true")

def test_settings_from_env(override_settings):
    settings = get_settings()
    assert settings.database_url == "sqlite:///./test.db"
    assert settings.debug is True

# Override via dependency
def test_with_test_settings():
    test_settings = Settings(
        database_url="sqlite:///./test.db",
        secret_key="test-secret",
        debug=True,
    )
    app.dependency_overrides[get_settings] = lambda: test_settings
    with TestClient(app) as client:
        response = client.get("/config")
        assert response.status_code == 200
    app.dependency_overrides.clear()
```

### Using `unittest.mock.patch`

```python
from unittest.mock import patch

def test_with_patched_env():
    with patch.dict("os.environ", {"DATABASE_URL": "sqlite:///./test.db"}):
        # settings re-loaded inside this block will use the patched env
        response = client.get("/health")
        assert response.status_code == 200
```

---

## 15. Complete `conftest.py` Example

```python
# tests/conftest.py
"""
Comprehensive conftest.py for a FastAPI application with:
- SQLAlchemy database
- JWT authentication
- Settings override
"""
import pytest
from datetime import timedelta
from typing import Generator

from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session
from sqlalchemy.pool import StaticPool

from app.main import app
from app.database import Base, get_db
from app.auth import get_current_user, create_access_token, User
from app.config import get_settings, Settings

# ---------------------------------------------------------------------------
# Test database setup
# ---------------------------------------------------------------------------
SQLALCHEMY_TEST_URL = "sqlite://"  # in-memory

engine = create_engine(
    SQLALCHEMY_TEST_URL,
    connect_args={"check_same_thread": False},
    poolclass=StaticPool,  # single connection for in-memory SQLite
)
TestingSessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)


@pytest.fixture(scope="session", autouse=True)
def create_test_tables():
    """Create all tables once per test session."""
    Base.metadata.create_all(bind=engine)
    yield
    Base.metadata.drop_all(bind=engine)


@pytest.fixture
def db() -> Generator[Session, None, None]:
    """
    Provide a transactional test database session.
    Each test gets a fresh transaction, rolled back afterwards.
    """
    connection = engine.connect()
    transaction = connection.begin()
    session = TestingSessionLocal(bind=connection)
    yield session
    session.close()
    transaction.rollback()
    connection.close()


# ---------------------------------------------------------------------------
# Settings override
# ---------------------------------------------------------------------------
test_settings = Settings(
    database_url="sqlite://",
    secret_key="test-secret-key-for-jwt",
    debug=True,
)


# ---------------------------------------------------------------------------
# Client fixtures
# ---------------------------------------------------------------------------
@pytest.fixture
def client(db: Session) -> Generator[TestClient, None, None]:
    """Unauthenticated test client with isolated DB session."""
    app.dependency_overrides[get_db] = lambda: db
    app.dependency_overrides[get_settings] = lambda: test_settings
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()


@pytest.fixture
def auth_client(db: Session) -> Generator[TestClient, None, None]:
    """Authenticated test client as a regular user."""
    mock_user = User(username="testuser", email="test@example.com")
    app.dependency_overrides[get_db] = lambda: db
    app.dependency_overrides[get_current_user] = lambda: mock_user
    app.dependency_overrides[get_settings] = lambda: test_settings
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()


@pytest.fixture
def admin_client(db: Session) -> Generator[TestClient, None, None]:
    """Authenticated test client as an admin user."""
    admin_user = User(username="admin", email="admin@example.com", is_superuser=True)
    app.dependency_overrides[get_db] = lambda: db
    app.dependency_overrides[get_current_user] = lambda: admin_user
    app.dependency_overrides[get_settings] = lambda: test_settings
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()


@pytest.fixture
def token_headers() -> dict[str, str]:
    """Return Authorization headers with a valid JWT."""
    token = create_access_token(
        {"sub": "testuser"},
        expires_delta=timedelta(minutes=30),
    )
    return {"Authorization": f"Bearer {token}"}


# ---------------------------------------------------------------------------
# Common test data fixtures
# ---------------------------------------------------------------------------
@pytest.fixture
def sample_item(db: Session):
    """Insert a sample item and return it."""
    from app.models import Item
    item = Item(name="Test Widget", price=9.99, owner_id=1)
    db.add(item)
    db.commit()
    db.refresh(item)
    return item


@pytest.fixture
def sample_user(db: Session):
    """Insert a sample user and return it."""
    from app.models import User as UserModel
    from app.auth import get_password_hash
    user = UserModel(
        username="fixture_user",
        email="fixture@example.com",
        hashed_password=get_password_hash("password123"),
    )
    db.add(user)
    db.commit()
    db.refresh(user)
    return user
```

### Example test using the full conftest

```python
# tests/test_items.py
def test_create_item(auth_client):
    response = auth_client.post("/items/", json={"name": "New Item", "price": 5.00})
    assert response.status_code == 201
    assert response.json()["name"] == "New Item"

def test_list_items(auth_client, sample_item):
    response = auth_client.get("/items/")
    assert response.status_code == 200
    items = response.json()
    assert any(i["name"] == "Test Widget" for i in items)

def test_admin_only_endpoint(client, auth_client, admin_client):
    # Unauthenticated
    assert client.get("/admin/").status_code == 401
    # Regular user
    assert auth_client.get("/admin/").status_code == 403
    # Admin
    assert admin_client.get("/admin/").status_code == 200
```

---

## 16. pytest.ini / pyproject.toml Configuration

```toml
# pyproject.toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py", "*_test.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = "-v --tb=short"
filterwarnings = [
    "ignore::DeprecationWarning",
]
```

```ini
# pytest.ini
[pytest]
testpaths = tests
addopts = -v --strict-markers
markers =
    slow: marks tests as slow
    integration: marks tests as integration tests
    unit: marks tests as unit tests
```
