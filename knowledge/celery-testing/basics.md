# Celery Testing — Basics

> **Source:** https://docs.celeryq.dev/en/stable/userguide/testing.html
> **Celery version:** 5.x

Celery ships a first-class pytest plugin (`celery.contrib.pytest`) that
provides fixtures for embedded workers, app instances, and configuration
overrides — all without requiring a running broker for most test scenarios.

---

## Plugin Setup

### Install

```bash
pip install "celery[pytest]"
# or separately:
pip install celery pytest-celery   # for the advanced Docker-based variant
```

### Enable the plugin (choose one)

```python
# 1 — via conftest.py (recommended)
# conftest.py
pytest_plugins = ("celery.contrib.pytest",)

# 2 — via pytest.ini / pyproject.toml
# pytest.ini
[pytest]
addopts = -p celery.contrib.pytest

# 3 — via environment variable (CI-friendly)
PYTEST_PLUGINS=celery.contrib.pytest pytest
```

> **Warning:** `celery.contrib.pytest` (unit/integration, mock-based) and
> `pytest-celery >= 1.0.0` (smoke tests, Docker-based) are **not** compatible.
> Pick one approach per test suite.

---

## Configuration: `CELERY_TASK_ALWAYS_EAGER` / `CELERY_TASK_EAGER_PROPAGATES`

The legacy eager mode executes tasks inline in the caller's thread without
any broker involvement. Celery's own documentation notes it is **not suitable
for proper unit tests** because eager mode bypasses the serialization,
routing, and retry machinery entirely. Use it only for the simplest sanity
checks.

```python
# Django settings — legacy approach
CELERY_TASK_ALWAYS_EAGER    = True   # all tasks run inline, synchronously
CELERY_TASK_EAGER_PROPAGATES = True  # exceptions raised instead of swallowed
```

Modern equivalent via pytest fixture (`celery_config` — see below):

```python
@pytest.fixture(scope="session")
def celery_config():
    return {
        "task_always_eager":      True,
        "task_eager_propagates":  True,
    }
```

---

## Core Fixtures

All fixtures are provided by `celery.contrib.pytest`. Unless stated otherwise,
they are **function-scoped** (fresh per test).

---

### `celery_app` — test Celery app instance

Returns a fully configured `Celery` application wired to in-memory
transports. Register tasks on it with `@celery_app.task`.

```python
def test_inline_task(celery_app, celery_worker):
    @celery_app.task
    def multiply(x, y):
        return x * y

    celery_worker.reload()           # make the worker aware of the new task
    result = multiply.delay(4, 4)
    assert result.get(timeout=10) == 16
```

---

### `celery_worker` — embedded worker

Starts a real Celery worker in a **background thread** for the duration of
the test. Default timeout for `.get()` calls is 10 seconds.

```python
def test_task_runs_in_worker(celery_app, celery_worker):
    @celery_app.task
    def add(x, y):
        return x + y

    celery_worker.reload()
    assert add.delay(3, 7).get(timeout=10) == 10
```

The worker shuts down automatically after the test.

---

### `celery_config` — configuration override (session-scoped)

Return a dict of Celery config keys to override for the whole test session.

```python
# conftest.py
import pytest

@pytest.fixture(scope="session")
def celery_config():
    return {
        "broker_url":    "memory://",       # in-process broker (no Redis/RabbitMQ)
        "result_backend": "cache+memory://", # in-process result store
        "task_serializer":   "json",
        "result_serializer": "json",
        "accept_content":    ["json"],
        "task_always_eager": False,          # use real worker, not eager mode
    }
```

---

### `celery_worker_parameters` — worker initialization

Fine-tune `WorkController` parameters: queues, concurrency, pool.

```python
@pytest.fixture(scope="session")
def celery_worker_parameters():
    return {
        "queues":      ("default", "high-priority"),
        "concurrency": 2,
        "loglevel":    "info",
    }
```

---

### `celery_enable_logging` — enable logging in embedded workers

Override to `True` to see worker log output during tests (useful for
debugging retry logic).

```python
@pytest.fixture(scope="session")
def celery_enable_logging():
    return True
```

