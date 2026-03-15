# testcontainers-python — Messaging Containers

**Official Documentation:** https://testcontainers-python.readthedocs.io/

---

## Table of Contents

1. [Kafka Container](#kafka-container)
2. [RabbitMQ Container](#rabbitmq-container)
3. [Pytest Fixtures for Messaging](#pytest-fixtures-for-messaging)
4. [Testing Producers and Consumers](#testing-producers-and-consumers)
5. [Celery with Real Broker](#celery-with-real-broker)

---

## Kafka Container

```python
from testcontainers.kafka import KafkaContainer

with KafkaContainer("confluentinc/cp-kafka:7.6.1") as kafka:
    bootstrap = kafka.get_bootstrap_server()
    # localhost:{random_port}
    print(f"Bootstrap servers: {bootstrap}")
```

**Constructor parameters:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `image` | `"confluentinc/cp-kafka:latest"` | Docker image (use specific version) |
| `port` | `9093` | Internal Kafka port |

**With confluent-kafka:**

```python
from confluent_kafka import Producer, Consumer, KafkaError
from testcontainers.kafka import KafkaContainer
import json

@pytest.fixture(scope="session")
def kafka_container():
    with KafkaContainer("confluentinc/cp-kafka:7.6.1") as kafka:
        yield kafka

@pytest.fixture(scope="session")
def kafka_bootstrap(kafka_container):
    return kafka_container.get_bootstrap_server()


def test_produce_and_consume(kafka_bootstrap):
    topic = "test-topic"

    # Create topic (auto-created on first produce with default config)
    producer = Producer({"bootstrap.servers": kafka_bootstrap})
    producer.produce(topic, key="key1", value=json.dumps({"event": "created"}))
    producer.flush(timeout=10)

    consumer = Consumer({
        "bootstrap.servers": kafka_bootstrap,
        "group.id":          "test-group",
        "auto.offset.reset": "earliest",
        "enable.auto.commit": False,
    })
    consumer.subscribe([topic])

    messages = []
    for _ in range(10):
        msg = consumer.poll(timeout=1.0)
        if msg is None:
            continue
        if msg.error():
            if msg.error().code() == KafkaError._PARTITION_EOF:
                break
            raise Exception(f"Kafka error: {msg.error()}")
        messages.append(json.loads(msg.value()))
        break  # got our message

    consumer.close()
    assert len(messages) == 1
    assert messages[0]["event"] == "created"
```

**With kafka-python:**

```python
from kafka import KafkaProducer, KafkaConsumer
import json

def test_kafka_python(kafka_bootstrap):
    topic = "test-events"

    producer = KafkaProducer(
        bootstrap_servers=kafka_bootstrap,
        value_serializer=lambda v: json.dumps(v).encode("utf-8"),
    )
    producer.send(topic, {"event_type": "order_placed", "order_id": 42})
    producer.flush()

    consumer = KafkaConsumer(
        topic,
        bootstrap_servers=kafka_bootstrap,
        auto_offset_reset="earliest",
        consumer_timeout_ms=5000,
        group_id="test-consumer",
        value_deserializer=lambda m: json.loads(m.decode("utf-8")),
    )

    received = [msg.value for msg in consumer]
    consumer.close()

    assert len(received) == 1
    assert received[0]["order_id"] == 42
```

**With aiokafka (async):**

```python
import asyncio
from aiokafka import AIOKafkaProducer, AIOKafkaConsumer

@pytest.mark.anyio
async def test_async_kafka(kafka_bootstrap):
    topic = "async-test-topic"

    producer = AIOKafkaProducer(bootstrap_servers=kafka_bootstrap)
    await producer.start()
    try:
        await producer.send_and_wait(
            topic,
            b'{"type": "user_created", "user_id": 1}',
        )
    finally:
        await producer.stop()

    consumer = AIOKafkaConsumer(
        topic,
        bootstrap_servers=kafka_bootstrap,
        auto_offset_reset="earliest",
        group_id="async-test-group",
    )
    await consumer.start()
    try:
        msg = await asyncio.wait_for(consumer.getone(), timeout=5.0)
        data = json.loads(msg.value)
        assert data["type"] == "user_created"
    finally:
        await consumer.stop()
```

---

## RabbitMQ Container

```python
from testcontainers.rabbitmq import RabbitMqContainer

with RabbitMqContainer("rabbitmq:3-management") as rmq:
    params = rmq.get_connection_params()
    # pika.ConnectionParameters object
    print(f"Host: {params.host}, Port: {params.port}")

# Or get components manually
with RabbitMqContainer("rabbitmq:3.12-management") as rmq:
    host     = rmq.get_container_host_ip()
    amqp_port = int(rmq.get_exposed_port(5672))
    mgmt_port = int(rmq.get_exposed_port(15672))
```

**Constructor parameters:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `image` | `"rabbitmq:latest"` | Docker image |
| `username` | `"guest"` | RabbitMQ username |
| `password` | `"guest"` | RabbitMQ password |
| `port` | `5672` | AMQP port |

**With pika:**

```python
import pika
from testcontainers.rabbitmq import RabbitMqContainer


@pytest.fixture(scope="session")
def rabbitmq_container():
    with RabbitMqContainer("rabbitmq:3-management") as rmq:
        yield rmq


def test_publish_and_consume(rabbitmq_container):
    params     = rabbitmq_container.get_connection_params()
    connection = pika.BlockingConnection(params)
    channel    = connection.channel()

    queue_name = "test-queue"
    channel.queue_declare(queue=queue_name, durable=False, auto_delete=True)

    # Publish
    channel.basic_publish(
        exchange="",
        routing_key=queue_name,
        body=b"Hello, RabbitMQ!",
        properties=pika.BasicProperties(delivery_mode=1),
    )

    # Consume
    method_frame, header_frame, body = channel.basic_get(queue_name, auto_ack=True)
    assert body == b"Hello, RabbitMQ!"

    channel.close()
    connection.close()
```

**With aio-pika (async):**

```python
import aio_pika

@pytest.mark.anyio
async def test_async_rabbitmq(rabbitmq_container):
    host = rabbitmq_container.get_container_host_ip()
    port = int(rabbitmq_container.get_exposed_port(5672))

    connection = await aio_pika.connect_robust(
        f"amqp://guest:guest@{host}:{port}/"
    )
    async with connection:
        channel = await connection.channel()
        queue   = await channel.declare_queue("async-test-queue", auto_delete=True)

        # Publish
        await channel.default_exchange.publish(
            aio_pika.Message(body=b'{"event": "test"}'),
            routing_key=queue.name,
        )

        # Consume one message
        async with queue.iterator(no_ack=True) as queue_iter:
            async for message in queue_iter:
                data = json.loads(message.body)
                assert data["event"] == "test"
                break
```

---

## Pytest Fixtures for Messaging

```python
# tests/conftest.py
import pytest
from testcontainers.kafka import KafkaContainer
from testcontainers.rabbitmq import RabbitMqContainer


@pytest.fixture(scope="session")
def kafka_container():
    with KafkaContainer("confluentinc/cp-kafka:7.6.1") as kafka:
        yield kafka


@pytest.fixture(scope="session")
def kafka_bootstrap(kafka_container):
    return kafka_container.get_bootstrap_server()


@pytest.fixture(scope="session")
def rabbitmq_container():
    with RabbitMqContainer("rabbitmq:3-management") as rmq:
        yield rmq


@pytest.fixture(scope="session")
def rabbitmq_params(rabbitmq_container):
    return rabbitmq_container.get_connection_params()
```

---

## Testing Producers and Consumers

### Event-Driven Architecture Pattern

```python
# myapp/events.py
from dataclasses import dataclass
from confluent_kafka import Producer
import json

@dataclass
class UserCreatedEvent:
    user_id: int
    username: str
    email: str

class EventPublisher:
    def __init__(self, bootstrap_servers: str):
        self.producer = Producer({"bootstrap.servers": bootstrap_servers})

    def publish_user_created(self, event: UserCreatedEvent):
        self.producer.produce(
            "user.created",
            key=str(event.user_id),
            value=json.dumps({
                "user_id": event.user_id,
                "username": event.username,
                "email": event.email,
            }),
        )
        self.producer.flush(timeout=10)


# tests/integration/test_events.py
import pytest
import json
from confluent_kafka import Consumer
from myapp.events import EventPublisher, UserCreatedEvent


def collect_messages(bootstrap: str, topic: str, count: int, timeout: float = 10.0):
    consumer = Consumer({
        "bootstrap.servers": bootstrap,
        "group.id":          f"test-{topic}-{uuid.uuid4()}",
        "auto.offset.reset": "earliest",
    })
    consumer.subscribe([topic])
    messages = []
    deadline = time.time() + timeout
    while len(messages) < count and time.time() < deadline:
        msg = consumer.poll(timeout=0.5)
        if msg and not msg.error():
            messages.append(json.loads(msg.value()))
    consumer.close()
    return messages


@pytest.mark.integration
def test_user_created_event_published(kafka_bootstrap):
    publisher = EventPublisher(kafka_bootstrap)
    event = UserCreatedEvent(user_id=1, username="alice", email="alice@example.com")
    publisher.publish_user_created(event)

    messages = collect_messages(kafka_bootstrap, "user.created", count=1)
    assert len(messages) == 1
    assert messages[0]["username"] == "alice"
    assert messages[0]["user_id"] == 1
```

---

## Celery with Real Broker

```python
# tests/conftest.py
import pytest
from testcontainers.redis import RedisContainer
from celery import Celery


@pytest.fixture(scope="session")
def redis_container():
    with RedisContainer("redis:7-alpine") as redis:
        yield redis


@pytest.fixture(scope="session")
def celery_app_real(redis_container):
    """Celery app configured to use real Redis broker+backend."""
    host = redis_container.get_container_host_ip()
    port = redis_container.get_exposed_port(6379)
    url  = f"redis://{host}:{port}/0"

    app = Celery("test_app")
    app.conf.update(
        broker_url=url,
        result_backend=url,
        task_always_eager=False,
        task_serializer="json",
        result_serializer="json",
        accept_content=["json"],
    )
    return app


@pytest.mark.integration
def test_real_celery_task(celery_app_real):
    from myapp.tasks import add

    result = add.apply_async(args=[2, 3])
    value = result.get(timeout=10)
    assert value == 5
    assert result.successful()
```
