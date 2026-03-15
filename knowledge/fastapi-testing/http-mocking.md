# FastAPI Testing — HTTP Mocking

> **Sources:** respx docs (https://lundberg.github.io/respx/), responses library (https://github.com/getsentry/responses), pytest-httpserver docs (https://pytest-httpserver.readthedocs.io/).

This guide covers three complementary HTTP mocking libraries for use alongside FastAPI tests:

| Library | Intercepts | Best for |
|---------|-----------|---------|
| **respx** | `httpx` (sync + async) | FastAPI services that call external APIs via httpx |
| **responses** | `requests` | Services using the `requests` library |
| **pytest-httpserver** | Real TCP | When you need a real server on a real port |

---

## Part 1: respx (httpx Mocking)

`respx` intercepts `httpx.Client` and `httpx.AsyncClient` calls. Since FastAPI itself uses httpx internally (via `TestClient`), respx is the natural choice for mocking outbound HTTP from FastAPI services.

```bash
pip install respx
```

---

### 1.1 `@respx.mock` Decorator

```python
import respx
import httpx

@respx.mock
def test_mock_decorator():
    respx.get("https://api.example.com/users").mock(
        return_value=httpx.Response(200, json=[{"id": 1, "name": "Alice"}])
    )
    response = httpx.get("https://api.example.com/users")
    assert response.status_code == 200
    assert response.json()[0]["name"] == "Alice"
```

### 1.2 `with respx.mock:` Context Manager

```python
def test_mock_context_manager():
    with respx.mock:
        respx.post("https://api.example.com/items").mock(
            return_value=httpx.Response(201, json={"id": 42, "name": "Widget"})
        )
        response = httpx.post("https://api.example.com/items", json={"name": "Widget"})
        assert response.status_code == 201
        assert response.json()["id"] == 42
```

### 1.3 pytest Fixture: `respx_mock`

`respx` ships a built-in pytest fixture that automatically resets state after each test:

```python
import pytest

def test_with_fixture(respx_mock):
    respx_mock.get("https://api.example.com/data").mock(
        return_value=httpx.Response(200, json={"value": 42})
    )
    response = httpx.get("https://api.example.com/data")
    assert response.json()["value"] == 42
    # No cleanup needed — respx_mock resets after the test
```

---

### 1.4 HTTP Method Routing

```python
with respx.mock:
    # GET
    respx.get("https://api.example.com/resource").respond(200, json={"ok": True})

    # POST
    respx.post("https://api.example.com/resource").respond(201, json={"created": True})

    # PUT
    respx.put("https://api.example.com/resource/1").respond(200)

    # PATCH
    respx.patch("https://api.example.com/resource/1").respond(200)

    # DELETE
    respx.delete("https://api.example.com/resource/1").respond(204)

    # HEAD / OPTIONS
    respx.head("https://api.example.com/resource").respond(200)
    respx.options("https://api.example.com/resource").respond(200)

    # Any method
    respx.route(method="GET", url="https://api.example.com/any").respond(200)
```

---

### 1.5 URL Matching

#### Exact URL

```python
respx.get("https://api.example.com/users/42").respond(200, json={"id": 42})
```

#### URL pattern (using `url__regex`)

```python
import re

respx.get(url__regex=r"https://api\.example\.com/users/\d+").respond(
    200, json={"id": 1}
)
```

#### URL with `httpx.URL`

```python
respx.get(httpx.URL("https://api.example.com/users")).respond(200)
```

#### Partial matching with `url__startswith`

```python
respx.get(url__startswith="https://api.example.com/").respond(200)
```

#### Matching path only (on a MockRouter)

```python
router = respx.MockRouter(base_url="https://api.example.com")
router.get("/users/").respond(200, json=[])
router.get("/users/1").respond(200, json={"id": 1})
```

---

### 1.6 Request Matchers

Match requests by headers, query params, JSON body, form data, or files:

```python
# Match by headers
respx.get(
    "https://api.example.com/data",
    headers={"Authorization": "Bearer my-token"},
).respond(200, json={"secret": "data"})

# Match by query params
respx.get(
    "https://api.example.com/search",
    params={"q": "fastapi", "page": "1"},
).respond(200, json={"results": []})

# Match by JSON content
respx.post(
    "https://api.example.com/users",
    json={"username": "alice"},
).respond(201, json={"id": 1, "username": "alice"})

# Match by form data
respx.post(
    "https://api.example.com/login",
    data={"username": "alice", "password": "secret"},
).respond(200, json={"token": "abc123"})

# Match by raw content / bytes
respx.post(
    "https://api.example.com/upload",
    content=b"raw bytes here",
).respond(200)

# Combine matchers
respx.post(
    "https://api.example.com/users",
    headers={"Content-Type": "application/json"},
    json={"role": "admin"},
).respond(201)
```

---

### 1.7 Response Definition

#### `.respond()` shorthand

```python
route = respx.get("https://api.example.com/data")

# Status code only
route.respond(204)

# With JSON body
route.respond(200, json={"key": "value"})

# With text body
route.respond(200, text="plain text response")

# With HTML
route.respond(200, html="<html><body>Hello</body></html>")

# With raw bytes
route.respond(200, content=b"\x00\x01\x02")

# With custom headers
route.respond(200, json={"ok": True}, headers={"X-Custom": "header-value"})
```

#### `.mock(return_value=)` explicit

```python
respx.get("https://api.example.com/data").mock(
    return_value=httpx.Response(
        status_code=200,
        json={"data": "value"},
        headers={"content-type": "application/json"},
    )
)
```

---

### 1.8 Side Effects

#### Raising exceptions

```python
# Connection refused
respx.get("https://api.example.com/down").mock(
    side_effect=httpx.ConnectError
)

# Timeout
respx.get("https://api.example.com/slow").mock(
    side_effect=httpx.TimeoutException
)

# Read timeout specifically
respx.get("https://api.example.com/read-timeout").mock(
    side_effect=httpx.ReadTimeout
)
```

```python
def test_service_handles_connection_error(respx_mock):
    respx_mock.get("https://external.api/data").mock(
        side_effect=httpx.ConnectError("Connection refused")
    )
    # Your FastAPI endpoint calls external API; verify fallback behaviour
    response = client.get("/proxy/data")
    assert response.status_code == 503
    assert response.json()["detail"] == "External service unavailable"
```

#### Custom callback as side effect

```python
import httpx

def custom_handler(request: httpx.Request) -> httpx.Response:
    user_id = request.url.path.split("/")[-1]
    return httpx.Response(200, json={"id": user_id, "name": f"User {user_id}"})

respx.get(url__regex=r"/users/\d+").mock(side_effect=custom_handler)
```

#### Rotating / sequence of responses

```python
# Respond differently on each call
respx.get("https://api.example.com/data").mock(
    side_effect=[
        httpx.Response(200, json={"attempt": 1}),
        httpx.Response(503, json={"error": "down"}),
        httpx.Response(200, json={"attempt": 3}),
    ]
)

# First call → 200, second call → 503, third call → 200
# Further calls raise StopIteration
```

---

### 1.9 Call Assertions

```python
with respx.mock:
    route = respx.get("https://api.example.com/users").respond(200, json=[])

    httpx.get("https://api.example.com/users")
    httpx.get("https://api.example.com/users")

    # Check call count
    assert route.called
    assert route.call_count == 2

    # Inspect individual calls
    last_request = route.calls.last.request
    assert last_request.method == "GET"

    # All calls
    for call in route.calls:
        assert call.request.url.host == "api.example.com"
```

#### `assert_all_called()` — every route must have been called

```python
with respx.mock(assert_all_called=True) as mock:
    mock.get("https://api.example.com/a").respond(200)
    mock.get("https://api.example.com/b").respond(200)

    httpx.get("https://api.example.com/a")
    # Test will FAIL if /b was not called
```

#### `assert_all_mocked` — disallow unmocked calls

```python
with respx.mock(assert_all_mocked=True):
    # Any unmocked request raises httpx.ConnectError
    httpx.get("https://not-mocked.com")  # raises ConnectError
```

---

### 1.10 `MockRouter` for Reusable Routers

```python
# tests/mocks/external_api.py
import respx
import httpx

external_api = respx.MockRouter(base_url="https://api.example.com", assert_all_mocked=False)

external_api.get("/users/").respond(200, json=[{"id": 1, "name": "Alice"}])
external_api.get(url__regex=r"/users/\d+").mock(
    side_effect=lambda req: httpx.Response(
        200, json={"id": int(req.url.path.split("/")[-1])}
    )
)
external_api.post("/users/").respond(201, json={"id": 99})
```

```python
# tests/test_with_router.py
import pytest
from tests.mocks.external_api import external_api
from app.main import app
from fastapi.testclient import TestClient

client = TestClient(app)

@pytest.fixture(autouse=True)
def mock_external(respx_mock):
    # Merge the predefined router into the test mock
    for route in external_api.routes:
        respx_mock.routes.append(route)

def test_list_users():
    response = client.get("/users/")  # your endpoint calls external_api internally
    assert response.status_code == 200
```

---

### 1.11 Async Support

respx works identically with `httpx.AsyncClient`:

```python
import httpx
import respx
import pytest

@pytest.mark.anyio
@respx.mock
async def test_async_external_call():
    respx.get("https://api.example.com/data").respond(200, json={"async": True})
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/data")
    assert response.json()["async"] is True

# With pytest fixture
@pytest.mark.anyio
async def test_async_with_fixture(respx_mock):
    respx_mock.get("https://api.example.com/data").respond(200, json={"value": 1})
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/data")
    assert response.json()["value"] == 1
```

---

### 1.12 Integration with FastAPI Services

```python
# app/services/weather.py
import httpx

async def get_weather(city: str) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.get(
            f"https://api.weather.com/v1/current",
            params={"city": city, "units": "metric"},
            headers={"X-API-Key": "production-key"},
        )
        response.raise_for_status()
        return response.json()
```

```python
# app/main.py
from fastapi import FastAPI, HTTPException
from app.services.weather import get_weather

app = FastAPI()

@app.get("/weather/{city}")
async def weather_endpoint(city: str):
    try:
        return await get_weather(city)
    except httpx.HTTPStatusError as e:
        raise HTTPException(status_code=e.response.status_code, detail="Weather API error")
    except httpx.ConnectError:
        raise HTTPException(status_code=503, detail="Weather service unavailable")
```

```python
# tests/test_weather.py
import respx
import httpx
import pytest
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

@respx.mock
def test_get_weather_success():
    respx.get(
        "https://api.weather.com/v1/current",
        params={"city": "London", "units": "metric"},
    ).respond(200, json={"temperature": 15.2, "condition": "Cloudy"})

    response = client.get("/weather/London")
    assert response.status_code == 200
    assert response.json()["temperature"] == 15.2

@respx.mock
def test_get_weather_api_error():
    respx.get(url__startswith="https://api.weather.com/").respond(500)
    response = client.get("/weather/London")
    assert response.status_code == 500

@respx.mock
def test_get_weather_unavailable():
    respx.get(url__startswith="https://api.weather.com/").mock(
        side_effect=httpx.ConnectError("Cannot connect")
    )
    response = client.get("/weather/London")
    assert response.status_code == 503
    assert response.json()["detail"] == "Weather service unavailable"
```

---

---

## Part 2: responses (requests Library Mocking)

`responses` intercepts the Python `requests` library. Use it when your FastAPI service uses `requests` instead of `httpx`.

```bash
pip install responses
```

---

### 2.1 `@responses.activate` Decorator

```python
import responses
import requests

@responses.activate
def test_requests_get():
    responses.add(
        responses.GET,
        "https://api.example.com/users",
        json=[{"id": 1, "name": "Alice"}],
        status=200,
    )
    response = requests.get("https://api.example.com/users")
    assert response.status_code == 200
    assert response.json()[0]["name"] == "Alice"
```

### 2.2 `responses.RequestsMock` Context Manager

```python
def test_requests_context_manager():
    with responses.RequestsMock() as rsps:
        rsps.add(
            responses.POST,
            "https://api.example.com/items",
            json={"id": 42},
            status=201,
        )
        response = requests.post("https://api.example.com/items", json={"name": "Widget"})
        assert response.status_code == 201
        assert response.json()["id"] == 42
```

---

### 2.3 Adding Responses — All Methods

```python
@responses.activate
def test_all_methods():
    responses.get("https://api.example.com/r", json={"method": "GET"})
    responses.post("https://api.example.com/r", json={"method": "POST"}, status=201)
    responses.put("https://api.example.com/r/1", json={"method": "PUT"})
    responses.patch("https://api.example.com/r/1", json={"method": "PATCH"})
    responses.delete("https://api.example.com/r/1", status=204)
    responses.head("https://api.example.com/r", headers={"X-Count": "42"})
    responses.options("https://api.example.com/r", headers={"Allow": "GET,POST"})

    assert requests.get("https://api.example.com/r").json()["method"] == "GET"
    assert requests.post("https://api.example.com/r").status_code == 201
```

---

### 2.4 URL Matching

#### Exact URL

```python
responses.add(responses.GET, "https://api.example.com/users", json=[])
```

#### URL with query string matching

```python
responses.add(
    responses.GET,
    "https://api.example.com/search",
    match=[responses.matchers.query_param_matcher({"q": "test", "page": "1"})],
    json={"results": []},
)
```

#### Regex URL pattern

```python
import re
responses.add(
    responses.GET,
    re.compile(r"https://api\.example\.com/users/\d+"),
    json={"id": 1, "name": "Matched"},
)
```

---

### 2.5 Request Matchers

#### JSON body matcher

```python
from responses import matchers

@responses.activate
def test_json_body_matcher():
    responses.post(
        "https://api.example.com/users",
        match=[matchers.json_params_matcher({"username": "alice", "role": "admin"})],
        json={"id": 1, "username": "alice"},
        status=201,
    )
    response = requests.post(
        "https://api.example.com/users",
        json={"username": "alice", "role": "admin"},
    )
    assert response.status_code == 201
```

#### Form data matcher

```python
@responses.activate
def test_form_data_matcher():
    responses.post(
        "https://auth.example.com/token",
        match=[matchers.urlencoded_params_matcher({"grant_type": "password", "username": "alice"})],
        json={"access_token": "token123"},
    )
```

#### Header matcher

```python
@responses.activate
def test_header_matcher():
    responses.get(
        "https://api.example.com/protected",
        match=[matchers.request_kwargs_matcher({"headers": {"Authorization": "Bearer token"}})],
        json={"data": "secret"},
    )
```

#### Query parameter matcher

```python
@responses.activate
def test_query_params():
    responses.get(
        "https://api.example.com/items",
        match=[matchers.query_param_matcher({"skip": "0", "limit": "10"})],
        json=[],
    )
    response = requests.get("https://api.example.com/items?skip=0&limit=10")
    assert response.status_code == 200
```

---

### 2.6 Callbacks as Responses

```python
import requests as req_lib  # avoid name collision

def request_callback(request):
    user_id = request.path_url.split("/")[-1]
    payload = {"id": user_id, "name": f"User {user_id}"}
    return (200, {}, json.dumps(payload))

@responses.activate
def test_callback():
    responses.add_callback(
        responses.GET,
        re.compile(r"https://api\.example\.com/users/\d+"),
        callback=request_callback,
        content_type="application/json",
    )
    response = req_lib.get("https://api.example.com/users/42")
    assert response.json()["id"] == "42"
```

---

### 2.7 Passthrough Requests

Allow some requests to reach the real network:

```python
@responses.activate
def test_with_passthrough():
    responses.add_passthrough("https://real-api.example.com")
    responses.add(
        responses.GET,
        "https://mocked-api.example.com/data",
        json={"mocked": True},
    )
    # This hits the real network
    # real_response = requests.get("https://real-api.example.com/health")
    # This is mocked
    mocked = requests.get("https://mocked-api.example.com/data")
    assert mocked.json()["mocked"] is True
```

---

### 2.8 Assertions

#### `assert_call_count`

```python
@responses.activate
def test_call_count():
    responses.get("https://api.example.com/items", json=[])

    requests.get("https://api.example.com/items")
    requests.get("https://api.example.com/items")

    assert len(responses.calls) == 2
    assert responses.calls[0].request.method == "GET"
```

#### `assert_all_requests_are_fired`

```python
@responses.activate(assert_all_requests_are_fired=True)
def test_all_mocked_used():
    responses.get("https://api.example.com/a", json={"a": 1})
    responses.get("https://api.example.com/b", json={"b": 2})

    requests.get("https://api.example.com/a")
    # Will FAIL if /b is never called
```

#### Inspecting `responses.calls`

```python
@responses.activate
def test_inspect_calls():
    responses.post("https://api.example.com/events", json={"queued": True}, status=202)

    requests.post("https://api.example.com/events", json={"type": "user.created", "id": 1})
    requests.post("https://api.example.com/events", json={"type": "order.placed", "id": 2})

    assert len(responses.calls) == 2

    first_call = responses.calls[0]
    assert json.loads(first_call.request.body)["type"] == "user.created"

    second_call = responses.calls[1]
    assert json.loads(second_call.request.body)["id"] == 2
```

---

### 2.9 Raising Exceptions

```python
import requests as req
from requests.exceptions import ConnectionError, Timeout

@responses.activate
def test_connection_error():
    responses.get(
        "https://api.example.com/unstable",
        body=ConnectionError("Connection refused"),
    )
    with pytest.raises(ConnectionError):
        req.get("https://api.example.com/unstable")

@responses.activate
def test_timeout():
    responses.get(
        "https://api.example.com/slow",
        body=Timeout("Read timeout"),
    )
    with pytest.raises(Timeout):
        req.get("https://api.example.com/slow")
```

---

### 2.10 Integration with FastAPI + requests Service

```python
# app/services/payment.py
import requests

PAYMENT_API_URL = "https://payment.example.com/api"

def charge_card(amount: float, token: str) -> dict:
    response = requests.post(
        f"{PAYMENT_API_URL}/charge",
        json={"amount": amount, "token": token},
        headers={"Authorization": f"Bearer {PAYMENT_API_URL}"},
        timeout=10,
    )
    response.raise_for_status()
    return response.json()
```

```python
# tests/test_payment.py
import json
import pytest
import responses as resp_lib
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

@resp_lib.activate
def test_successful_payment():
    resp_lib.post(
        "https://payment.example.com/api/charge",
        json={"transaction_id": "txn_123", "status": "approved"},
        status=200,
    )
    response = client.post("/checkout/", json={"amount": 99.99, "card_token": "tok_abc"})
    assert response.status_code == 200
    assert response.json()["transaction_id"] == "txn_123"

@resp_lib.activate
def test_payment_declined():
    resp_lib.post(
        "https://payment.example.com/api/charge",
        json={"error": "card_declined"},
        status=402,
    )
    response = client.post("/checkout/", json={"amount": 99.99, "card_token": "tok_bad"})
    assert response.status_code == 402
    assert "declined" in response.json()["detail"].lower()
```

---

---

## Part 3: pytest-httpserver (Real TCP Server)

`pytest-httpserver` starts a real HTTP server (using Python's `wsgiref` or `werkzeug`) on localhost. Use it when:

- You need a real socket (testing with `socket.setdefaulttimeout`, cert validation, etc.)
- You test code that creates its own `requests.Session` with custom adapters
- You verify TLS/certificate behaviour
- You test chunked transfer encoding edge cases

```bash
pip install pytest-httpserver
```

---

### 3.1 The `httpserver` Fixture

`httpserver` is automatically available once `pytest-httpserver` is installed:

```python
# tests/test_httpserver_basic.py
import requests

def test_basic_handler(httpserver):
    httpserver.expect_request("/hello").respond_with_data(
        '{"message": "world"}',
        content_type="application/json",
    )
    url = httpserver.url_for("/hello")
    response = requests.get(url)
    assert response.status_code == 200
    assert response.json() == {"message": "world"}
```

---

### 3.2 Request Handlers — Ordered vs Unordered

#### Ordered (default) — `expect_ordered_request`

Ordered handlers must be triggered in the exact sequence they were registered:

```python
def test_ordered_requests(httpserver):
    httpserver.expect_ordered_request("/step/1", method="POST").respond_with_data(
        '{"step": 1}', content_type="application/json"
    )
    httpserver.expect_ordered_request("/step/2", method="POST").respond_with_data(
        '{"step": 2}', content_type="application/json"
    )

    r1 = requests.post(httpserver.url_for("/step/1"))
    r2 = requests.post(httpserver.url_for("/step/2"))

    assert r1.json()["step"] == 1
    assert r2.json()["step"] == 2
    httpserver.check_assertions()  # verify both were called
```

#### Unordered — `expect_request`

Unordered handlers respond to any matching request regardless of order:

```python
def test_unordered_requests(httpserver):
    httpserver.expect_request("/a").respond_with_data("A")
    httpserver.expect_request("/b").respond_with_data("B")

    # Can call in any order
    assert requests.get(httpserver.url_for("/b")).text == "B"
    assert requests.get(httpserver.url_for("/a")).text == "A"
```

---

### 3.3 Handler Matchers

#### URI matcher

```python
def test_uri_matcher(httpserver):
    httpserver.expect_request("/users/42").respond_with_json({"id": 42})
    response = requests.get(httpserver.url_for("/users/42"))
    assert response.json()["id"] == 42
```

#### URI pattern (regex)

```python
def test_uri_pattern(httpserver):
    import re
    httpserver.expect_request(
        re.compile(r"^/users/\d+$")
    ).respond_with_json({"matched": True})
    assert requests.get(httpserver.url_for("/users/123")).json()["matched"] is True
```

#### Method matcher

```python
def test_method_matcher(httpserver):
    httpserver.expect_request("/resource", method="DELETE").respond_with_data(
        "", status=204
    )
    response = requests.delete(httpserver.url_for("/resource"))
    assert response.status_code == 204

    # GET is NOT mocked — returns 500
    get_response = requests.get(httpserver.url_for("/resource"))
    assert get_response.status_code == 500
```

#### Header matcher

```python
def test_header_matcher(httpserver):
    httpserver.expect_request(
        "/protected",
        headers={"Authorization": "Bearer my-token"},
    ).respond_with_json({"access": "granted"})

    # With correct header
    response = requests.get(
        httpserver.url_for("/protected"),
        headers={"Authorization": "Bearer my-token"},
    )
    assert response.json()["access"] == "granted"

    # With wrong header — 500
    bad = requests.get(httpserver.url_for("/protected"))
    assert bad.status_code == 500
```

#### JSON body matcher

```python
def test_json_body_matcher(httpserver):
    httpserver.expect_request(
        "/items",
        method="POST",
        json={"name": "Widget", "price": 9.99},
    ).respond_with_json({"id": 1, "name": "Widget"}, status=201)

    response = requests.post(
        httpserver.url_for("/items"),
        json={"name": "Widget", "price": 9.99},
    )
    assert response.status_code == 201
    assert response.json()["id"] == 1
```

#### Raw data matcher

```python
def test_data_matcher(httpserver):
    httpserver.expect_request(
        "/upload",
        method="POST",
        data=b"raw binary data",
    ).respond_with_data("Received", status=200)

    response = requests.post(
        httpserver.url_for("/upload"),
        data=b"raw binary data",
    )
    assert response.text == "Received"
```

#### Query string matcher

```python
def test_query_string(httpserver):
    httpserver.expect_request(
        "/search",
        query_string="q=fastapi&limit=10",
    ).respond_with_json({"results": []})

    response = requests.get(httpserver.url_for("/search") + "?q=fastapi&limit=10")
    assert response.status_code == 200
```

---

### 3.4 Response Definition

```python
def test_response_types(httpserver):
    # Plain text
    httpserver.expect_request("/text").respond_with_data(
        "Hello", content_type="text/plain"
    )

    # JSON
    httpserver.expect_request("/json").respond_with_json(
        {"key": "value"},
        status=200,
        headers={"X-Custom": "header"},
    )

    # HTML
    httpserver.expect_request("/html").respond_with_data(
        "<html>Hi</html>",
        content_type="text/html",
        status=200,
    )

    # Custom werkzeug Response
    from werkzeug.wrappers import Response
    httpserver.expect_request("/custom").respond_with_response(
        Response(response='{"custom": true}', status=201, mimetype="application/json")
    )

    assert requests.get(httpserver.url_for("/text")).text == "Hello"
    assert requests.get(httpserver.url_for("/json")).json()["key"] == "value"
```

---

### 3.5 Permanent vs One-Shot Handlers

```python
def test_permanent_handler(httpserver):
    """Permanent handler responds to all matching requests."""
    httpserver.expect_request("/ping").respond_with_data("pong")

    for _ in range(5):
        response = requests.get(httpserver.url_for("/ping"))
        assert response.text == "pong"

def test_oneshot_handler(httpserver):
    """One-shot handler responds only once, then returns 500."""
    httpserver.expect_oneshot_request("/consume").respond_with_json({"used": True})

    first = requests.get(httpserver.url_for("/consume"))
    assert first.status_code == 200

    second = requests.get(httpserver.url_for("/consume"))
    assert second.status_code == 500  # handler exhausted
```

---

### 3.6 HTTPS Server Setup

```python
# conftest.py
import ssl
import pytest
from pytest_httpserver import HTTPServer

@pytest.fixture(scope="session")
def https_server(tmp_path_factory):
    """Spin up a real HTTPS server with a self-signed cert."""
    import trustme

    ca = trustme.CA()
    server_cert = ca.issue_cert("localhost")

    # Write certs to temp files
    certs_dir = tmp_path_factory.mktemp("certs")
    server_cert_file = str(certs_dir / "server.pem")
    server_key_file = str(certs_dir / "server.key")
    ca_cert_file = str(certs_dir / "ca.pem")

    server_cert.private_key_pem.write_to_path(server_key_file)
    server_cert.cert_chain_pems[0].write_to_path(server_cert_file)
    ca.cert_pem.write_to_path(ca_cert_file)

    ssl_context = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
    ssl_context.load_cert_chain(server_cert_file, server_key_file)

    server = HTTPServer(ssl_context=ssl_context)
    server.start()
    yield server, ca_cert_file
    server.clear()
    server.stop()


def test_https_endpoint(https_server):
    server, ca_cert_file = https_server
    server.expect_request("/secure").respond_with_json({"tls": True})

    response = requests.get(
        server.url_for("/secure"),
        verify=ca_cert_file,
    )
    assert response.json()["tls"] is True
```

---

### 3.7 `wait_for_handler_calls()` for Async Clients

When testing code that makes HTTP calls asynchronously (e.g., a background task or separate thread), use `wait_for_handler_calls`:

```python
import threading

def test_async_client_calls(httpserver):
    httpserver.expect_request("/webhook").respond_with_data("OK")

    # Simulate async call from background thread
    def background_call():
        import time
        time.sleep(0.05)
        requests.post(httpserver.url_for("/webhook"), json={"event": "test"})

    thread = threading.Thread(target=background_call)
    thread.start()

    # Wait up to 2 seconds for the handler to be called
    httpserver.wait_for_handler_calls(raise_assertions=True, timeout=2)
    thread.join()
```

---

### 3.8 `check_assertions()` — Verify All Ordered Handlers Were Called

```python
def test_webhook_sequence(httpserver):
    httpserver.expect_ordered_request("/webhook/start", method="POST").respond_with_data("OK")
    httpserver.expect_ordered_request("/webhook/complete", method="POST").respond_with_data("OK")

    # Trigger your FastAPI endpoint that should call both webhooks
    response = client.post("/process-job/", json={"job_id": "123"})
    assert response.status_code == 200

    # Verify both webhook calls happened in order
    httpserver.check_assertions()
```

---

### 3.9 Custom Error Responses

```python
def test_server_error_handling(httpserver):
    httpserver.expect_request("/flaky").respond_with_data(
        "Internal Server Error",
        content_type="text/plain",
        status=500,
    )

    # Test that your FastAPI app handles the upstream 500 gracefully
    response = client.get("/proxy/flaky")
    assert response.status_code == 502  # your app returns 502 Bad Gateway
```

---

### 3.10 Integration with FastAPI (Outbound HTTP to External Service)

```python
# app/services/notification.py
import httpx

NOTIFICATION_URL = "https://notifications.example.com"

async def send_webhook(url: str, payload: dict) -> bool:
    async with httpx.AsyncClient() as client:
        response = await client.post(url, json=payload, timeout=5.0)
        return response.is_success
```

```python
# app/main.py
@app.post("/send-notification/")
async def trigger_notification(webhook_url: str, data: dict):
    success = await send_webhook(webhook_url, data)
    if not success:
        raise HTTPException(status_code=502, detail="Notification failed")
    return {"sent": True}
```

```python
# tests/test_notification.py
import pytest
import httpx
import pytest_asyncio
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_send_notification(httpserver):
    httpserver.expect_request("/webhook", method="POST").respond_with_json(
        {"received": True}, status=200
    )
    webhook_url = httpserver.url_for("/webhook")
    response = client.post(
        "/send-notification/",
        params={"webhook_url": webhook_url},
        json={"event": "order.created", "order_id": 123},
    )
    assert response.status_code == 200
    assert response.json()["sent"] is True
    httpserver.check_assertions()

def test_send_notification_failure(httpserver):
    httpserver.expect_request("/webhook", method="POST").respond_with_data("", status=503)
    webhook_url = httpserver.url_for("/webhook")
    response = client.post(
        "/send-notification/",
        params={"webhook_url": webhook_url},
        json={"event": "test"},
    )
    assert response.status_code == 502
```

---

## Part 4: Choosing the Right Library

| Scenario | Best Library |
|---------|-------------|
| FastAPI service calls external API via `httpx` | **respx** |
| FastAPI service calls external API via `requests` | **responses** |
| Testing TLS/certificate validation | **pytest-httpserver** |
| Testing chunked transfer encoding | **pytest-httpserver** |
| Testing webhook delivery (your app POSTs out) | **pytest-httpserver** |
| Async streaming mock responses | **respx** |
| Complex request matching with callbacks | **respx** or **responses** |
| Verifying request order | **pytest-httpserver** (ordered) |
| Rotating responses on sequential calls | **respx** (side_effect list) |
| Background task makes HTTP calls | **pytest-httpserver** (with `wait_for_handler_calls`) |

---

## Part 5: Combining Libraries

In complex scenarios, you may combine multiple libraries:

```python
# tests/test_combined.py
import respx
import httpx
import responses as resp
import requests
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

@respx.mock
@resp.activate
def test_dual_mocking():
    """
    Your app uses httpx to call Service A and requests to call Service B.
    """
    # Mock Service A (httpx)
    respx.get("https://service-a.example.com/data").respond(200, json={"a": 1})

    # Mock Service B (requests)
    resp.get("https://service-b.example.com/data", json={"b": 2})

    response = client.get("/aggregate")
    assert response.status_code == 200
    data = response.json()
    assert data["service_a"]["a"] == 1
    assert data["service_b"]["b"] == 2
```