---

### `celery_includes` — auto-import task modules

Ensures task modules are imported before the worker starts so the worker
discovers all registered tasks.

```python
@pytest.fixture(scope="session")
def celery_includes():
    return [
        "myapp.tasks",
        "myapp.notifications.tasks",
    ]
```

---

### `celery_worker_pool` — execution pool type

```python
@pytest.fixture(scope="session")
def celery_worker_pool():
    return "solo"    # single-threaded; "prefork", "gevent", "eventlet" also valid
```

---

### `@pytest.mark.celery` — per-test configuration override

```python
@pytest.mark.celery(result_backend="redis://localhost:6379/1")
def test_redis_backend(celery_app, celery_worker):
    @celery_app.task
    def noop():
        return "ok"
    celery_worker.reload()
    assert noop.delay().get(timeout=5) == "ok"
```

---

## Task Testing Strategies

### Strategy 1: Direct call (no broker, no worker)

Call the task function directly as a plain Python function. No Celery
machinery involved. Fastest.

```python
# tasks.py
from celery import shared_task

@shared_task
def add(x, y):
    return x + y

# test_tasks.py
from myapp.tasks import add

def test_add_direct():
    assert add(3, 4) == 7   # direct call, no Celery at all
```

---

### Strategy 2: `.apply()` — synchronous execution with full Celery machinery

`.apply()` runs the task **in the current thread**, goes through
serialization/deserialization, and returns an `EagerResult`. Retry logic,
hooks, and signals all fire.

```python
from myapp.tasks import add

def test_add_apply():
    result = add.apply(args=(3, 4))
    assert result.successful()
    assert result.get() == 7
    assert result.state == "SUCCESS"
```

Pass kwargs:

```python
result = add.apply(kwargs={"x": 3, "y": 4})
```

Override task options per-call:

```python
result = add.apply(args=(1, 2), retries=0, throw=True)
```

---

### Strategy 3: `.delay()` / `.apply_async()` with embedded worker

Requires the `celery_worker` fixture. The task is serialized, routed through
the in-memory broker, picked up by the embedded worker, and executed in a
background thread.

```python
from myapp.tasks import send_email

def test_send_email_via_worker(celery_worker):
    result = send_email.delay(to="test@example.com", subject="Hi")
    assert result.get(timeout=10) == "sent"
```

---

### Strategy 4: Mock the task body, test only the Celery glue

When you only want to verify routing, countdown, queue selection, or that a
task was called — mock the underlying function.

```python
from unittest.mock import patch, MagicMock
from myapp.tasks import process_order

def test_process_order_called_with_correct_args():
    with patch("myapp.tasks.process_order.apply_async") as mock_async:
        from myapp.views import checkout_view
        checkout_view(order_id=99)
        mock_async.assert_called_once_with(
            args=(99,),
            countdown=5,
            queue="orders",
        )
```

Mock the product-layer side effect:

```python
from unittest.mock import patch
from myapp.tasks import send_order

class TestSendOrder:
    @patch("myapp.tasks.Product.order")
    def test_success(self, mock_product_order):
        send_order(product_pk=1, count=3, price="30.00")
        mock_product_order.assert_called_once_with(3, "30.00")

    @patch("myapp.tasks.Product.order", side_effect=OutOfStockError)
    def test_out_of_stock_raises(self, _mock):
        with pytest.raises(OutOfStockError):
            send_order(product_pk=1, count=3, price="30.00")
```

---

### Strategy 5: Real broker via testcontainers

For smoke tests that must exercise the real broker and result backend.

```python
import pytest
from testcontainers.redis import RedisContainer
from testcontainers.rabbitmq import RabbitMqContainer
from celery import Celery

@pytest.fixture(scope="session")
def redis_container():
    with RedisContainer() as redis:
        yield redis

@pytest.fixture(scope="session")
def rabbitmq_container():
    with RabbitMqContainer() as rmq:
        yield rmq

@pytest.fixture(scope="session")
def celery_config(redis_container, rabbitmq_container):
    return {
        "broker_url":    rabbitmq_container.get_connection_url(),
        "result_backend": f"redis://{redis_container.get_container_host_ip()}:"
                          f"{redis_container.get_exposed_port(6379)}/0",
    }
```

