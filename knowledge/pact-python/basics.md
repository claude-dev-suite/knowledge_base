# Pact Python — Consumer-Driven Contract Testing

Comprehensive guide to contract testing with pact-python: consumer tests, provider verification, matchers, Pact Broker integration, and CI/CD workflows.

**Official Documentation:**
- Pact Python: https://docs.pact.io/implementation_guides/python
- Pact reference: https://docs.pact.io/
- Pact Broker: https://docs.pact.io/pact_broker
- GitHub: https://github.com/pact-foundation/pact-python

---

## Table of Contents

1. [Consumer-Driven Contract Testing](#consumer-driven-contract-testing)
2. [Installation](#installation)
3. [Consumer Side Tests](#consumer-side-tests)
4. [All Matchers](#all-matchers)
5. [Provider Verification](#provider-verification)
6. [Pact Broker Integration](#pact-broker-integration)
7. [GitHub Actions CI Workflow](#github-actions-ci-workflow)
8. [V3 Pact Format and Async Message Pacts](#v3-pact-format-and-async-message-pacts)

---

## Consumer-Driven Contract Testing

### Why Contract Testing?

In a microservices architecture, integration testing is expensive: you need the entire system running to verify that Service A can talk to Service B. Contract testing solves this by:

1. **Consumer writes a pact** — a formal description of what it expects from the provider (request shape, response shape, status codes)
2. **Pact is stored** — in a file or Pact Broker
3. **Provider verifies the pact** — runs against the actual provider code without the consumer

This eliminates the need for shared integration environments just to verify API compatibility.

```
┌──────────────────┐     pact file      ┌──────────────────┐
│   Consumer Test  │──────────────────▶ │ Provider Verifier│
│  (no real server)│                    │ (no consumer app)│
│  generates pact  │                    │ verifies pact    │
└──────────────────┘                    └──────────────────┘
         │                                       │
         │          ┌─────────────┐              │
         └─────────▶│ Pact Broker │◀─────────────┘
                    │  (central)  │
                    └─────────────┘
```

### Key Concepts

| Term | Meaning |
|------|---------|
| **Consumer** | The service that makes HTTP requests (the client) |
| **Provider** | The service that responds to requests (the API) |
| **Pact** | A JSON file describing the interaction contract |
| **Interaction** | One request/response pair in the pact |
| **Provider State** | Setup instructions for the provider before an interaction runs |
| **Pact Broker** | Central server for storing and distributing pacts |

---

## Installation

```bash
# Core pact-python library (includes mock server binary)
pip install pact-python

# For provider verification from Pact Broker
pip install pact-python

# With pytest
pip install pact-python pytest
```

**Version note:** pact-python v2.x wraps the Ruby pact-mock-service binary. v3.x (pact-python ≥ 2.3) supports V3 pact specification with native Rust core.

```bash
# Check version
python -c "import pact; print(pact.__version__)"

# Install pact broker CLI (for publishing and can-i-deploy)
pip install pact-python[broker]
# Or use the standalone Pact CLI binary:
# https://github.com/pact-foundation/pact-ruby-standalone/releases
```

---

## Consumer Side Tests

The consumer test:
1. Starts a local mock HTTP server (the Pact mock provider)
2. Defines expected interactions (given state + request → response)
3. Runs the actual consumer code against the mock server
4. Asserts that the consumer called the mock correctly
5. Writes the pact JSON file if assertions pass

### Basic Consumer Test Structure

```python
# tests/consumer/test_user_service_contract.py
import json
import pytest
from pact import Consumer, Provider


PACT_DIR = "tests/consumer/pacts"
MOCK_HOST = "localhost"
MOCK_PORT = 1234


@pytest.fixture(scope="module")
def pact():
    """
    Create the Pact object and start the mock provider server.
    scope="module" so the server starts once for all tests in the file.
    """
    pact = Consumer("UserService").has_pact_with(
        Provider("AccountService"),
        host_name=MOCK_HOST,
        port=MOCK_PORT,
        pact_dir=PACT_DIR,
        # V2 is default; use version="3.0.0" for V3 spec
        version="2.0.0",
        # Log level for pact mock server
        log_dir="logs/",
        log_level="INFO",
    )
    pact.start_service()
    yield pact
    pact.stop_service()


def test_get_user_by_id(pact):
    """Consumer expects to get a user object by ID."""
    from myapp.clients.account_client import AccountClient
    from pact import Like

    # Define the interaction
    (
        pact
        .given("User with ID 123 exists")
        .upon_receiving("A request for user 123")
        .with_request(
            method="GET",
            path="/api/users/123",
            headers={"Accept": "application/json"},
        )
        .will_respond_with(
            status=200,
            headers={"Content-Type": "application/json"},
            body={
                "id": Like(123),                    # Type match: any integer
                "email": Like("user@example.com"),  # Type match: any string
                "name": Like("Alice Smith"),
                "is_active": Like(True),
            },
        )
    )

    # Run the consumer code against the mock server
    with pact:
        client = AccountClient(base_url=f"http://{MOCK_HOST}:{MOCK_PORT}")
        user = client.get_user(user_id=123)

    # Verify the consumer correctly used the response
    assert user.id == 123
    assert user.email is not None
    assert user.name is not None
```

### Using Context Manager vs start/stop

```python
# Option A: Context manager (recommended for single interaction)
with pact:
    result = client.call_api()

# Option B: Manual start/stop (for module-scoped pact fixture)
pact.start_service()
try:
    # run tests
    with pact:
        result = client.call_api()
finally:
    pact.stop_service()

# Option C: Using pact as fixture with autouse
@pytest.fixture(autouse=True, scope="module")
def pact_server():
    p = Consumer("A").has_pact_with(Provider("B"), port=1234, pact_dir="pacts/")
    p.start_service()
    yield p
    p.stop_service()
```

### Complete Consumer Test Example — CRUD API

```python
# tests/consumer/test_product_api_contract.py
import pytest
from pact import Consumer, Provider, Like, EachLike, Term

PACT_DIR = "tests/consumer/pacts"


@pytest.fixture(scope="module")
def pact():
    p = Consumer("StoreFront").has_pact_with(
        Provider("ProductAPI"),
        host_name="localhost",
        port=1235,
        pact_dir=PACT_DIR,
    )
    p.start_service()
    yield p
    p.stop_service()


@pytest.fixture
def product_client(pact):
    from myapp.clients.product_client import ProductClient
    return ProductClient(base_url="http://localhost:1235")


def test_list_products(pact, product_client):
    """Consumer expects a list of products."""
    (
        pact
        .given("Products exist in the catalog")
        .upon_receiving("A request for all products")
        .with_request(
            method="GET",
            path="/api/products",
            query={"page": "1", "per_page": "10"},
        )
        .will_respond_with(
            status=200,
            body={
                "items": EachLike(
                    {
                        "id": Like(1),
                        "name": Like("Widget"),
                        "price": Like(9.99),
                        "sku": Term(r"^SKU-\d{6}$", "SKU-000001"),
                    },
                    minimum=1,
                ),
                "total": Like(42),
                "page": Like(1),
            },
        )
    )

    with pact:
        products = product_client.list_products(page=1, per_page=10)

    assert len(products.items) >= 1
    assert products.total >= 0


def test_get_product_found(pact, product_client):
    """Consumer expects to retrieve a product by SKU."""
    (
        pact
        .given("Product SKU-000001 exists")
        .upon_receiving("A GET request for product SKU-000001")
        .with_request(
            method="GET",
            path="/api/products/SKU-000001",
        )
        .will_respond_with(
            status=200,
            body={
                "id": Like(1),
                "name": Like("Widget"),
                "sku": "SKU-000001",      # Exact match for the requested SKU
                "price": Like(9.99),
                "stock": Like(100),
                "category": Like("tools"),
            },
        )
    )

    with pact:
        product = product_client.get_product("SKU-000001")

    assert product.sku == "SKU-000001"


def test_get_product_not_found(pact, product_client):
    """Consumer handles 404 gracefully."""
    (
        pact
        .given("Product SKU-999999 does not exist")
        .upon_receiving("A GET request for non-existent product")
        .with_request(
            method="GET",
            path="/api/products/SKU-999999",
        )
        .will_respond_with(
            status=404,
            body={
                "error": Like("Product not found"),
                "code": Like("NOT_FOUND"),
            },
        )
    )

    with pact:
        from myapp.clients.exceptions import ProductNotFoundError
        with pytest.raises(ProductNotFoundError):
            product_client.get_product("SKU-999999")


def test_create_product(pact, product_client):
    """Consumer expects to create a new product."""
    (
        pact
        .given("The catalog accepts new products")
        .upon_receiving("A POST request to create a product")
        .with_request(
            method="POST",
            path="/api/products",
            headers={"Content-Type": "application/json"},
            body={
                "name": "New Widget",
                "price": 19.99,
                "category": "tools",
            },
        )
        .will_respond_with(
            status=201,
            headers={"Content-Type": "application/json"},
            body={
                "id": Like(99),
                "name": "New Widget",        # Exact match — sent by consumer
                "price": 19.99,
                "sku": Term(r"^SKU-\d{6}$", "SKU-000099"),
                "created_at": Term(
                    r"^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}",
                    "2026-01-15T10:30:00",
                ),
            },
        )
    )

    with pact:
        product = product_client.create_product(
            name="New Widget",
            price=19.99,
            category="tools",
        )

    assert product.id is not None
    assert product.name == "New Widget"
    assert product.sku.startswith("SKU-")


def test_delete_product(pact, product_client):
    """Consumer expects 204 No Content on successful delete."""
    (
        pact
        .given("Product SKU-000001 exists")
        .upon_receiving("A DELETE request for product SKU-000001")
        .with_request(
            method="DELETE",
            path="/api/products/SKU-000001",
        )
        .will_respond_with(status=204, body=None)
    )

    with pact:
        result = product_client.delete_product("SKU-000001")

    assert result is True
```

### Generated Pact File

After running the consumer tests, pact files are written to `PACT_DIR`:

```json
// tests/consumer/pacts/StoreFront-ProductAPI.json
{
  "consumer": { "name": "StoreFront" },
  "provider": { "name": "ProductAPI" },
  "interactions": [
    {
      "description": "A GET request for product SKU-000001",
      "providerState": "Product SKU-000001 exists",
      "request": {
        "method": "GET",
        "path": "/api/products/SKU-000001"
      },
      "response": {
        "status": 200,
        "headers": { "Content-Type": "application/json" },
        "body": {
          "id": 1,
          "name": "Widget",
          "sku": "SKU-000001",
          "price": 9.99,
          "stock": 100,
          "category": "tools"
        },
        "matchingRules": {
          "$.body.id": { "match": "type" },
          "$.body.name": { "match": "type" },
          "$.body.price": { "match": "type" },
          "$.body.stock": { "match": "type" },
          "$.body.category": { "match": "type" }
        }
      }
    }
  ],
  "metadata": {
    "pactSpecification": { "version": "2.0.0" }
  }
}
```

---

## All Matchers

Matchers allow flexible verification: instead of requiring an exact value, you specify the type, pattern, or structure.

### Import

```python
from pact import Like, EachLike, Term
from pact.matchers import (
    Format,
    Integer,
    Decimal,
    Boolean,
    String,
    Includes,
    NotNull,
    Null,
)
```

### Like(value) — Type Matching

Matches any value of the same type. The `value` is used as the example in the pact file.

```python
from pact import Like

# Matches any integer
Like(42)                    # matches: 1, 0, -5, 99999

# Matches any string
Like("hello")               # matches: "foo", "bar", ""

# Matches any float/decimal
Like(9.99)                  # matches: 0.0, 1.5, 999.99

# Matches any boolean
Like(True)                  # matches: True, False

# Matches any object with the same keys (values type-matched recursively)
Like({
    "id": 1,
    "email": "user@example.com",
    "active": True,
})

# Nested Like
Like({
    "user": Like({
        "id": Like(1),
        "profile": Like({
            "bio": Like("A developer"),
        }),
    }),
})
```

### EachLike(item, minimum=1) — Array Element Matching

Matches an array where each element matches the given template. The `minimum` parameter sets the required minimum array length.

```python
from pact import EachLike

# Array of at least 1 integer
EachLike(1)

# Array of at least 2 strings
EachLike("item", minimum=2)

# Array of objects — each element must match the template
EachLike({
    "id": Like(1),
    "name": Like("Product A"),
    "price": Like(9.99),
}, minimum=1)

# Nested: array of arrays
EachLike(EachLike(1), minimum=1)

# In a response body
will_respond_with(
    status=200,
    body={
        "products": EachLike({
            "id": Like(1),
            "name": Like("Widget"),
        }, minimum=2),  # Response must have at least 2 products
    },
)
```

### Term(regex, value) — Regex Matching

Matches any string matching the regex pattern. `value` must itself match the regex (used as the example).

```python
from pact import Term

# UUID
Term(
    r"^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$",
    "550e8400-e29b-41d4-a716-446655440000",
)

# ISO 8601 date
Term(r"^\d{4}-\d{2}-\d{2}$", "2026-01-15")

# ISO 8601 datetime
Term(
    r"^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}(\.\d+)?Z?$",
    "2026-01-15T10:30:00Z",
)

# SKU format
Term(r"^SKU-[A-Z0-9]{6}$", "SKU-ABC123")

# Email (simplified)
Term(r"^[^@]+@[^@]+\.[^@]+$", "user@example.com")

# Phone number
Term(r"^\+?[0-9]{10,15}$", "+15551234567")

# Enum-like: exactly one of these values
Term(r"^(pending|confirmed|shipped|delivered)$", "pending")

# HTTP URL
Term(r"^https?://", "https://example.com/image.png")
```

### Format — Predefined Formats (V3)

Available in pact-python with V3 specification:

```python
from pact.matchers import Format

Format.date         # "2026-01-15"      regex: ^\d{4}-\d{2}-\d{2}$
Format.time         # "10:30:00"        regex: ^\d{2}:\d{2}:\d{2}$
Format.datetime     # "2026-01-15T10:30:00Z"
Format.uuid         # "550e8400-..."    UUID v4
Format.ipv4         # "192.168.1.1"
Format.ipv6         # "2001:0db8::1"

# Usage
will_respond_with(
    status=200,
    body={
        "id": Format.uuid,
        "created_at": Format.datetime,
        "published_date": Format.date,
    },
)
```

### Integer(value), Decimal(value), Boolean(value), String(value)

Explicit type matchers for primitive types:

```python
from pact.matchers import Integer, Decimal, Boolean, String

Integer(42)         # Matches any integer; 42 is the example
Decimal(3.14)       # Matches any float/decimal
Boolean(True)       # Matches any boolean
String("hello")     # Matches any string (same as Like("hello"))

# These differ from Like() in that they enforce the specific Python type
# Integer will reject 3.14 even though Like(42) would accept it
```

### Includes(string) — Substring Match

```python
from pact.matchers import Includes

# Matches any string that contains "error"
Includes("error")

# Use case: error message responses
will_respond_with(
    status=400,
    body={
        "message": Includes("validation failed"),
    },
)
```

### NotNull(), Null()

```python
from pact.matchers import NotNull, Null

# Field must be present and not null
NotNull()

# Field must be null
Null()

# Use case: optional fields
will_respond_with(
    status=200,
    body={
        "id": Like(1),
        "deleted_at": Null(),       # Not deleted → null
        "verified_at": NotNull(),   # Verified → has a value
    },
)
```

### Nested Matchers in Objects and Arrays

```python
from pact import Like, EachLike, Term
from pact.matchers import Integer, Decimal, NotNull

# Complex nested structure
will_respond_with(
    status=200,
    body={
        "order": {
            "id": Integer(1001),
            "status": Term(r"^(pending|confirmed|shipped)$", "confirmed"),
            "total": Decimal(99.99),
            "customer": Like({
                "id": Integer(1),
                "name": Like("Alice Smith"),
                "email": Term(r"^[^@]+@[^@]+\.[^@]+$", "alice@example.com"),
            }),
            "items": EachLike({
                "product_id": Integer(1),
                "name": Like("Widget"),
                "quantity": Integer(2),
                "unit_price": Decimal(9.99),
                "subtotal": Decimal(19.98),
            }, minimum=1),
            "shipping_address": Like({
                "street": Like("123 Main St"),
                "city": Like("Springfield"),
                "country": Term(r"^[A-Z]{2}$", "US"),
                "postal_code": Term(r"^\d{5}(-\d{4})?$", "12345"),
            }),
            "created_at": Term(
                r"^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}",
                "2026-01-15T10:30:00Z",
            ),
            "notes": NotNull(),
        },
    },
)
```

### Request Body Matchers

Matchers work in request bodies too (for POST/PUT/PATCH):

```python
with_request(
    method="POST",
    path="/api/orders",
    headers={"Content-Type": "application/json"},
    body={
        "customer_id": Integer(1),
        "items": EachLike({
            "product_id": Integer(1),
            "quantity": Integer(1),
        }, minimum=1),
    },
)
```

---

## Provider Verification

Provider verification runs the pact file against the real provider code, verifying that the provider actually behaves as the consumer expects.

### Verifier Setup

```python
# tests/provider/test_product_api_provider.py
import pytest
from pact import Verifier


PACT_DIR = "tests/consumer/pacts"
PROVIDER_HOST = "localhost"
PROVIDER_PORT = 8000


@pytest.fixture(scope="module")
def provider_server():
    """Start the real FastAPI provider on a test port."""
    import uvicorn
    import threading
    from myapp.main import app

    server = uvicorn.Server(
        config=uvicorn.Config(app, host=PROVIDER_HOST, port=PROVIDER_PORT, log_level="warning")
    )
    thread = threading.Thread(target=server.run, daemon=True)
    thread.start()
    # Wait for server to be ready
    import time
    import httpx
    for _ in range(30):
        try:
            httpx.get(f"http://{PROVIDER_HOST}:{PROVIDER_PORT}/health")
            break
        except Exception:
            time.sleep(0.5)
    yield
    server.should_exit = True


def test_product_api_pact_verification(provider_server):
    """Verify all consumer pacts against the running provider."""
    verifier = Verifier(
        provider="ProductAPI",
        provider_base_url=f"http://{PROVIDER_HOST}:{PROVIDER_PORT}",
    )

    output, _ = verifier.verify_pacts(
        sources=[f"{PACT_DIR}/StoreFront-ProductAPI.json"],
        # URL for provider state setup endpoint
        provider_states_setup_url=(
            f"http://{PROVIDER_HOST}:{PROVIDER_PORT}/_pact/provider-states"
        ),
    )

    assert output == 0, "Pact verification failed"
```

### Provider States — Setting Up Test Data

The provider must expose an endpoint that the verifier calls before each interaction to set up the required state.

```python
# myapp/testing/provider_states.py
"""
Provider state endpoint — only active during pact verification.
Mounted at /_pact/provider-states in test mode.
"""
from fastapi import APIRouter, Request
from sqlalchemy.orm import Session

from myapp.database import get_db
from myapp.models import Product

router = APIRouter(prefix="/_pact")


@router.post("/provider-states")
async def setup_provider_state(request: Request):
    """
    Called by pact verifier before each interaction.
    Body: {"state": "Product SKU-000001 exists", "action": "setup"}
    """
    body = await request.json()
    state = body.get("state", "")
    action = body.get("action", "setup")

    if action == "setup":
        await handle_setup(state)
    elif action == "teardown":
        await handle_teardown(state)

    return {"status": "ok"}


async def handle_setup(state: str):
    from myapp.database import AsyncSessionLocal
    async with AsyncSessionLocal() as session:
        if state == "Product SKU-000001 exists":
            product = Product(
                id=1,
                name="Widget",
                sku="SKU-000001",
                price=9.99,
                stock=100,
                category="tools",
            )
            session.add(product)
            await session.commit()

        elif state == "Product SKU-999999 does not exist":
            # Ensure it doesn't exist (it won't by default in test DB)
            existing = await session.get(Product, "SKU-999999")
            if existing:
                await session.delete(existing)
                await session.commit()

        elif state == "Products exist in the catalog":
            for i in range(5):
                product = Product(
                    name=f"Product {i}",
                    sku=f"SKU-{i:06d}",
                    price=float(i * 10 + 9.99),
                    stock=100,
                    category="tools",
                )
                session.add(product)
            await session.commit()

        elif state == "The catalog accepts new products":
            pass  # No setup needed — just needs an empty DB


async def handle_teardown(state: str):
    from myapp.database import AsyncSessionLocal
    from sqlalchemy import text
    async with AsyncSessionLocal() as session:
        # Clean up test data after each interaction
        await session.execute(text("DELETE FROM products"))
        await session.commit()
```

### Pytest Fixtures for Provider States

```python
# tests/provider/conftest.py
import pytest
from myapp.database import Base, engine


@pytest.fixture(autouse=True)
def clean_db():
    """Reset database before each pact interaction."""
    Base.metadata.drop_all(engine)
    Base.metadata.create_all(engine)
    yield
    Base.metadata.drop_all(engine)
```

### Verification Against Local Pact Files

```python
def test_verify_local_pact(provider_server):
    verifier = Verifier(
        provider="ProductAPI",
        provider_base_url="http://localhost:8000",
    )

    output, _ = verifier.verify_pacts(
        sources=["tests/consumer/pacts/StoreFront-ProductAPI.json"],
        provider_states_setup_url="http://localhost:8000/_pact/provider-states",
        # Enable detailed output
        verbose=True,
        # Log pact interactions
        log_dir="logs/",
        log_level="DEBUG",
    )
    assert output == 0
```

---

## Pact Broker Integration

The Pact Broker is a central server for storing, versioning, and distributing pacts between teams.

### Publishing Pacts

**Via pact-broker CLI:**
```bash
# Install pact standalone
# https://github.com/pact-foundation/pact-ruby-standalone/releases

pact-broker publish \
  tests/consumer/pacts/ \
  --consumer-app-version "$(git rev-parse HEAD)" \
  --branch "$(git rev-parse --abbrev-ref HEAD)" \
  --broker-base-url "https://your-pact-broker.example.com" \
  --broker-token "$PACT_BROKER_TOKEN"
```

**Via Python (in CI):**
```python
# publish_pacts.py
import subprocess
import os
import sys


def publish_pacts():
    result = subprocess.run(
        [
            "pact-broker", "publish",
            "tests/consumer/pacts/",
            "--consumer-app-version", os.getenv("GIT_COMMIT", "local"),
            "--branch", os.getenv("GIT_BRANCH", "main"),
            "--broker-base-url", os.environ["PACT_BROKER_BASE_URL"],
            "--broker-token", os.environ["PACT_BROKER_TOKEN"],
        ],
        check=True,
        capture_output=True,
        text=True,
    )
    print(result.stdout)


if __name__ == "__main__":
    publish_pacts()
```

**Via pact-python Broker client:**
```python
from pact.broker import Broker


def publish_consumer_pacts(
    broker_url: str,
    broker_token: str,
    consumer_version: str,
    pact_files: list[str],
    branch: str = "main",
):
    broker = Broker(
        broker_base_url=broker_url,
        broker_token=broker_token,
    )
    broker.publish_pacts(
        pact_files=pact_files,
        consumer_version=consumer_version,
        branch=branch,
        tags=[branch],
    )
```

### Provider Verification Against Pact Broker

```python
# tests/provider/test_verify_from_broker.py
import os
import pytest
from pact import Verifier


def test_verify_pacts_from_broker(provider_server):
    """
    Verify all pacts for this provider from the Pact Broker.
    This fetches pacts from all consumers that contract with ProductAPI.
    """
    broker_url = os.environ["PACT_BROKER_BASE_URL"]
    broker_token = os.environ["PACT_BROKER_TOKEN"]
    provider_version = os.environ.get("GIT_COMMIT", "local")
    provider_branch = os.environ.get("GIT_BRANCH", "main")

    verifier = Verifier(
        provider="ProductAPI",
        provider_base_url="http://localhost:8000",
    )

    output, _ = verifier.verify_with_broker(
        broker_url=broker_url,
        broker_token=broker_token,
        # Fetch pacts for consumers on main branch
        consumer_version_selectors=[
            {"branch": "main"},
            {"deployedOrReleased": True},  # Fetch pacts from deployed consumers
        ],
        provider_states_setup_url="http://localhost:8000/_pact/provider-states",
        # Publish verification results back to the broker
        publish_verification_results=True,
        provider_version=provider_version,
        provider_version_branch=provider_branch,
    )

    assert output == 0, "Pact verification failed — see logs for details"
```

### can-i-deploy — Deployment Gate

`can-i-deploy` checks the Pact Broker to determine whether all contracts are verified before deploying:

```bash
# Check if StoreFront v1.2.3 can be deployed to production
pact-broker can-i-deploy \
  --pacticipant "StoreFront" \
  --version "$(git rev-parse HEAD)" \
  --to-environment production \
  --broker-base-url "$PACT_BROKER_BASE_URL" \
  --broker-token "$PACT_BROKER_TOKEN"
# Exit code 0 = safe to deploy
# Exit code 1 = verification failed or missing
```

```bash
# Check multiple participants at once
pact-broker can-i-deploy \
  --pacticipant "StoreFront" --version "abc123" \
  --pacticipant "ProductAPI" --version "def456" \
  --to-environment production \
  --broker-base-url "$PACT_BROKER_BASE_URL" \
  --broker-token "$PACT_BROKER_TOKEN"
```

### Webhook Configuration

Configure the Pact Broker to trigger provider verification when a new consumer pact is published:

```bash
# Create webhook via CLI
pact-broker create-webhook \
  "https://ci.example.com/api/trigger-build" \
  --request POST \
  --header "Authorization: Bearer $CI_TOKEN" \
  --body '{"branch": "${pactbroker.providerVersionBranch}"}' \
  --description "Trigger ProductAPI build when pact changes" \
  --contract-content-changed \
  --provider ProductAPI \
  --broker-base-url "$PACT_BROKER_BASE_URL" \
  --broker-token "$PACT_BROKER_TOKEN"
```

---

## GitHub Actions CI Workflow

### Consumer Pipeline

```yaml
# .github/workflows/consumer-pact.yml
name: Consumer Pact Tests

on:
  push:
    branches: [main, "feature/**"]
  pull_request:

jobs:
  consumer-tests:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Cache pip
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('**/pyproject.toml') }}

      - name: Install dependencies
        run: pip install -e ".[test]"

      - name: Run consumer pact tests
        run: |
          pytest tests/consumer/ \
            -v \
            --tb=short \
            --junitxml=reports/consumer-pact-junit.xml

      - name: Publish pacts to Pact Broker
        if: github.event_name == 'push'
        env:
          PACT_BROKER_BASE_URL: ${{ secrets.PACT_BROKER_BASE_URL }}
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}
        run: |
          pact-broker publish \
            tests/consumer/pacts/ \
            --consumer-app-version "${{ github.sha }}" \
            --branch "${{ github.ref_name }}" \
            --broker-base-url "$PACT_BROKER_BASE_URL" \
            --broker-token "$PACT_BROKER_TOKEN"

      - name: Upload pact files
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: pact-files
          path: tests/consumer/pacts/
```

### Provider Pipeline

```yaml
# .github/workflows/provider-pact.yml
name: Provider Pact Verification

on:
  push:
    branches: [main]
  pull_request:
  # Trigger when consumer publishes a new pact (via Pact Broker webhook)
  repository_dispatch:
    types: [pact-changed]

jobs:
  provider-verification:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpassword
          POSTGRES_DB: testdb
        options: >-
          --health-cmd "pg_isready"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    env:
      DATABASE_URL: postgresql://testuser:testpassword@localhost:5432/testdb
      PACT_BROKER_BASE_URL: ${{ secrets.PACT_BROKER_BASE_URL }}
      PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}
      GIT_COMMIT: ${{ github.sha }}
      GIT_BRANCH: ${{ github.ref_name }}

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: pip install -e ".[test]"

      - name: Run database migrations
        run: alembic upgrade head

      - name: Run provider pact verification
        run: |
          pytest tests/provider/test_verify_from_broker.py \
            -v \
            --tb=short

      - name: Can I Deploy?
        if: github.ref == 'refs/heads/main'
        run: |
          pact-broker can-i-deploy \
            --pacticipant "ProductAPI" \
            --version "${{ github.sha }}" \
            --to-environment production \
            --broker-base-url "$PACT_BROKER_BASE_URL" \
            --broker-token "$PACT_BROKER_TOKEN"

      - name: Record deployment
        if: github.ref == 'refs/heads/main'
        run: |
          pact-broker record-deployment \
            --pacticipant "ProductAPI" \
            --version "${{ github.sha }}" \
            --environment production \
            --broker-base-url "$PACT_BROKER_BASE_URL" \
            --broker-token "$PACT_BROKER_TOKEN"
```

---

## V3 Pact Format and Async Message Pacts

### V3 Specification Features

Pact V3 (supported via pact-python ≥ 2.3) adds:
- Multiple provider states per interaction
- Body matching by content type
- Generators (dynamic value generation for requests)
- Message pacts (async/event-driven contracts)

```python
# Enable V3 in Consumer
pact = Consumer("OrderService").has_pact_with(
    Provider("NotificationService"),
    host_name="localhost",
    port=1236,
    pact_dir="tests/consumer/pacts",
    version="3.0.0",   # ← V3 spec
)
```

### Multiple Provider States (V3)

```python
# V3: multiple states per interaction
(
    pact
    .given([
        {"name": "User alice@example.com exists"},
        {"name": "User has notifications enabled"},
    ])
    .upon_receiving("A request to send notification to alice")
    .with_request(
        method="POST",
        path="/api/notifications",
        body={"user_email": "alice@example.com", "message": "Hello"},
    )
    .will_respond_with(
        status=202,
        body={"queued": True, "notification_id": Like("uuid-1234")},
    )
)
```

### Message Pacts — Kafka / Async Messaging

Message pacts test event-driven contracts where the consumer processes messages from a queue/topic, rather than HTTP requests.

**Consumer message test:**
```python
# tests/consumer/test_order_events_contract.py
import json
import pytest
from pact import MessageConsumer, Provider, Like, Term


@pytest.fixture(scope="module")
def message_pact():
    pact = MessageConsumer("InventoryService").has_pact_with(
        Provider("OrderService"),
        pact_dir="tests/consumer/pacts",
        version="3.0.0",
    )
    yield pact


def test_order_created_event_consumed(message_pact):
    """InventoryService consumes OrderCreated events from OrderService."""
    from myapp.handlers.order_events import handle_order_created

    # Define the expected message
    expected_message = message_pact.given(
        "An order has been created"
    ).expects_to_receive(
        "An OrderCreated event"
    ).with_content(
        {
            "event_type": "ORDER_CREATED",
            "order_id": Like("order-uuid-1234"),
            "customer_id": Like("customer-uuid-5678"),
            "items": [
                {
                    "product_id": Like("product-uuid-9012"),
                    "sku": Term(r"^SKU-\d{6}$", "SKU-000001"),
                    "quantity": Like(2),
                }
            ],
            "total": Like(19.98),
            "created_at": Term(
                r"^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}",
                "2026-01-15T10:30:00Z",
            ),
        }
    ).with_metadata(
        {
            "content-type": "application/json",
            "kafka-topic": "orders.created",
        }
    )

    # Verify the consumer correctly handles this message
    def message_handler(raw_message):
        """The actual consumer handler under test."""
        message = json.loads(raw_message["contents"])
        handle_order_created(message)

    message_pact.verify(message_handler, "An OrderCreated event")


def test_order_cancelled_event_consumed(message_pact):
    """InventoryService restores stock when an order is cancelled."""
    from myapp.handlers.order_events import handle_order_cancelled

    expected_message = message_pact.given(
        "An order exists and has been cancelled"
    ).expects_to_receive(
        "An OrderCancelled event"
    ).with_content(
        {
            "event_type": "ORDER_CANCELLED",
            "order_id": Like("order-uuid-1234"),
            "items": [
                {
                    "product_id": Like("product-uuid-9012"),
                    "quantity": Like(2),
                }
            ],
            "cancelled_at": Term(
                r"^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}",
                "2026-01-15T11:00:00Z",
            ),
        }
    )

    def message_handler(raw_message):
        message = json.loads(raw_message["contents"])
        handle_order_cancelled(message)

    message_pact.verify(message_handler, "An OrderCancelled event")
```

**Provider message test — producing the events:**
```python
# tests/provider/test_order_events_provider.py
import json
import pytest
from pact import MessageProvider


@pytest.fixture(scope="module")
def message_provider():
    """Configure the provider to produce messages for verification."""

    # Map provider states to message producers
    def produce_order_created():
        """Produce an OrderCreated event for pact verification."""
        from myapp.services.order_service import OrderService
        # Either call real service or construct directly
        return {
            "event_type": "ORDER_CREATED",
            "order_id": "order-uuid-1234",
            "customer_id": "customer-uuid-5678",
            "items": [
                {
                    "product_id": "product-uuid-9012",
                    "sku": "SKU-000001",
                    "quantity": 2,
                }
            ],
            "total": 19.98,
            "created_at": "2026-01-15T10:30:00Z",
        }

    def produce_order_cancelled():
        return {
            "event_type": "ORDER_CANCELLED",
            "order_id": "order-uuid-1234",
            "items": [
                {
                    "product_id": "product-uuid-9012",
                    "quantity": 2,
                }
            ],
            "cancelled_at": "2026-01-15T11:00:00Z",
        }

    provider = MessageProvider(
        provider="OrderService",
        consumer="InventoryService",
        pact_dir="tests/consumer/pacts",
        message_providers={
            "An OrderCreated event": produce_order_created,
            "An OrderCancelled event": produce_order_cancelled,
        },
    )
    return provider


def test_verify_order_event_pacts(message_provider):
    """Verify that OrderService produces messages matching consumer pacts."""
    with message_provider:
        pass  # Verification runs inside context manager
```

### V3 Generators

Generators produce dynamic values in requests during provider verification (e.g., timestamps that must be current):

```python
# V3 only
from pact.matchers import DateTime, Date, RandomInt, Regex, UUID

with_request(
    method="POST",
    path="/api/tokens",
    body={
        "requested_at": DateTime(
            format="yyyy-MM-dd'T'HH:mm:ssZ",
            value="2026-01-15T10:30:00+0000",
        ),
        "nonce": UUID(),           # Generate a random UUID for each verification
        "count": RandomInt(1, 100), # Random integer between 1 and 100
    },
)
```
