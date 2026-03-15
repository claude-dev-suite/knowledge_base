# pytest-django — Basics

**Official Documentation:** https://pytest-django.readthedocs.io/
**Version:** 4.10.0+ (compatible with Django 4.2+, pytest 7+)

---

## Table of Contents

1. [Installation and Configuration](#installation-and-configuration)
2. [Database Access Markers](#database-access-markers)
3. [Core Database Fixtures](#core-database-fixtures)
4. [HTTP Client Fixtures](#http-client-fixtures)
5. [User and Auth Fixtures](#user-and-auth-fixtures)
6. [Settings Fixture](#settings-fixture)
7. [Email Fixtures](#email-fixtures)
8. [Query Assertion Fixtures](#query-assertion-fixtures)
9. [On-Commit Callbacks Fixture](#on-commit-callbacks-fixture)
10. [Async Fixtures](#async-fixtures)
11. [Live Server Fixture](#live-server-fixture)
12. [Misc Configuration](#misc-configuration)
13. [pytest.ini / pyproject.toml Reference](#pytestini--pyprojecttoml-reference)
14. [Using Markers on Classes and Modules](#using-markers-on-classes-and-modules)
15. [Django TestCase Compatibility](#django-testcase-compatibility)
16. [DRF APIClient Integration](#drf-apiclient-integration)
17. [pytest_django.asserts Helpers](#pytest_djangoasserts-helpers)
18. [Recommended conftest.py](#recommended-conftestpy)

---

## Installation and Configuration

```bash
pip install pytest-django

# Or with DRF support
pip install pytest-django djangorestframework

# Common companion packages
pip install factory-boy faker pytest-xdist pytest-cov
```

### pyproject.toml (recommended)

```toml
[tool.pytest.ini_options]
DJANGO_SETTINGS_MODULE = "myproject.settings.test"
python_files   = ["tests.py", "test_*.py", "*_tests.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = "-v --tb=short --strict-markers"
# Optional: reuse test DB across runs (requires --create-db after schema changes)
# addopts = "-v --reuse-db"
```

### pytest.ini (alternative)

```ini
[pytest]
DJANGO_SETTINGS_MODULE = yourproject.settings.test
python_files = tests.py test_*.py *_tests.py
addopts = -v --tb=short
```

### tox.ini

```ini
[pytest]
DJANGO_SETTINGS_MODULE = yourproject.settings.test
```

### CLI overrides

```bash
# Override settings module
pytest --ds=myproject.settings.ci

# Force DB recreation (when used with --reuse-db in addopts)
pytest --create-db

# Disable Django migrations (faster; creates tables from models directly)
pytest --no-migrations

# Set DEBUG=True for this run
pytest --django-debug-mode=true
```

### Environment variable

```bash
export DJANGO_SETTINGS_MODULE=myproject.settings.test
pytest
```

### Precedence order (highest to lowest)

1. `--ds` command-line flag
2. `DJANGO_SETTINGS_MODULE` environment variable
3. `DJANGO_SETTINGS_MODULE` in `pytest.ini` / `pyproject.toml`

### Recommended test settings file (`settings/test.py`)

```python
from .base import *

# Fast in-memory database for unit tests
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": ":memory:",
    }
}

# Use a fast, insecure password hasher to speed up create_user() calls
PASSWORD_HASHERS = ["django.contrib.auth.hashers.MD5PasswordHasher"]

# Capture emails in memory instead of sending
EMAIL_BACKEND = "django.core.mail.backends.locmem.EmailBackend"

# Disable real caching
CACHES = {"default": {"BACKEND": "django.core.cache.backends.dummy.DummyCache"}}

# Silence Celery (run tasks synchronously if you need them to run at all)
CELERY_TASK_ALWAYS_EAGER = True
CELERY_TASK_EAGER_PROPAGATES = True

# Avoid hitting external storage
DEFAULT_FILE_STORAGE = "django.core.files.storage.FileSystemStorage"
MEDIA_ROOT = "/tmp/django_test_media"

# Disable debug toolbar and other dev-only middleware
DEBUG = False
```

### Python path configuration

By default pytest-django auto-discovers `manage.py` to set the Python path. Disable
this if you have multiple `manage.py` files or a `src/` layout:

```ini
[pytest]
django_find_project = false
pythonpath = . src
```

Or install your package in editable mode (preferred):

```bash
pip install --editable .
```

### Programmatic settings in conftest.py (no settings module)

```python
# conftest.py
import django
from django.conf import settings


def pytest_configure():
    settings.configure(
        DATABASES={"default": {"ENGINE": "django.db.backends.sqlite3", "NAME": ":memory:"}},
        INSTALLED_APPS=[
            "django.contrib.contenttypes",
            "django.contrib.auth",
            "myapp",
        ],
        ROOT_URLCONF="myapp.urls",
        SECRET_KEY="test-secret-key-not-for-production",
        DEFAULT_AUTO_FIELD="django.db.models.BigAutoField",
    )
```

---

## Database Access Markers

### Default behavior: no DB access

pytest-django blocks all database access by default. Any test that touches the DB
without permission raises a `django.test.utils.DatabaseBlocker` exception. This is
intentional — it forces you to be explicit and keeps unit tests fast.

### `@pytest.mark.django_db` — option reference

```python
import pytest

# ── Basic ─────────────────────────────────────────────────────────────────────
# Wraps each test in a savepoint that is rolled back after the test.
# Equivalent to Django's TestCase. Fastest option.
@pytest.mark.django_db
def test_create_item():
    from myapp.models import Item
    item = Item.objects.create(name="Widget")
    assert item.pk is not None
    assert Item.objects.count() == 1  # rolled back after this test


# ── transaction=True ─────────────────────────────────────────────────────────
# Uses a real transaction (no wrapping savepoint).
# Equivalent to Django's TransactionTestCase.
# Required when testing code that calls transaction.on_commit(),
# select_for_update(), LISTEN/NOTIFY, signals fired after commit, etc.
# Slower: flushes all tables between tests.
@pytest.mark.django_db(transaction=True)
def test_on_commit_fired():
    from django.db import transaction
    results = []

    def callback():
        results.append("fired")

    with transaction.atomic():
        transaction.on_commit(callback)

    assert results == ["fired"]


# ── reset_sequences=True ─────────────────────────────────────────────────────
# Resets auto-increment sequences (PostgreSQL, Oracle, etc.) to 1 before the test.
# Requires transaction=True.
# Useful when tests assert specific PK values.
@pytest.mark.django_db(transaction=True, reset_sequences=True)
def test_pk_starts_at_one():
    from myapp.models import Item
    item = Item.objects.create(name="First")
    assert item.pk == 1


# ── databases ─────────────────────────────────────────────────────────────────
# Opt-in to additional databases. Default: only 'default'.
@pytest.mark.django_db(databases=["default", "analytics"])
def test_cross_db_read():
    from myapp.models import Event
    assert Event.objects.using("analytics").count() >= 0


# Use the "__all__" shortcut to access every configured database.
@pytest.mark.django_db(databases="__all__")
def test_all_databases():
    from myapp.models import Item
    assert Item.objects.count() == 0


# ── serialized_rollback=True ──────────────────────────────────────────────────
# After the test DB is flushed (transaction=True), Django can reload fixtures
# that were serialized at startup. Needed when tests depend on data loaded
# via data migrations or initial_data fixtures.
# ~3x slower than plain transaction=True.
@pytest.mark.django_db(transaction=True, serialized_rollback=True)
def test_initial_data_present():
    from myapp.models import Category
    assert Category.objects.filter(slug="uncategorized").exists()
```

### Transaction behavior comparison

| Marker option            | Django equivalent      | Isolation mechanism  | Speed   | Tests `commit()` |
|--------------------------|------------------------|----------------------|---------|------------------|
| `(default)`              | `TestCase`             | SAVEPOINT rollback   | fastest | No               |
| `transaction=True`       | `TransactionTestCase`  | Table flush (TRUNCATE)| slower  | Yes              |
| `transaction=True, reset_sequences=True` | `TransactionTestCase` + `reset_sequences=True` | Table flush | slowest | Yes |
| `serialized_rollback=True` | `TransactionTestCase` + serialized rollback | Table flush + reload | ~3x slower | Yes |

### Module and class level markers

```python
# Apply to every test in the module
pytestmark = pytest.mark.django_db

# Apply to every test in a class
class TestOrders:
    pytestmark = pytest.mark.django_db(transaction=True)

    def test_create_order(self):
        ...

    def test_cancel_order(self):
        ...
```

---

## Core Database Fixtures

These fixtures grant database access to other fixtures (not test functions — use the
marker for test functions).

### `db` — non-transactional access (savepoint rollback)

```python
import pytest


@pytest.fixture
def article(db):
    """Create a test article. DB access rolls back after each test."""
    from myapp.models import Article
    return Article.objects.create(
        title="Test Article",
        body="Body text.",
        published=True,
    )


@pytest.fixture
def user(db):
    from django.contrib.auth import get_user_model
    User = get_user_model()
    return User.objects.create_user(
        username="alice",
        email="alice@example.com",
        password="testpass123",
    )
```

### `transactional_db` — real transaction access

```python
@pytest.fixture
def payment(transactional_db):
    """Fixture for tests that assert transaction.on_commit() behavior."""
    from myapp.models import Payment
    return Payment.objects.create(amount=100, status="pending")
```

### `django_db_reset_sequences`

Grants transactional DB access AND resets sequences. Use in fixtures that create
objects whose PK value matters to the test.

```python
@pytest.fixture
def first_item(django_db_reset_sequences):
    from myapp.models import Item
    return Item.objects.create(name="First item")  # guaranteed pk=1


def test_item_pk_is_one(first_item):
    assert first_item.pk == 1
```

### `django_db_serialized_rollback`

Grants transactional DB access AND re-serializes data after flush.

```python
@pytest.fixture
def with_initial_data(django_db_serialized_rollback):
    """Test sees data loaded by data migrations."""
    pass


def test_category_exists(with_initial_data):
    from myapp.models import Category
    assert Category.objects.filter(name="General").exists()
```

### `django_db_blocker` — manual control

```python
@pytest.fixture(scope="session")
def load_fixtures(django_db_setup, django_db_blocker):
    """Load fixtures once for the whole test session."""
    from django.core.management import call_command
    with django_db_blocker.unblock():
        call_command("loaddata", "test_data.json")
```

---

## HTTP Client Fixtures

### `client` — Django test client

`django.test.Client` instance, unauthenticated. Does NOT automatically get `db` access
— the test must use `@pytest.mark.django_db` or accept a db-granting fixture.

```python
@pytest.mark.django_db
def test_homepage(client):
    response = client.get("/")
    assert response.status_code == 200


@pytest.mark.django_db
def test_json_post(client):
    import json
    response = client.post(
        "/api/items/",
        data=json.dumps({"name": "Widget", "price": "9.99"}),
        content_type="application/json",
    )
    assert response.status_code == 201
    assert response.json()["name"] == "Widget"


@pytest.mark.django_db
def test_redirect_after_login(client, django_user_model):
    django_user_model.objects.create_user("alice", password="pass")
    response = client.post("/accounts/login/", {"username": "alice", "password": "pass"})
    assert response.status_code == 302


@pytest.mark.django_db
def test_force_login(client, django_user_model):
    user = django_user_model.objects.create_user("alice", password="pass")
    client.force_login(user)   # bypasses credential checking
    response = client.get("/dashboard/")
    assert response.status_code == 200


@pytest.mark.django_db
def test_session_manipulation(client, django_user_model):
    user = django_user_model.objects.create_user("alice", password="pass")
    client.force_login(user)
    session = client.session
    session["cart"] = [1, 2, 3]
    session.save()
    response = client.get("/cart/")
    assert response.status_code == 200
```

### `admin_client` — pre-authenticated superuser client

`admin_client` automatically enables DB access (you do not need `@pytest.mark.django_db`
separately when only using `admin_client`).

```python
def test_admin_index(admin_client):
    response = admin_client.get("/admin/")
    assert response.status_code == 200


def test_admin_changelist(admin_client):
    response = admin_client.get("/admin/myapp/article/")
    assert response.status_code == 200


def test_admin_can_create(admin_client):
    response = admin_client.post(
        "/admin/myapp/article/add/",
        {
            "title": "New Article",
            "body": "Content",
            "published": True,
            "_save": "Save",
        },
    )
    # Admin redirects to changelist on success
    assert response.status_code == 302
```

### `rf` — RequestFactory

Creates request objects directly without Django's URL routing or middleware.
Fast for view unit tests where you don't need middleware processing.

```python
from myapp.views import ArticleListView, ArticleDetailView


def test_list_view_renders(rf, admin_user):
    request = rf.get("/articles/")
    request.user = admin_user
    response = ArticleListView.as_view()(request)
    assert response.status_code == 200


def test_detail_view_not_found(rf, user, db):
    request = rf.get("/articles/999/")
    request.user = user
    response = ArticleDetailView.as_view()(request, pk=999)
    assert response.status_code == 404


def test_post_creates_object(rf, user, db):
    import json
    request = rf.post(
        "/articles/",
        data=json.dumps({"title": "Test", "body": "Body"}),
        content_type="application/json",
    )
    request.user = user
    response = ArticleDetailView.as_view()(request)
    assert response.status_code == 201
```

---

## User and Auth Fixtures

### `django_user_model`

A shortcut to the User model configured by `AUTH_USER_MODEL`. Works with custom user
models automatically.

```python
@pytest.mark.django_db
def test_create_user(django_user_model):
    user = django_user_model.objects.create_user(
        username="bob",
        email="bob@example.com",
        password="hunter2",
    )
    assert user.pk is not None
    assert user.check_password("hunter2")
    assert not user.is_staff
    assert not user.is_superuser


@pytest.mark.django_db
def test_create_superuser(django_user_model):
    admin = django_user_model.objects.create_superuser(
        username="godmode",
        email="god@example.com",
        password="omnipotent",
    )
    assert admin.is_superuser
    assert admin.is_staff


# Common pattern: reusable user fixture
@pytest.fixture
def regular_user(db, django_user_model):
    return django_user_model.objects.create_user(
        username="regular",
        email="regular@example.com",
        password="regularpass",
    )
```

### `django_username_field`

Returns the name of the field used as the username (e.g., `"username"` for the default
User model, or `"email"` for custom user models that use email as username).

```python
def test_username_field_name(django_username_field):
    # For default User model
    assert django_username_field == "username"
    # For custom email-based User model it would be "email"


@pytest.fixture
def user_by_field(db, django_user_model, django_username_field):
    """Create a user regardless of the username field name."""
    return django_user_model.objects.create_user(
        **{django_username_field: "testuser@example.com"},
        password="testpass",
    )
```

### `admin_user`

A pre-created superuser with `username="admin"`, `password="password"`. Automatically
enables DB access.

```python
def test_superuser_attributes(admin_user):
    assert admin_user.is_superuser is True
    assert admin_user.is_staff is True
    assert admin_user.username == "admin"


def test_admin_view_with_rf(rf, admin_user):
    from myapp.views import AdminOnlyView
    request = rf.get("/admin-only/")
    request.user = admin_user
    response = AdminOnlyView.as_view()(request)
    assert response.status_code == 200
```

---

## Settings Fixture

The `settings` fixture provides a handle to Django's settings object. Any attribute
you change is automatically reverted after the test ends.

```python
def test_feature_flag_off(client, settings):
    settings.ENABLE_NEW_CHECKOUT = False
    response = client.get("/checkout/")
    assert response.status_code == 404  # feature gated


def test_feature_flag_on(client, settings):
    settings.ENABLE_NEW_CHECKOUT = True
    response = client.get("/checkout/")
    assert response.status_code == 200


def test_custom_email_backend(settings, mailoutbox):
    # mailoutbox already sets locmem backend, but you can also set it here
    settings.EMAIL_BACKEND = "django.core.mail.backends.locmem.EmailBackend"
    from django.core.mail import send_mail
    send_mail("Hi", "Body", "from@example.com", ["to@example.com"])
    assert len(mailoutbox) == 1


def test_rate_limit_threshold(settings):
    settings.API_RATE_LIMIT = 5
    # ... exercise rate-limiting code
    pass


def test_media_root(settings, tmp_path):
    settings.MEDIA_ROOT = str(tmp_path / "media")
    settings.DEFAULT_FILE_STORAGE = "django.core.files.storage.FileSystemStorage"
    # ... test file upload handling


# autouse pattern: apply settings override to every test in the module
@pytest.fixture(autouse=True)
def disable_throttling(settings):
    """Remove all DRF throttle classes during tests."""
    settings.REST_FRAMEWORK = {
        **getattr(settings, "REST_FRAMEWORK", {}),
        "DEFAULT_THROTTLE_CLASSES": [],
        "DEFAULT_THROTTLE_RATES": {},
    }
```

### `django_settings` fixture

An alias for `settings`. Both refer to the same fixture object.

---

## Email Fixtures

### `mailoutbox`

`mailoutbox` provides the list `django.core.mail.outbox`. It automatically sets
`EMAIL_BACKEND` to `locmem` and clears the outbox before each test.

```python
from django.core import mail


def test_send_welcome_email(mailoutbox, django_user_model, db):
    user = django_user_model.objects.create_user(
        username="new_user",
        email="new@example.com",
        password="pass",
    )
    # Assume a signal or service sends a welcome email on create
    assert len(mailoutbox) == 1
    msg = mailoutbox[0]
    assert msg.subject == "Welcome to Our Service"
    assert "new@example.com" in msg.to
    assert "welcome" in msg.body.lower()


def test_html_email(mailoutbox):
    mail.send_mail(
        subject="Newsletter",
        message="Plain text version",
        from_email="news@example.com",
        recipient_list=["subscriber@example.com"],
        html_message="<h1>Newsletter</h1><p>HTML version</p>",
    )
    assert len(mailoutbox) == 1
    msg = mailoutbox[0]
    assert msg.alternatives  # has HTML alternative
    html_body, mime = msg.alternatives[0]
    assert "<h1>Newsletter" in html_body
    assert mime == "text/html"


def test_multiple_recipients(mailoutbox):
    mail.send_mail(
        subject="Announcement",
        message="Body",
        from_email="admin@example.com",
        recipient_list=["a@example.com", "b@example.com", "c@example.com"],
    )
    assert len(mailoutbox) == 1
    assert len(mailoutbox[0].to) == 3


def test_email_reply_to(mailoutbox):
    msg = mail.EmailMessage(
        subject="Reply to this",
        body="Body",
        from_email="noreply@example.com",
        to=["user@example.com"],
        reply_to=["support@example.com"],
    )
    msg.send()
    assert mailoutbox[0].reply_to == ["support@example.com"]


def test_no_emails_sent_on_dry_run(mailoutbox, client, settings):
    settings.SEND_EMAILS = False
    client.get("/api/trigger-notification/")
    assert len(mailoutbox) == 0  # dry run suppresses sending


def test_outbox_cleared_between_tests(mailoutbox):
    # This test always starts with an empty outbox
    assert len(mailoutbox) == 0
```

### `django_mail_backend`

The underlying fixture that configures `locmem`. Rarely used directly; prefer
`mailoutbox`.

---

## Query Assertion Fixtures

### `django_assert_num_queries`

Asserts that exactly N SQL queries are executed inside the `with` block.

```python
def test_list_view_query_count(client, django_assert_num_queries, db):
    from myapp.models import Article
    Article.objects.bulk_create([
        Article(title=f"Article {i}", body="body", published=True)
        for i in range(5)
    ])
    with django_assert_num_queries(2):
        # Expect: 1 auth/session query + 1 articles query (with select_related)
        response = client.get("/api/articles/")
    assert response.status_code == 200


def test_single_query(db, django_assert_num_queries):
    from django.contrib.auth import get_user_model
    User = get_user_model()
    User.objects.create_user("alice", password="pass")

    with django_assert_num_queries(1):
        User.objects.get(username="alice")


def test_n_plus_one_detected(db, django_assert_num_queries):
    from myapp.models import Author, Book
    author = Author.objects.create(name="Author")
    Book.objects.bulk_create([Book(title=f"Book {i}", author=author) for i in range(3)])

    with django_assert_num_queries(1):
        # This uses select_related, so only 1 JOIN query
        books = list(Book.objects.select_related("author"))
        for book in books:
            _ = book.author.name  # no extra queries


# Use connection= to target a non-default database
def test_query_on_secondary_db(db, django_assert_num_queries):
    from django.db import connections
    conn = connections["analytics"]
    with django_assert_num_queries(1, connection=conn):
        # ... query analytics db
        pass
```

### `django_assert_max_num_queries`

Like `django_assert_num_queries` but asserts N or fewer queries (useful when the
exact count is variable but you want to prevent unbounded growth).

```python
def test_dashboard_bounded_queries(client, django_assert_max_num_queries, db):
    with django_assert_max_num_queries(15):
        response = client.get("/dashboard/")
    assert response.status_code == 200


def test_paginated_endpoint_efficiency(auth_api_client, django_assert_max_num_queries, db):
    # Should not exceed 5 queries regardless of page size
    with django_assert_max_num_queries(5):
        response = auth_api_client.get("/api/articles/?page=1&page_size=50")
    assert response.status_code == 200
```

---

## On-Commit Callbacks Fixture

`django_capture_on_commit_callbacks` captures `transaction.on_commit()` callbacks
that would otherwise only fire after a real commit (which doesn't happen in default
`TestCase`-style tests).

The `execute=True` parameter runs the captured callbacks immediately, letting you
assert their side effects in the same test.

```python
def test_order_sends_confirmation_email(
    client, db, mailoutbox, django_capture_on_commit_callbacks
):
    with django_capture_on_commit_callbacks(execute=True) as callbacks:
        response = client.post(
            "/orders/",
            data='{"product_id": 1, "quantity": 2}',
            content_type="application/json",
        )
    assert response.status_code == 201
    assert len(callbacks) == 1          # one on_commit callback registered
    assert len(mailoutbox) == 1         # the callback fired and sent an email
    assert mailoutbox[0].subject == "Order Confirmation"


def test_callback_registered_but_not_executed(
    db, django_capture_on_commit_callbacks
):
    """Capture callbacks without executing them."""
    from django.db import transaction

    with django_capture_on_commit_callbacks(execute=False) as callbacks:
        with transaction.atomic():
            transaction.on_commit(lambda: None)

    assert len(callbacks) == 1   # callback was registered
    # but not executed


def test_celery_task_dispatched_on_commit(
    db, django_capture_on_commit_callbacks
):
    from unittest.mock import patch

    with patch("myapp.tasks.send_notification.delay") as mock_task:
        with django_capture_on_commit_callbacks(execute=True):
            # Code under test calls transaction.on_commit(send_notification.delay)
            from myapp.services import create_order
            create_order(user_id=1, product_id=2)

    mock_task.assert_called_once()
```

---

## Async Fixtures

### `async_client`

`django.test.AsyncClient` wrapped as a pytest fixture. Requires `pytest-asyncio` or
Django's built-in async test support.

```python
import pytest


@pytest.mark.django_db(transaction=True)
@pytest.mark.asyncio
async def test_async_view(async_client):
    response = await async_client.get("/async-endpoint/")
    assert response.status_code == 200


@pytest.mark.django_db(transaction=True)
@pytest.mark.asyncio
async def test_async_post(async_client):
    import json
    response = await async_client.post(
        "/async-items/",
        data=json.dumps({"name": "Widget"}),
        content_type="application/json",
    )
    assert response.status_code == 201
```

### `async_rf`

Async variant of `RequestFactory`.

```python
import pytest


@pytest.mark.asyncio
async def test_async_view_unit(async_rf, db):
    from myapp.views import AsyncItemView

    request = async_rf.get("/items/")
    response = await AsyncItemView.as_view()(request)
    assert response.status_code == 200
```

---

## Live Server Fixture

`live_server` runs a real Django server in a background thread. Required for
Selenium, Playwright, or any browser-based integration tests.

```python
import pytest


def test_homepage_title(live_server):
    import urllib.request
    with urllib.request.urlopen(live_server.url + "/") as resp:
        content = resp.read().decode()
    assert "My App" in content


def test_with_requests_library(live_server):
    import requests
    response = requests.get(f"{live_server.url}/api/health/")
    assert response.status_code == 200
    assert response.json()["status"] == "ok"


# live_server.url returns e.g. "http://localhost:52187"
def test_api_base_url(live_server):
    assert live_server.url.startswith("http://localhost:")
```

---

## Misc Configuration

### `--reuse-db` and `--create-db`

```bash
# Preserve the test database between runs (fast for large schemas)
pytest --reuse-db

# Force recreation (run after schema changes)
pytest --reuse-db --create-db
```

Add to `pytest.ini` permanently:

```ini
[pytest]
addopts = --reuse-db
```

### `--no-migrations`

Skip running Django migrations; create tables directly from model definitions.
Significantly faster for large projects with many migrations.

```bash
pytest --no-migrations
# Alias:
pytest --nomigrations
```

### `django_db_setup` session fixture

Override to customize the test database creation/teardown lifecycle:

```python
# conftest.py
import pytest


@pytest.fixture(scope="session")
def django_db_setup():
    """Use an existing external database instead of creating a fresh one."""
    from django.conf import settings
    settings.DATABASES["default"] = {
        "ENGINE": "django.db.backends.postgresql",
        "HOST": "db.example.com",
        "NAME": "staging_readonly_db",
        "USER": "test_user",
        "PASSWORD": "test_pass",
    }
```

### Parallel execution with pytest-xdist

```bash
pip install pytest-xdist
pytest -n 4  # 4 worker processes, each gets its own test database
```

pytest-django automatically suffixes the database name per worker:
`test_mydb_gw0`, `test_mydb_gw1`, etc.

---

## pytest.ini / pyproject.toml Reference

| Option | Values | Default | Description |
|--------|--------|---------|-------------|
| `DJANGO_SETTINGS_MODULE` | module path | — | Settings module to use |
| `django_find_project` | `true`/`false` | `true` | Auto-discover `manage.py` and add its directory to sys.path |
| `django_debug_mode` | `true`/`false`/`keep` | `false` | Set `DEBUG` during tests. `keep` preserves your setting. |
| `FAIL_INVALID_TEMPLATE_VARS` | `true`/`false` | `false` | Fail on undefined template variables (same as `--fail-on-template-vars`) |

```toml
# pyproject.toml — complete reference
[tool.pytest.ini_options]
DJANGO_SETTINGS_MODULE = "myproject.settings.test"
django_find_project    = true
django_debug_mode      = "false"
FAIL_INVALID_TEMPLATE_VARS = false
addopts = "--reuse-db -v --tb=short"
```

---

## Using Markers on Classes and Modules

### Class-level marker

```python
import pytest


@pytest.mark.django_db
class TestArticleAPI:
    def test_create(self, client):
        response = client.post("/api/articles/", {"title": "T", "body": "B"})
        assert response.status_code == 201

    def test_list(self, client):
        response = client.get("/api/articles/")
        assert response.status_code == 200

    def test_delete_requires_auth(self, client):
        response = client.delete("/api/articles/1/")
        assert response.status_code in (401, 403)


@pytest.mark.django_db(transaction=True)
class TestPaymentFlow:
    def test_payment_commit(self):
        from myapp.models import Payment
        from django.db import transaction

        completed = []
        transaction.on_commit(lambda: completed.append(True))
        assert completed == [True]
```

### Module-level marker

```python
# At the top of the test file — applies to ALL tests in the file
pytestmark = pytest.mark.django_db

def test_one():
    from myapp.models import Item
    assert Item.objects.count() == 0

def test_two():
    from myapp.models import Category
    assert Category.objects.exists() is False
```

---

## Django TestCase Compatibility

pytest-django is fully compatible with `django.test.TestCase` and its subclasses.
You can mix pytest-style functions with class-based TestCase in the same suite.

```python
from django.test import TestCase, TransactionTestCase


class ArticleTestCase(TestCase):
    """Uses Django's TestCase — still discovered and run by pytest."""

    def setUp(self):
        from myapp.models import Article
        self.article = Article.objects.create(title="Hello", body="World")

    def test_article_str(self):
        self.assertEqual(str(self.article), "Hello")

    def test_article_published(self):
        self.assertFalse(self.article.published)


class OrderTransactionTestCase(TransactionTestCase):
    def test_order_commit(self):
        from django.db import transaction
        fired = []
        with transaction.atomic():
            transaction.on_commit(lambda: fired.append(True))
        self.assertEqual(fired, [True])
```

### Accessing pytest fixtures from TestCase

Use `self.client`, `self.settings()`, etc. from Django's TestCase directly.
To use pytest fixtures inside a TestCase, use `@pytest.mark.usefixtures`:

```python
import pytest
from django.test import TestCase


@pytest.mark.usefixtures("settings")
class TestWithSettings(TestCase):
    def test_debug_off(self):
        from django.conf import settings
        self.assertFalse(settings.DEBUG)
```

---

## DRF APIClient Integration

```python
import pytest
from rest_framework.test import APIClient, APIRequestFactory
from rest_framework import status


# ── Fixtures ──────────────────────────────────────────────────────────────────

@pytest.fixture
def api_client():
    return APIClient()


@pytest.fixture
def user(db, django_user_model):
    return django_user_model.objects.create_user(
        username="testuser",
        email="test@example.com",
        password="testpass123",
    )


@pytest.fixture
def auth_client(api_client, user):
    """Authenticated DRF client (bypasses authentication backends)."""
    api_client.force_authenticate(user=user)
    return api_client


@pytest.fixture
def admin_api_client(api_client, django_user_model, db):
    admin = django_user_model.objects.create_superuser(
        username="admin", email="admin@example.com", password="adminpass"
    )
    api_client.force_authenticate(user=admin)
    return api_client


# ── Tests ─────────────────────────────────────────────────────────────────────

@pytest.mark.django_db
def test_list_endpoint_unauthenticated(api_client):
    response = api_client.get("/api/v1/articles/")
    assert response.status_code == status.HTTP_200_OK  # public list


@pytest.mark.django_db
def test_create_requires_auth(api_client):
    response = api_client.post("/api/v1/articles/", {"title": "T", "body": "B"})
    assert response.status_code == status.HTTP_401_UNAUTHORIZED


@pytest.mark.django_db
def test_create_article(auth_client):
    response = auth_client.post(
        "/api/v1/articles/",
        {"title": "My Article", "body": "Content here"},
        format="json",
    )
    assert response.status_code == status.HTTP_201_CREATED
    assert response.data["title"] == "My Article"
    assert "id" in response.data


@pytest.mark.django_db
def test_token_authentication(api_client, user):
    from rest_framework.authtoken.models import Token
    token = Token.objects.create(user=user)
    api_client.credentials(HTTP_AUTHORIZATION=f"Token {token.key}")
    response = api_client.get("/api/v1/me/")
    assert response.status_code == status.HTTP_200_OK


@pytest.mark.django_db
def test_jwt_authentication(api_client, user):
    # Uses djangorestframework-simplejwt
    resp = api_client.post(
        "/api/token/",
        {"username": "testuser", "password": "testpass123"},
        format="json",
    )
    assert resp.status_code == status.HTTP_200_OK
    token = resp.data["access"]
    api_client.credentials(HTTP_AUTHORIZATION=f"Bearer {token}")
    resp2 = api_client.get("/api/v1/me/")
    assert resp2.status_code == status.HTTP_200_OK


@pytest.mark.django_db
def test_api_request_factory(db, user):
    from myapp.views import ArticleViewSet
    factory = APIRequestFactory()
    request = factory.get("/api/v1/articles/")
    force_authenticate(request, user=user)
    view = ArticleViewSet.as_view({"get": "list"})
    response = view(request)
    assert response.status_code == status.HTTP_200_OK


@pytest.mark.django_db
def test_pagination(auth_client, db):
    from myapp.models import Article
    Article.objects.bulk_create([
        Article(title=f"Article {i}", body="body") for i in range(15)
    ])
    response = auth_client.get("/api/v1/articles/?page=1")
    assert response.status_code == status.HTTP_200_OK
    assert "results" in response.data
    assert "count" in response.data
    assert response.data["count"] == 15
```

---

## pytest_django.asserts Helpers

Import Django's assertion methods as standalone functions:

```python
from pytest_django.asserts import (
    assertContains,
    assertNotContains,
    assertRedirects,
    assertTemplateUsed,
    assertTemplateNotUsed,
    assertFormError,
    assertQuerySetEqual,
    assertNumQueries,
    assertRaisesMessage,
    assertWarnsMessage,
    assertFieldOutput,
    assertHTMLEqual,
    assertJSONEqual,
    assertURLEqual,
    assertInHTML,
)


@pytest.mark.django_db
def test_template_assertions(client):
    response = client.get("/about/")
    assertTemplateUsed(response, "about.html")
    assertNotContains(response, "Error")
    assertContains(response, "About Us", status_code=200)


@pytest.mark.django_db
def test_redirect_assertions(client, django_user_model):
    django_user_model.objects.create_user("alice", password="pass")
    response = client.post("/login/", {"username": "alice", "password": "pass"})
    assertRedirects(response, "/dashboard/", status_code=302)


@pytest.mark.django_db
def test_form_error_assertion(client):
    response = client.post("/signup/", {"username": "", "password": "short"})
    assertFormError(response.context["form"], "username", "This field is required.")


@pytest.mark.django_db
def test_json_response(client):
    response = client.get("/api/config/")
    assertJSONEqual(
        response.content,
        {"version": "1.0", "debug": False},
    )
```

---

## Recommended conftest.py

A production-ready `conftest.py` combining all the patterns above:

```python
# conftest.py
import pytest


# ── Performance optimizations (autouse = applied to every test) ───────────────

@pytest.fixture(autouse=True)
def fast_password_hasher(settings):
    """Replace bcrypt with MD5 — creates users ~100x faster in tests."""
    settings.PASSWORD_HASHERS = ["django.contrib.auth.hashers.MD5PasswordHasher"]


@pytest.fixture(autouse=True)
def no_real_cache(settings):
    """Prevent tests from reading/writing a shared cache."""
    settings.CACHES = {
        "default": {"BACKEND": "django.core.cache.backends.dummy.DummyCache"}
    }


@pytest.fixture(autouse=True)
def locmem_email(settings):
    """Capture emails in memory; do not send them."""
    settings.EMAIL_BACKEND = "django.core.mail.backends.locmem.EmailBackend"


# ── User fixtures ─────────────────────────────────────────────────────────────

@pytest.fixture
def user(db, django_user_model):
    return django_user_model.objects.create_user(
        username="testuser",
        email="testuser@example.com",
        password="testpass123",
    )


@pytest.fixture
def superuser(db, django_user_model):
    return django_user_model.objects.create_superuser(
        username="admin",
        email="admin@example.com",
        password="adminpass123",
    )


# ── Django test client fixtures ───────────────────────────────────────────────

@pytest.fixture
def auth_client(client, user):
    """Django test client authenticated as a regular user."""
    client.force_login(user)
    return client


@pytest.fixture
def admin_web_client(client, superuser):
    """Django test client authenticated as superuser."""
    client.force_login(superuser)
    return client


# ── DRF APIClient fixtures ────────────────────────────────────────────────────

@pytest.fixture
def api_client():
    from rest_framework.test import APIClient
    return APIClient()


@pytest.fixture
def auth_api_client(api_client, user):
    """DRF APIClient authenticated as a regular user."""
    api_client.force_authenticate(user=user)
    return api_client


@pytest.fixture
def admin_api_client(api_client, superuser):
    """DRF APIClient authenticated as superuser."""
    api_client.force_authenticate(user=superuser)
    return api_client


# ── Optional: reusable model fixtures ────────────────────────────────────────

@pytest.fixture
def article(db):
    from myapp.models import Article
    return Article.objects.create(
        title="Test Article",
        body="Test body content.",
        published=True,
    )
```