---

## Testing Task Results: `AsyncResult`

```python
from celery.result import AsyncResult
from myapp.tasks import long_running

# After dispatching with a worker:
result: AsyncResult = long_running.delay()

# Check state
result.state       # "PENDING" | "STARTED" | "SUCCESS" | "FAILURE" | "RETRY"
result.ready()     # True when SUCCESS or FAILURE
result.successful()  # True only when SUCCESS
result.failed()    # True only when FAILURE

# Block and retrieve
value = result.get(timeout=30)           # raises on failure by default
value = result.get(timeout=30, propagate=False)  # returns exception instead

# Release backend resources (important!)
result.forget()   # frees result backend memory

# Inspect a task by known ID:
r = AsyncResult("some-task-uuid")
print(r.state, r.result)
```

Task states:

| State | Meaning |
|---|---|
| `PENDING` | Waiting to be executed or task ID unknown |
| `STARTED` | Worker has picked up the task (`track_started=True` required) |
| `SUCCESS` | Completed successfully; `result.result` holds the return value |
| `FAILURE` | Raised an unhandled exception; `result.result` holds the exception |
| `RETRY` | Being retried; `result.result` holds the triggering exception |
| `REVOKED` | Task was cancelled before execution |

Custom states (progress tracking):

```python
@app.task(bind=True)
def bulk_import(self, file_path):
    rows = load_csv(file_path)
    for i, row in enumerate(rows):
        process(row)
        self.update_state(
            state="PROGRESS",
            meta={"current": i + 1, "total": len(rows)},
        )
    return {"imported": len(rows)}

# In a test:
def test_bulk_import_reports_progress(celery_worker):
    result = bulk_import.delay("test.csv")
    final = result.get(timeout=30)
    assert final["imported"] > 0
```

---

## Testing Error Handling and Retry Behavior

### Basic retry test

```python
import pytest
from unittest.mock import patch, call
from myapp.tasks import fetch_from_api

def test_task_retries_on_timeout():
    """Verify the task retries exactly max_retries times then raises."""
    call_count = 0

    def flaky_api(*args, **kwargs):
        nonlocal call_count
        call_count += 1
        raise TimeoutError("network unavailable")

    with patch("myapp.tasks.requests.get", side_effect=flaky_api):
        result = fetch_from_api.apply(args=("https://api.example.com",), throw=False)
        # task_always_eager=True: retries execute synchronously
        assert result.failed()
        assert isinstance(result.result, MaxRetriesExceededError)
```

### Testing `MaxRetriesExceededError`

```python
from celery.exceptions import MaxRetriesExceededError

@app.task(bind=True, max_retries=3, default_retry_delay=0)
def unreliable_task(self, url):
    try:
        return requests.get(url).json()
    except Exception as exc:
        raise self.retry(exc=exc)

def test_max_retries_exceeded():
    with patch("myapp.tasks.requests.get", side_effect=ConnectionError("down")):
        with pytest.raises(MaxRetriesExceededError):
            unreliable_task.apply(args=("http://down.example.com",), throw=True)
```

### Testing exponential backoff (assert countdown values)

```python
@app.task(bind=True, autoretry_for=(RequestException,), retry_backoff=True, max_retries=4)
def call_service(self, payload):
    return requests.post("https://svc/endpoint", json=payload).json()

def test_call_service_uses_backoff(mocker):
    mocker.patch("myapp.tasks.requests.post", side_effect=RequestException)
    apply_async_spy = mocker.spy(call_service, "apply_async")
    call_service.apply(args=({"key": "val"},), throw=False)
    # Verify retry was scheduled (backoff means countdown > 0 on subsequent attempts)
    assert apply_async_spy.call_count >= 1
```

### Testing that specific exceptions are NOT retried

