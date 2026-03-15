# testcontainers-python — Basics

**Official Documentation:** https://testcontainers-python.readthedocs.io/
**GitHub:** https://github.com/testcontainers/testcontainers-python
**Version:** 4.14.1+ (Python ≥ 3.10)

---

## Table of Contents

1. [Installation](#installation)
2. [Core Concepts](#core-concepts)
3. [PostgreSQL Container](#postgresql-container)
4. [MySQL Container](#mysql-container)
5. [MongoDB Container](#mongodb-container)
6. [Redis Container](#redis-container)
7. [Pytest Fixture Patterns](#pytest-fixture-patterns)
8. [Wait Strategies](#wait-strategies)
9. [GenericContainer](#genericcontainer)
10. [Configuration](#configuration)

---

## Installation

```bash
# Core package (GenericContainer, DockerCompose, Network)
pip install testcontainers

# With specific extras
pip install "testcontainers[postgres]"
pip install "testcontainers[mysql]"
pip install "testcontainers[mongodb]"
pip install "testcontainers[redis]"
pip install "testcontainers[kafka]"
pip install "testcontainers[rabbitmq]"
pip install "testcontainers[elasticsearch]"
pip install "testcontainers[minio]"
pip install "testcontainers[localstack]"
pip install "testcontainers[cassandra]"
pip install "testcontainers[clickhouse]"

# Multiple extras
pip install "testcontainers[postgres,redis,kafka,rabbitmq]"
```

**Full extras catalogue:**

| Extra | Additional deps |
|-------|----------------|
| `postgres` | psycopg2 installed separately |
| `mysql` | sqlalchemy>=2, pymysql[rsa]>=1 |
| `mongodb` | pymongo>=4 |
| `redis` | redis>=7 |
| `kafka` | *(none — confluent-kafka or kafka-python separately)* |
| `rabbitmq` | pika>=1 |
| `minio` | minio>=7 |
| `localstack` | boto3>=1 |
| `elasticsearch` | elasticsearch>=8 |
| `cassandra` | cassandra-driver |
| `clickhouse` | clickhouse-driver |
| `neo4j` | neo4j>=6 |
| `selenium` | selenium>=4 |

**Requirements:**
- Python 3.10+
- Docker Engine running and accessible
- Docker socket at `/var/run/docker.sock` (default) or `DOCKER_HOST` env var

---

## Core Concepts

### Context Manager Protocol

Every container implements the context manager protocol (`__enter__` / `__exit__`):

```python
from testcontainers.postgres import PostgresContainer

# Automatic start/stop via context manager
with PostgresContainer("postgres:16-alpine") as pg:
    # Container is running here
    url = pg.get_connection_url()
    # ... use url
# Container is stopped and removed here
```

### DockerContainer Base Class

All containers inherit from `DockerContainer` which provides the full fluent API:

```python
from testcontainers.core.container import DockerContainer

container = (
    DockerContainer("myimage:latest")
    .with_exposed_ports(8080, 8443)
    .with_env("ENV_VAR", "value")
    .with_volume_mapping("/host/path", "/container/path", "ro")
    .with_bind_ports(8080, 8080)
    .with_kwargs(mem_limit="512m", network_mode="bridge")
    .with_name("my-test-container")
    .with_command("--flag value")
)

with container as c:
    host = c.get_container_host_ip()
    port = c.get_exposed_port(8080)
```

**DockerContainer methods:**

| Method | Description |
|--------|-------------|
| `.with_exposed_ports(*ports)` | Expose and map ports (random host port) |
| `.with_bind_ports(container, host)` | Bind a fixed host port |
| `.with_env(key, value)` | Set environment variable |
| `.with_volume_mapping(host, container, mode)` | Mount volume |
| `.with_command(cmd)` | Override entrypoint command |
| `.with_name(name)` | Set container name |
| `.with_kwargs(**kwargs)` | Pass docker-py `run()` kwargs directly |
| `.with_wait_for(strategy)` | Set wait strategy |
| `.get_container_host_ip()` | Returns mapped host (usually `localhost`) |
| `.get_exposed_port(port)` | Returns mapped host port as string |
| `.exec(cmd)` | Execute command inside running container |
| `.get_logs()` | Return container stdout/stderr logs |

---

## PostgreSQL Container

```python
from testcontainers.postgres import PostgresContainer

# Default: postgres:latest, db=test, user=test, password=test
with PostgresContainer("postgres:16-alpine") as pg:
    url = pg.get_connection_url()
    # postgresql+psycopg2://test:test@localhost:{port}/test
    print(url)

# Custom credentials
with PostgresContainer(
    image="postgres:16",
    username="myuser",
    password="mysecret",
    dbname="mydb",
    port=5432,
    driver="psycopg2",  # changes URL scheme
) as pg:
    url = pg.get_connection_url()
    # postgresql+psycopg2://myuser:mysecret@localhost:{port}/mydb
```

**Constructor parameters:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `image` | `"postgres:latest"` | Docker image |
| `username` | `"test"` | DB username |
| `password` | `"test"` | DB password |
| `dbname` | `"test"` | Database name |
| `port` | `5432` | Container port |
| `driver` | `"psycopg2"` | SQLAlchemy driver suffix |

**Connection with SQLAlchemy:**

```python
from sqlalchemy import create_engine, text

with PostgresContainer("postgres:16-alpine") as pg:
    engine = create_engine(pg.get_connection_url())
    with engine.connect() as conn:
        result = conn.execute(text("SELECT version()"))
        print(result.scalar())
```

**With asyncpg:**

```python
import asyncio
import asyncpg

async def main():
    with PostgresContainer("postgres:16-alpine") as pg:
        # Convert psycopg2 URL to asyncpg-compatible
        url = pg.get_connection_url().replace("postgresql+psycopg2://", "postgresql://")
        conn = await asyncpg.connect(url)
        version = await conn.fetchval("SELECT version()")
        await conn.close()

asyncio.run(main())
```

---

## MySQL Container

```python
from testcontainers.mysql import MySqlContainer

with MySqlContainer("mysql:8.0") as mysql:
    url = mysql.get_connection_url()
    # mysql+pymysql://test:test@localhost:{port}/test

# Custom config
with MySqlContainer(
    image="mysql:8.0",
    username="appuser",
    password="appsecret",
    dbname="appdb",
) as mysql:
    engine = create_engine(mysql.get_connection_url())
```

**Note:** MySQL 8.0 containers take longer to be ready. The built-in wait strategy
checks the port, but you may need to wait for `"ready for connections"` in logs:

```python
from testcontainers.core.waiting_utils import wait_for_logs

with MySqlContainer("mysql:8.0") as mysql:
    wait_for_logs(mysql, "ready for connections", timeout=60)
    url = mysql.get_connection_url()
```

---

## MongoDB Container

```python
from testcontainers.mongodb import MongoDbContainer
from pymongo import MongoClient

with MongoDbContainer("mongo:7") as mongo:
    client = MongoClient(mongo.get_connection_url())
    db = client["testdb"]
    collection = db["users"]
    collection.insert_one({"name": "Alice", "age": 30})
    doc = collection.find_one({"name": "Alice"})
    assert doc["age"] == 30

# With authentication
with MongoDbContainer(
    image="mongo:7",
    username="admin",
    password="secret",
) as mongo:
    url = mongo.get_connection_url()
    # mongodb://admin:secret@localhost:{port}
```

---

## Redis Container

```python
from testcontainers.redis import RedisContainer
import redis

with RedisContainer("redis:7-alpine") as redis_container:
    host = redis_container.get_container_host_ip()
    port = int(redis_container.get_exposed_port(6379))

    client = redis.Redis(host=host, port=port, decode_responses=True)
    client.set("key", "value")
    assert client.get("key") == "value"
    client.delete("key")
```

**With redis-py URL:**

```python
with RedisContainer("redis:7-alpine") as redis_container:
    url = redis_container.get_connection_url()
    # redis://localhost:{port}
    client = redis.Redis.from_url(url)
```

**AsyncRedis:**

```python
import redis.asyncio as aioredis

async def test_async_redis():
    with RedisContainer("redis:7-alpine") as redis_container:
        url = redis_container.get_connection_url()
        client = aioredis.from_url(url)
        await client.set("key", "value")
        value = await client.get("key")
        assert value == b"value"
        await client.aclose()
```

---

## Pytest Fixture Patterns

### Session-Scoped (Recommended)

One container for the entire test session. Fastest approach.

```python
# tests/conftest.py
import pytest
from testcontainers.postgres import PostgresContainer
from testcontainers.redis import RedisContainer

@pytest.fixture(scope="session")
def postgres_container():
    """Start PostgreSQL container once for the entire test session."""
    with PostgresContainer("postgres:16-alpine") as pg:
        yield pg

@pytest.fixture(scope="session")
def redis_container():
    """Start Redis container once for the entire test session."""
    with RedisContainer("redis:7-alpine") as redis:
        yield redis

@pytest.fixture(scope="session")
def db_engine(postgres_container):
    from sqlalchemy import create_engine
    from myapp.models import Base

    engine = create_engine(postgres_container.get_connection_url())
    Base.metadata.create_all(engine)
    yield engine
    Base.metadata.drop_all(engine)

@pytest.fixture
def db_session(db_engine):
    """Per-test session with savepoint rollback."""
    from sqlalchemy.orm import sessionmaker
    conn = db_engine.connect()
    trans = conn.begin()
    Session = sessionmaker(bind=conn)
    session = Session()
    nested = conn.begin_nested()  # SAVEPOINT

    yield session

    session.close()
    nested.rollback()
    trans.rollback()
    conn.close()
```

### Module-Scoped

New container per test module. Balance between speed and isolation.

```python
@pytest.fixture(scope="module")
def postgres_container():
    with PostgresContainer("postgres:16-alpine") as pg:
        yield pg
```

### Function-Scoped (Slowest, Most Isolated)

New container per test. Only use for tests that require completely fresh state.

```python
@pytest.fixture
def postgres_container():
    with PostgresContainer("postgres:16-alpine") as pg:
        yield pg
```

### request.addfinalizer Pattern

For complex teardown in session fixtures:

```python
@pytest.fixture(scope="session")
def postgres_container(request):
    pg = PostgresContainer("postgres:16-alpine")
    pg.start()
    request.addfinalizer(pg.stop)
    yield pg
```

---

## Wait Strategies

```python
from testcontainers.core.waiting_utils import (
    wait_for_logs,
    wait_container_is_ready,
)
from testcontainers.core.wait.http_wait import HttpWaitStrategy
from testcontainers.core.wait.log_wait import LogMessageWaitStrategy
from testcontainers.core.wait.exec_wait import ExecWaitStrategy
```

### Log Message Wait

```python
# Wait for a specific log pattern (regex)
with DockerContainer("myapp:latest").with_exposed_ports(8080) as c:
    wait_for_logs(c, "Server started", timeout=60)
    # or with regex:
    wait_for_logs(c, r".*Listening on port \d+.*", timeout=60)
```

### HTTP Wait

```python
container = (
    DockerContainer("myapp:latest")
    .with_exposed_ports(8080)
    .with_wait_for(
        HttpWaitStrategy("/health")
        .with_status_code(200)
        .with_startup_timeout(60)
        .with_response_body_contains("ok")
    )
)
```

### Exec Wait

```python
container = (
    DockerContainer("myapp:latest")
    .with_wait_for(
        ExecWaitStrategy(["pg_isready", "-U", "postgres"])
    )
)
```

### Port Wait (default for most containers)

```python
container = (
    DockerContainer("myapp:latest")
    .with_exposed_ports(8080)
    # Default: waits for the port to be accessible
)
```

---

## GenericContainer

For images without dedicated modules:

```python
from testcontainers.core.container import DockerContainer

@pytest.fixture(scope="session")
def meilisearch_container():
    with (
        DockerContainer("getmeili/meilisearch:v1.5")
        .with_exposed_ports(7700)
        .with_env("MEILI_MASTER_KEY", "test-master-key")
        .with_env("MEILI_ENV", "development")
    ) as container:
        wait_for_logs(container, "Server listening on", timeout=30)
        yield container

@pytest.fixture(scope="session")
def meilisearch_url(meilisearch_container):
    host = meilisearch_container.get_container_host_ip()
    port = meilisearch_container.get_exposed_port(7700)
    return f"http://{host}:{port}"
```

---

## Configuration

### Environment Variables

```bash
# Disable Ryuk (auto-cleanup daemon) — useful in some CI environments
TESTCONTAINERS_RYUK_DISABLED=true

# Custom Docker host
DOCKER_HOST=unix:///var/run/docker.sock
DOCKER_HOST=tcp://localhost:2375

# Ryuk image override
TESTCONTAINERS_RYUK_IMAGE=testcontainers/ryuk:0.7.0

# Disable SSL verification for Docker
DOCKER_TLS_VERIFY=0
```

### ~/.testcontainers.properties

```properties
# Enable container reuse across runs (dev mode)
testcontainers.reuse.enable=true

# Custom Ryuk image
ryuk.container.image=testcontainers/ryuk:0.7.0
```

### Container Reuse (Dev Mode)

```python
from testcontainers.postgres import PostgresContainer

container = (
    PostgresContainer("postgres:16-alpine")
    .with_kwargs(labels={"testcontainers-session-id": "dev-reuse"})
)

with container as pg:
    # Container will be reused if testcontainers.reuse.enable=true
    yield pg
```

---

## Common Pitfalls

| Problem | Cause | Solution |
|---------|-------|----------|
| `ConnectionRefused` immediately | Container not ready | Use `wait_for_logs` or `HttpWaitStrategy` |
| Port `5432` not available | Using container port, not host port | Use `get_exposed_port(5432)` |
| Docker socket permission denied | User not in docker group | `sudo usermod -aG docker $USER` |
| `TimeoutError` starting container | Image pull slow or health check too strict | Increase `with_startup_timeout(120)` |
| MySQL not ready even after port open | MySQL logs "ready" twice | Wait for second "ready for connections" log |
| Elasticsearch OOM | Default JVM heap too large | `.with_env("ES_JAVA_OPTS", "-Xmx256m -Xms256m")` |
| Kafka consumer can't connect | Broker advertised host | Set `KAFKA_ADVERTISED_LISTENERS` to `localhost:{port}` |