```python
@app.task(autoretry_for=(IOError,), dont_autoretry_for=(PermissionError,))
def read_file(path):
    with open(path) as f:
        return f.read()

def test_permission_error_not_retried():
    with patch("builtins.open", side_effect=PermissionError("denied")):
        result = read_file.apply(args=("/secret/file",), throw=False)
        assert result.failed()
        assert isinstance(result.result, PermissionError)
```

---

## Testing Chains, Chords, and Groups

All canvas primitives support `.apply()` for synchronous testing.

### `chain`

```python
from celery import chain
from myapp.tasks import add, multiply

def test_chain_passes_result():
    # chain: add(2, 2) → multiply by 4 → add 10
    result = chain(
        add.s(2, 2),       # returns 4
        multiply.s(4),     # receives 4, returns 16
        add.s(10),         # receives 16, returns 26
    ).apply()
    assert result.get() == 26

# Access intermediate results:
def test_chain_intermediate_results():
    result = chain(add.s(4, 4), multiply.s(2), add.s(1)).apply()
    assert result.get() == 17              # (4+4)*2 + 1
    assert result.parent.get() == 16       # (4+4)*2
    assert result.parent.parent.get() == 8 # 4+4

# Pipe operator shorthand:
def test_chain_pipe_operator():
    result = (add.s(1, 1) | multiply.s(5) | add.s(0)).apply()
    assert result.get() == 10
```

### `chord`

```python
from celery import chord
from myapp.tasks import add, tsum

def test_chord_executes_header_then_callback():
    # header: [add(0,0), add(1,1), add(2,2), add(3,3)]
    # callback: tsum([0, 2, 4, 6]) = 12
    header   = [add.s(i, i) for i in range(4)]
    callback = tsum.s()

    result = chord(header)(callback)
    assert result.get(timeout=10) == 12

# Inline apply:
def test_chord_apply():
    result = chord([add.s(i, i) for i in range(5)], tsum.s()).apply()
    assert result.get() == 20   # 0+2+4+6+8

# Immutable callback (ignores header results):
def test_chord_immutable_callback(celery_worker):
    from myapp.tasks import notify
    c = chord(
        [add.s(i, i) for i in range(3)],
        notify.si("Import complete"),   # .si() = immutable signature
    )
    result = c.delay()
    assert result.get(timeout=10) == "Import complete"
```

### `group`

```python
from celery import group
from myapp.tasks import add

def test_group_runs_in_parallel():
    job = group(add.s(i, i) for i in range(5))
    result = job.apply()
    assert result.get() == [0, 2, 4, 6, 8]

def test_group_result_inspection():
    job = group(add.s(i, i) for i in range(4))
    result = job.apply()
    assert result.successful()
    assert result.ready()
    assert result.completed_count() == 4

# Group embedded in a chain (auto-upgrades to chord):
def test_group_in_chain():
    pipeline = (
        group(add.s(i, i) for i in range(4))
        | tsum.s()
    )
    result = pipeline.apply()
    assert result.get() == 12   # 0+2+4+6
```

---

## Testing Periodic Tasks (Beat)

Celery Beat determines *when* to dispatch tasks; the tasks themselves are
ordinary Celery tasks. Test the task logic independently from the schedule.

### Verify schedule configuration

```python
from celery.schedules import crontab
from myapp.celery import app

def test_periodic_task_is_registered():
    assert "cleanup-old-sessions" in app.conf.beat_schedule

def test_periodic_task_runs_daily_at_midnight():
    schedule = app.conf.beat_schedule["cleanup-old-sessions"]["schedule"]
    assert isinstance(schedule, crontab)
    assert schedule.hour == {0}
    assert schedule.minute == {0}
```

### Mock `datetime.now` / `datetime.utcnow` for time-sensitive tasks

```python
import datetime
from unittest.mock import patch
from myapp.tasks import archive_old_orders

FIXED_NOW = datetime.datetime(2025, 6, 15, 12, 0, 0)

def test_archive_orders_uses_cutoff_date():
    with patch("myapp.tasks.datetime") as mock_dt:
        mock_dt.utcnow.return_value = FIXED_NOW
        mock_dt.timedelta = datetime.timedelta  # keep timedelta real

        result = archive_old_orders.apply()
        assert result.successful()

        # Verify that orders older than 90 days from FIXED_NOW were selected
        cutoff = FIXED_NOW - datetime.timedelta(days=90)
        # assert query was called with correct cutoff ...

# Using freezegun (recommended for complex time-dependent tasks):
# pip install freezegun
from freezegun import freeze_time

@freeze_time("2025-06-15 12:00:00")
def test_archive_orders_frozen_time():
    result = archive_old_orders.apply()
    assert result.successful()
```

---

## Testing Task Hooks

Override task hooks by subclassing `Task`.

```python
import pytest
from celery import Task
from unittest.mock import MagicMock, patch

# tasks.py
class NotifyingTask(Task):
    def on_success(self, retval, task_id, args, kwargs):
        metrics.increment("task.success", tags={"task": self.name})

    def on_failure(self, exc, task_id, args, kwargs, einfo):
        metrics.increment("task.failure", tags={"task": self.name})
        alert_ops(f"Task {self.name} failed: {exc}")

    def on_retry(self, exc, task_id, args, kwargs, einfo):
        metrics.increment("task.retry", tags={"task": self.name})

    def after_return(self, status, retval, task_id, args, kwargs, einfo):
        db.close_old_connections()

@app.task(base=NotifyingTask, bind=True)
def important_job(self, item_id):
    return process(item_id)

# tests/test_hooks.py
def test_on_success_fires_metric():
    with patch("myapp.tasks.metrics") as mock_metrics:
        result = important_job.apply(args=(42,))
        assert result.successful()
        mock_metrics.increment.assert_called_once_with(
            "task.success", tags={"task": "myapp.tasks.important_job"}
        )

def test_on_failure_fires_alert():
    with patch("myapp.tasks.process", side_effect=ValueError("bad item")), \
         patch("myapp.tasks.metrics") as mock_metrics, \
         patch("myapp.tasks.alert_ops") as mock_alert:
        result = important_job.apply(args=(99,), throw=False)
        assert result.failed()
        mock_metrics.increment.assert_called_with(
            "task.failure", tags={"task": "myapp.tasks.important_job"}
        )
        mock_alert.assert_called_once()

def test_after_return_always_closes_connections():
    with patch("myapp.tasks.db") as mock_db:
        # Success path
        important_job.apply(args=(1,))
        mock_db.close_old_connections.assert_called_once()

    with patch("myapp.tasks.db") as mock_db, \
         patch("myapp.tasks.process", side_effect=RuntimeError):
        # Failure path
        important_job.apply(args=(1,), throw=False)
        mock_db.close_old_connections.assert_called_once()
```

---

## Testing Signals

Connect signal handlers temporarily in tests using the `connect` /
`disconnect` pattern or `pytest-mock`.

```python
from celery.signals import (
    task_prerun,
    task_postrun,
    task_success,
    task_failure,
    task_retry,
)
from myapp.tasks import add

def test_task_prerun_fires():
    received = []

    def handler(sender=None, task_id=None, task=None, args=None, kwargs=None, **kw):
        received.append({"task_id": task_id, "args": args})

    task_prerun.connect(handler)
    try:
        add.apply(args=(1, 2))
        assert len(received) == 1
        assert received[0]["args"] == (1, 2)
    finally:
        task_prerun.disconnect(handler)

def test_task_success_signal():
    results = []
    task_success.connect(lambda sender=None, result=None, **kw:
                         results.append(result), weak=False)
    try:
        add.apply(args=(3, 4))
        assert results == [7]
    finally:
        task_success.disconnect()

def test_task_failure_signal():
    errors = []

    def on_failure(sender=None, task_id=None, exception=None, **kw):
        errors.append(exception)

    task_failure.connect(on_failure)
    try:
        from myapp.tasks import failing_task
        failing_task.apply(throw=False)
        assert len(errors) == 1
        assert isinstance(errors[0], ValueError)
    finally:
        task_failure.disconnect(on_failure)

def test_task_retry_signal(mocker):
    retry_events = []
    task_retry.connect(
        lambda sender=None, request=None, reason=None, **kw:
            retry_events.append(reason),
        weak=False,
    )
    try:
        with mocker.patch("myapp.tasks.external_service", side_effect=IOError):
            from myapp.tasks import call_external
            call_external.apply(throw=False)
        assert len(retry_events) > 0
    finally:
        task_retry.disconnect()
```

---

## `@shared_task` Testing Patterns

`@shared_task` binds to whichever Celery app is current at call time.
This makes it suitable for reusable Django apps.

```python
# myapp/tasks.py
from celery import shared_task

@shared_task
def send_welcome_email(user_id):
    user = User.objects.get(pk=user_id)
    send_mail(
        subject="Welcome!",
        message=f"Hi {user.username}",
        from_email="noreply@example.com",
        recipient_list=[user.email],
    )
    return user.email

# tests/test_tasks.py
import pytest
from unittest.mock import patch
from myapp.tasks import send_welcome_email

# Direct call — no Celery app needed
def test_send_welcome_email_direct(db):
    user = UserFactory.create()
    with patch("django.core.mail.send_mail") as mock_mail:
        result = send_welcome_email(user.pk)
        assert result == user.email
        mock_mail.assert_called_once_with(
            subject="Welcome!",
            message=f"Hi {user.username}",
            from_email="noreply@example.com",
            recipient_list=[user.email],
        )

# Via .apply() — uses the test Celery app
def test_send_welcome_email_apply(db):
    user = UserFactory.create()
    with patch("django.core.mail.send_mail"):
        result = send_welcome_email.apply(args=(user.pk,))
        assert result.successful()
        assert result.get() == user.email
```

---

## Django Integration

### Project structure

```
myproject/
├── myproject/
│   ├── celery.py       # Celery app definition
│   └── settings.py
├── myapp/
│   └── tasks.py
└── tests/
    ├── conftest.py
    └── test_tasks.py
```

### `myproject/celery.py`

```python
import os
from celery import Celery

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "myproject.settings")

app = Celery("myproject")
app.config_from_object("django.conf:settings", namespace="CELERY")
app.autodiscover_tasks()
```

### `tests/conftest.py` (Django + Celery)

```python
import pytest
import django
from celery.contrib.pytest import celery_app, celery_worker  # noqa: F401 — re-export

pytest_plugins = ("celery.contrib.pytest",)

@pytest.fixture(scope="session")
def celery_config():
    return {
        "broker_url":              "memory://",
        "result_backend":          "cache+memory://",
        "task_always_eager":       False,
        "task_eager_propagates":   True,
        "task_store_eager_result": True,
    }

@pytest.fixture(scope="session")
def celery_includes():
    return ["myapp.tasks"]
```

### `myapp/tests/test_tasks.py`

```python
import pytest
from unittest.mock import patch
from myapp.tasks import send_invoice, process_refund
from myapp.models import Invoice, Order

@pytest.mark.django_db
def test_send_invoice_marks_as_sent():
    invoice = InvoiceFactory.create(sent=False)
    with patch("myapp.tasks.send_email_via_ses") as mock_send:
        result = send_invoice.apply(args=(invoice.pk,))
        assert result.successful()
    invoice.refresh_from_db()
    assert invoice.sent is True
    mock_send.assert_called_once()

@pytest.mark.django_db
def test_process_refund_via_worker(celery_worker):
    order = OrderFactory.create(status="paid", amount=100)
    result = process_refund.delay(order.pk, amount=50)
    final = result.get(timeout=15)
    assert final["refunded"] == 50
    order.refresh_from_db()
    assert order.status == "partially_refunded"
```

---

## Complete Real-World Example: Task with Retry Logic, Fully Tested

### The task

```python
# myapp/tasks.py
import logging
from celery import shared_task
from celery.exceptions import MaxRetriesExceededError
import requests

logger = logging.getLogger(__name__)

class WebhookTask(shared_task.__class__):
    """Custom task class with lifecycle hooks for observability."""

    def on_success(self, retval, task_id, args, kwargs):
        logger.info("Webhook delivered: task_id=%s url=%s", task_id, args[0])

    def on_failure(self, exc, task_id, args, kwargs, einfo):
        logger.error(
            "Webhook failed permanently: task_id=%s url=%s error=%s",
            task_id, args[0], exc,
        )
        # Persist failure record to DB
        from myapp.models import WebhookDelivery
        WebhookDelivery.objects.filter(task_id=task_id).update(
            status="failed", error=str(exc)
        )

    def on_retry(self, exc, task_id, args, kwargs, einfo):
        logger.warning(
            "Webhook retry scheduled: task_id=%s attempt=%d",
            task_id, self.request.retries,
        )

    def after_return(self, status, retval, task_id, args, kwargs, einfo):
        logger.debug("Webhook task finished: task_id=%s status=%s", task_id, status)


@shared_task(
    bind=True,
    base=WebhookTask,
    max_retries=5,
    default_retry_delay=60,       # 60 seconds base delay
    autoretry_for=(requests.ConnectionError, requests.Timeout),
    retry_backoff=True,           # exponential: 1s, 2s, 4s, 8s, 16s ...
    retry_backoff_max=600,        # cap at 10 minutes
    retry_jitter=True,            # randomise to avoid thundering herd
    name="myapp.tasks.deliver_webhook",
)
def deliver_webhook(self, url: str, payload: dict, secret: str = "") -> dict:
    """
    POST payload to url, with automatic exponential-backoff retries.
    Returns the response JSON on success.
    """
    headers = {"Content-Type": "application/json"}
    if secret:
        import hmac, hashlib, json
        sig = hmac.new(
            secret.encode(), json.dumps(payload).encode(), hashlib.sha256
        ).hexdigest()
        headers["X-Signature-256"] = f"sha256={sig}"

    resp = requests.post(url, json=payload, headers=headers, timeout=10)
    resp.raise_for_status()   # 4xx / 5xx → raise; autoretry does NOT cover these
    return {"status": resp.status_code, "body": resp.json()}
```

### The tests

```python
# tests/test_deliver_webhook.py
import pytest
import requests
from unittest.mock import patch, MagicMock, call
from celery.exceptions import MaxRetriesExceededError
from celery.signals import task_failure, task_success, task_retry

from myapp.tasks import deliver_webhook


# ── helpers ────────────────────────────────────────────────────────────────

def make_response(status_code: int, json_body: dict) -> MagicMock:
    resp = MagicMock()
    resp.status_code = status_code
    resp.json.return_value = json_body
    resp.raise_for_status.side_effect = (
        None if status_code < 400
        else requests.HTTPError(response=resp)
    )
    return resp


# ── success path ───────────────────────────────────────────────────────────

def test_deliver_webhook_success():
    with patch("myapp.tasks.requests.post") as mock_post:
        mock_post.return_value = make_response(200, {"received": True})
        result = deliver_webhook.apply(
            args=("https://webhook.example.com/hook",),
            kwargs={"payload": {"event": "order.created", "id": 1}},
        )
    assert result.successful()
    assert result.get() == {"status": 200, "body": {"received": True}}
    mock_post.assert_called_once()


# ── retry path ─────────────────────────────────────────────────────────────

def test_deliver_webhook_retries_on_timeout():
    """ConnectionError triggers autoretry; after max_retries, MaxRetriesExceededError."""
    with patch("myapp.tasks.requests.post",
               side_effect=requests.ConnectionError("refused")):
        result = deliver_webhook.apply(
            args=("https://down.example.com/hook",),
            kwargs={"payload": {}},
            throw=False,
        )
    assert result.failed()
    assert isinstance(result.result, MaxRetriesExceededError)


def test_deliver_webhook_succeeds_on_second_attempt():
    responses = [
        requests.ConnectionError("flaky"),
        make_response(200, {"ok": True}),
    ]
    with patch("myapp.tasks.requests.post", side_effect=responses):
        result = deliver_webhook.apply(
            args=("https://api.example.com/hook",),
            kwargs={"payload": {"event": "ping"}},
        )
    assert result.successful()
    assert result.get()["status"] == 200


# ── HTTP error path ─────────────────────────────────────────────────────────

def test_deliver_webhook_http_404_not_retried():
    """HTTP 4xx errors are NOT in autoretry_for — they fail immediately."""
    with patch("myapp.tasks.requests.post",
               return_value=make_response(404, {"error": "not found"})):
        result = deliver_webhook.apply(
            args=("https://api.example.com/missing",),
            kwargs={"payload": {}},
            throw=False,
        )
    assert result.failed()
    assert isinstance(result.result, requests.HTTPError)


# ── HMAC signature ─────────────────────────────────────────────────────────

def test_deliver_webhook_attaches_hmac_signature():
    with patch("myapp.tasks.requests.post") as mock_post:
        mock_post.return_value = make_response(200, {})
        deliver_webhook.apply(
            args=("https://secure.example.com/hook",),
            kwargs={"payload": {"id": 1}, "secret": "mysecret"},
        )
    _, call_kwargs = mock_post.call_args
    headers = call_kwargs["headers"]
    assert "X-Signature-256" in headers
    assert headers["X-Signature-256"].startswith("sha256=")


# ── lifecycle hooks ─────────────────────────────────────────────────────────

def test_on_failure_hook_updates_db(db):
    from myapp.models import WebhookDelivery

    delivery = WebhookDelivery.objects.create(
        task_id="test-task-id", url="https://fail.example.com", status="pending"
    )

    with patch("myapp.tasks.requests.post",
               side_effect=requests.ConnectionError("down")):
        result = deliver_webhook.apply(
            task_id="test-task-id",
            args=("https://fail.example.com",),
            kwargs={"payload": {}},
            throw=False,
        )

    delivery.refresh_from_db()
    assert delivery.status == "failed"
    assert "down" in delivery.error


# ── signals ────────────────────────────────────────────────────────────────

def test_task_success_signal_fires():
    success_events = []
    task_success.connect(
        lambda sender=None, result=None, **kw: success_events.append(result),
        sender=deliver_webhook,
        weak=False,
    )
    try:
        with patch("myapp.tasks.requests.post",
                   return_value=make_response(200, {"received": True})):
            deliver_webhook.apply(
                args=("https://ok.example.com",), kwargs={"payload": {}}
            )
        assert len(success_events) == 1
        assert success_events[0]["status"] == 200
    finally:
        task_success.disconnect(sender=deliver_webhook)


def test_task_retry_signal_fires_on_network_error():
    retry_reasons = []
    task_retry.connect(
        lambda sender=None, reason=None, **kw: retry_reasons.append(reason),
        sender=deliver_webhook,
        weak=False,
    )
    try:
        responses = [
            requests.ConnectionError("attempt 1"),
            requests.ConnectionError("attempt 2"),
            make_response(200, {}),
        ]
        with patch("myapp.tasks.requests.post", side_effect=responses):
            deliver_webhook.apply(
                args=("https://flaky.example.com",), kwargs={"payload": {}}
            )
        assert len(retry_reasons) == 2
        assert all(isinstance(r, requests.ConnectionError) for r in retry_reasons)
    finally:
        task_retry.disconnect(sender=deliver_webhook)
```

---

## Quick Reference

### Fixture Summary

| Fixture | Scope | Purpose |
|---|---|---|
| `celery_app` | function | Test Celery app instance |
| `celery_worker` | function | Embedded worker thread |
| `celery_config` | session | Override app configuration |
| `celery_worker_parameters` | session | `WorkController` init args |
| `celery_enable_logging` | session | Show worker log output |
| `celery_includes` | session | Task module auto-import |
| `celery_worker_pool` | session | Execution pool type |
| `celery_session_worker` | session | Persistent session-wide worker |
| `celery_session_app` | session | Persistent session-wide app |

### Task execution method comparison

| Method | Broker | Serialization | Retries | Hooks/Signals | Speed |
|---|---|---|---|---|---|
| Direct call `task(args)` | No | No | No | No | Fastest |
| `.apply()` | No | Yes | Yes | Yes | Fast |
| `.delay()` + `celery_worker` | Yes (in-memory) | Yes | Yes | Yes | Moderate |
| `.delay()` + real broker | Yes | Yes | Yes | Yes | Slowest |
