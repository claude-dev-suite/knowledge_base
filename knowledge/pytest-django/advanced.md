# pytest-django — Advanced Patterns

**Official Documentation:** https://pytest-django.readthedocs.io/
**Version:** 4.10.0+ (Django 4.2+, pytest 7+)

---

## Table of Contents

1. [DRF Deep Dive](#drf-deep-dive)
2. [factory_boy Integration](#factory_boy-integration)
3. [Django Signals Testing](#django-signals-testing)
4. [Management Commands Testing](#management-commands-testing)
5. [Async Django Views Testing](#async-django-views-testing)
6. [Query Count Testing and Optimization](#query-count-testing-and-optimization)
7. [on_commit Callbacks — Advanced Patterns](#on_commit-callbacks--advanced-patterns)
8. [Multi-Database Testing](#multi-database-testing)
9. [Custom Test Database Setup](#custom-test-database-setup)
10. [Testcontainers with pytest-django](#testcontainers-with-pytest-django)
11. [Celery Task Testing](#celery-task-testing)
12. [Static Files and Media Files](#static-files-and-media-files)
13. [Django Channels WebSocket Testing](#django-channels-websocket-testing)
14. [Complete Real-World conftest.py](#complete-real-world-conftestpy)

---

## DRF Deep Dive

### APIClient — all authentication strategies

```python
import pytest
from rest_framework.test import APIClient, APIRequestFactory
from rest_framework import status
from rest_framework.authtoken.models import Token


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


# ── Strategy 1: force_authenticate (no real auth, fastest) ───────────────────

@pytest.mark.django_db
def test_force_authenticate(api_client, user):
    api_client.force_authenticate(user=user)
    response = api_client.get("/api/v1/profile/")
    assert response.status_code == status.HTTP_200_OK


@pytest.mark.django_db
def test_force_authenticate_with_token(api_client, user):
    """Simulate token auth at the transport layer without HTTP overhead."""
    token = Token.objects.create(user=user)
    api_client.force_authenticate(user=user, token=token)
    response = api_client.get("/api/v1/profile/")
    assert response.status_code == status.HTTP_200_OK


@pytest.mark.django_db
def test_unauthenticate(api_client, user):
    api_client.force_authenticate(user=user)
    api_client.force_authenticate(user=None)  # remove authentication
    response = api_client.get("/api/v1/profile/")
    assert response.status_code == status.HTTP_401_UNAUTHORIZED


# ── Strategy 2: Token authentication (real header) ───────────────────────────

@pytest.mark.django_db
def test_drf_token_auth(api_client, user):
    token = Token.objects.create(user=user)
    api_client.credentials(HTTP_AUTHORIZATION=f"Token {token.key}")
    response = api_client.get("/api/v1/profile/")
    assert response.status_code == status.HTTP_200_OK


@pytest.mark.django_db
def test_clear_credentials(api_client, user):
    token = Token.objects.create(user=user)
    api_client.credentials(HTTP_AUTHORIZATION=f"Token {token.key}")
    api_client.credentials()  # clear — subsequent requests are unauthenticated
    response = api_client.get("/api/v1/profile/")
    assert response.status_code == status.HTTP_401_UNAUTHORIZED


# ── Strategy 3: Session login ─────────────────────────────────────────────────

@pytest.mark.django_db
def test_session_login(api_client, user):
    logged_in = api_client.login(username="testuser", password="testpass123")
    assert logged_in is True
    response = api_client.get("/api/v1/profile/")
    assert response.status_code == status.HTTP_200_OK
    api_client.logout()


# ── Strategy 4: JWT (djangorestframework-simplejwt) ──────────────────────────

@pytest.mark.django_db
def test_jwt_obtain_and_use(api_client, user):
    # Obtain JWT pair
    resp = api_client.post(
        "/api/token/",
        {"username": "testuser", "password": "testpass123"},
        format="json",
    )
    assert resp.status_code == status.HTTP_200_OK
    access = resp.data["access"]
    refresh = resp.data["refresh"]

    # Use access token
    api_client.credentials(HTTP_AUTHORIZATION=f"Bearer {access}")
    profile_resp = api_client.get("/api/v1/profile/")
    assert profile_resp.status_code == status.HTTP_200_OK


@pytest.mark.django_db
def test_jwt_refresh(api_client, user):
    obtain = api_client.post(
        "/api/token/",
        {"username": "testuser", "password": "testpass123"},
        format="json",
    )
    refresh_token = obtain.data["refresh"]

    # Refresh the access token
    resp = api_client.post(
        "/api/token/refresh/",
        {"refresh": refresh_token},
        format="json",
    )
    assert resp.status_code == status.HTTP_200_OK
    assert "access" in resp.data


@pytest.mark.django_db
def test_jwt_expired_token(api_client, settings):
    """Test behavior with an expired token (use short lifetime in test settings)."""
    import time
    settings.SIMPLE_JWT = {
        "ACCESS_TOKEN_LIFETIME": __import__("datetime").timedelta(seconds=0),
    }
    # After expiry the server should return 401
    api_client.credentials(HTTP_AUTHORIZATION="Bearer expiredtoken")
    resp = api_client.get("/api/v1/profile/")
    assert resp.status_code == status.HTTP_401_UNAUTHORIZED


# ── APIRequestFactory — view-level unit tests ─────────────────────────────────

@pytest.mark.django_db
def test_view_directly_with_factory(user):
    from myapp.views import ArticleViewSet
    from rest_framework.test import force_authenticate

    factory = APIRequestFactory()
    request = factory.get("/api/v1/articles/")
    force_authenticate(request, user=user)

    view = ArticleViewSet.as_view({"get": "list"})
    response = view(request)
    # Must call render() to access response.data on factory-generated responses
    response.accepted_renderer = __import__("rest_framework").renderers.JSONRenderer()
    response.accepted_media_type = "application/json"
    response.renderer_context = {}
    response.render()
    assert response.status_code == status.HTTP_200_OK


@pytest.mark.django_db
def test_post_with_factory(user):
    from myapp.views import ArticleViewSet
    from rest_framework.test import force_authenticate

    factory = APIRequestFactory()
    request = factory.post(
        "/api/v1/articles/",
        {"title": "Test", "body": "Content"},
        format="json",
    )
    force_authenticate(request, user=user)
    view = ArticleViewSet.as_view({"post": "create"})
    response = view(request)
    assert response.status_code == status.HTTP_201_CREATED


# ── Response data assertions ───────────────────────────────────────────────────

@pytest.mark.django_db
def test_response_data_structure(auth_api_client):
    response = auth_api_client.get("/api/v1/articles/")
    assert response.status_code == status.HTTP_200_OK

    # Access parsed data directly (no need to json.loads)
    data = response.data
    assert "count" in data
    assert "results" in data
    assert isinstance(data["results"], list)


@pytest.mark.django_db
def test_response_headers(auth_api_client):
    response = auth_api_client.get("/api/v1/articles/")
    assert response["Content-Type"] == "application/json"
    assert "X-Request-Id" in response  # custom header example


# ── Permissions testing ────────────────────────────────────────────────────────

@pytest.mark.django_db
def test_permission_denied_for_regular_user(auth_api_client):
    response = auth_api_client.delete("/api/v1/admin/users/1/")
    assert response.status_code == status.HTTP_403_FORBIDDEN


@pytest.mark.django_db
def test_permission_granted_for_admin(admin_api_client):
    response = admin_api_client.get("/api/v1/admin/users/")
    assert response.status_code == status.HTTP_200_OK


# ── CSRF enforcement testing ───────────────────────────────────────────────────

@pytest.mark.django_db
def test_csrf_enforcement(user):
    """Test that session-authenticated requests enforce CSRF when configured."""
    client_with_csrf = APIClient(enforce_csrf_checks=True)
    client_with_csrf.login(username="testuser", password="testpass123")
    response = client_with_csrf.post("/api/v1/articles/", {"title": "T", "body": "B"})
    assert response.status_code == status.HTTP_403_FORBIDDEN  # missing CSRF token
```

### DRF error response testing

```python
@pytest.mark.django_db
def test_validation_error_response(auth_api_client):
    response = auth_api_client.post("/api/v1/articles/", {}, format="json")
    assert response.status_code == status.HTTP_400_BAD_REQUEST
    assert "title" in response.data  # field-level error
    assert "body" in response.data


@pytest.mark.django_db
def test_not_found_response(auth_api_client):
    response = auth_api_client.get("/api/v1/articles/99999/")
    assert response.status_code == status.HTTP_404_NOT_FOUND
    assert response.data["detail"] == "No Article matches the given query."


@pytest.mark.django_db
def test_method_not_allowed(auth_api_client):
    response = auth_api_client.patch("/api/v1/articles/")  # list endpoint
    assert response.status_code == status.HTTP_405_METHOD_NOT_ALLOWED
```

---

## factory_boy Integration

### Installation

```bash
pip install factory-boy
```

### `DjangoModelFactory` — basic usage

```python
# tests/factories.py
import factory
import factory.django
from django.contrib.auth import get_user_model
from myapp.models import Article, Comment, Category, Tag


class UserFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = get_user_model()
        django_get_or_create = ("username",)  # idempotent: get if exists, create otherwise

    username = factory.Sequence(lambda n: f"user{n}")
    email = factory.LazyAttribute(lambda obj: f"{obj.username}@example.com")
    password = factory.django.Password("testpass123")
    is_active = True
    is_staff = False


class SuperUserFactory(UserFactory):
    is_staff = True
    is_superuser = True
    username = factory.Sequence(lambda n: f"admin{n}")


class CategoryFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Category
        django_get_or_create = ("slug",)

    name = factory.Sequence(lambda n: f"Category {n}")
    slug = factory.LazyAttribute(lambda obj: obj.name.lower().replace(" ", "-"))


class ArticleFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Article

    title = factory.Sequence(lambda n: f"Article {n}")
    body = factory.Faker("paragraphs", nb=3, as_text=True)
    author = factory.SubFactory(UserFactory)
    category = factory.SubFactory(CategoryFactory)
    published = True
    created_at = factory.Faker("date_time_this_year", tzinfo=__import__("pytz").UTC)


class CommentFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Comment

    article = factory.SubFactory(ArticleFactory)
    author = factory.SubFactory(UserFactory)
    body = factory.Faker("sentence")
```

### Using factories in pytest fixtures

```python
# conftest.py or test files
import pytest
from tests.factories import UserFactory, ArticleFactory, CategoryFactory, CommentFactory


@pytest.fixture
def user(db):
    return UserFactory()


@pytest.fixture
def article(db):
    return ArticleFactory()


@pytest.fixture
def published_articles(db):
    """Create 5 published articles with distinct authors."""
    return ArticleFactory.create_batch(5, published=True)


@pytest.fixture
def unpublished_article(db):
    return ArticleFactory(published=False)


@pytest.fixture
def article_with_comments(db):
    article = ArticleFactory()
    CommentFactory.create_batch(3, article=article)
    return article


# ── Tests using factories ──────────────────────────────────────────────────────

@pytest.mark.django_db
def test_article_list_returns_published_only(client, published_articles, unpublished_article):
    response = client.get("/api/articles/")
    assert response.status_code == 200
    assert response.json()["count"] == 5


@pytest.mark.django_db
def test_article_author_relationship(article):
    assert article.author is not None
    assert article.author.pk is not None


@pytest.mark.django_db
def test_override_factory_attrs(db):
    """Override individual attributes at call time."""
    article = ArticleFactory(title="Custom Title", published=False)
    assert article.title == "Custom Title"
    assert article.published is False


@pytest.mark.django_db
def test_batch_creation(db):
    articles = ArticleFactory.create_batch(10, published=True)
    assert len(articles) == 10
    from myapp.models import Article
    assert Article.objects.filter(published=True).count() == 10
```

### `django_get_or_create` — idempotent factories

```python
class TagFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Tag
        django_get_or_create = ("name",)

    name = factory.Iterator(["python", "django", "testing", "api"])


@pytest.mark.django_db
def test_tag_deduplication(db):
    t1 = TagFactory(name="python")
    t2 = TagFactory(name="python")
    assert t1.pk == t2.pk  # same object retrieved
```

### `@factory.django.mute_signals`

Suppress signals during factory execution to avoid side effects (e.g., auto-created
profiles, email notifications, Celery tasks):

```python
from django.db.models import signals


@factory.django.mute_signals(signals.post_save)
class UserFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = get_user_model()

    username = factory.Sequence(lambda n: f"user{n}")
    email = factory.LazyAttribute(lambda obj: f"{obj.username}@example.com")
    password = factory.django.Password("testpass")


# Or as a context manager in a single call:
@pytest.mark.django_db
def test_create_without_post_save_signal(db):
    with factory.django.mute_signals(signals.post_save):
        user = UserFactory()
    # profile was NOT auto-created via post_save signal
    from myapp.models import UserProfile
    assert not UserProfile.objects.filter(user=user).exists()
```

### `FileField` and `ImageField`

```python
class DocumentFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Document

    name = factory.Sequence(lambda n: f"document_{n}.pdf")
    # Generate file from raw bytes
    file = factory.django.FileField(
        filename="test.pdf",
        data=b"%PDF-1.4 test pdf content",
    )


class AvatarFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = UserProfile

    user = factory.SubFactory(UserFactory)
    # Pillow required; generates a synthetic JPEG
    avatar = factory.django.ImageField(
        filename="avatar.jpg",
        width=200,
        height=200,
        color="blue",
        format="JPEG",
    )


class ProfileFromFileFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = UserProfile

    user = factory.SubFactory(UserFactory)
    # Load from an actual file on disk
    avatar = factory.django.ImageField(from_path="tests/fixtures/sample_avatar.jpg")


@pytest.mark.django_db
def test_document_upload(db, settings, tmp_path):
    settings.MEDIA_ROOT = str(tmp_path)
    doc = DocumentFactory()
    assert doc.file.name.endswith(".pdf")
    assert doc.file.size > 0


@pytest.mark.django_db
def test_avatar_dimensions(db, settings, tmp_path):
    settings.MEDIA_ROOT = str(tmp_path)
    from PIL import Image
    profile = AvatarFactory()
    with Image.open(profile.avatar.path) as img:
        assert img.size == (200, 200)
```

### `PostGeneration` and `RelatedFactory`

```python
class ArticleWithTagsFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Article

    title = factory.Sequence(lambda n: f"Article {n}")
    body = "Body"
    author = factory.SubFactory(UserFactory)

    @factory.post_generation
    def tags(self, create, extracted, **kwargs):
        if not create:
            return
        if extracted:
            for tag in extracted:
                self.tags.add(tag)
        else:
            # Default: add two random tags
            TagFactory.create_batch(2)
            self.tags.set(Tag.objects.all()[:2])


@pytest.mark.django_db
def test_article_with_explicit_tags(db):
    from myapp.models import Tag
    python_tag = TagFactory(name="python")
    django_tag = TagFactory(name="django")
    article = ArticleWithTagsFactory(tags=[python_tag, django_tag])
    assert article.tags.count() == 2
    assert article.tags.filter(name="python").exists()
```

### Traits — conditional attribute sets

```python
class ArticleFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Article

    title = factory.Sequence(lambda n: f"Article {n}")
    body = "Body text"
    author = factory.SubFactory(UserFactory)
    published = False

    class Params:
        # Activate with ArticleFactory(published_now=True)
        published_now = factory.Trait(
            published=True,
            published_at=factory.Faker("date_time_this_month", tzinfo=__import__("pytz").UTC),
        )
        # Activate with ArticleFactory(featured=True)
        featured = factory.Trait(
            published=True,
            is_featured=True,
        )


@pytest.mark.django_db
def test_published_article_trait(db):
    article = ArticleFactory(published_now=True)
    assert article.published is True
    assert article.published_at is not None
```

### Multi-database factories

```python
class AnalyticsEventFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = AnalyticsEvent
        database = "analytics"   # all operations go to the 'analytics' DB

    event_type = "page_view"
    user_id = factory.Sequence(lambda n: n)
    created_at = factory.Faker("date_time_this_year", tzinfo=__import__("pytz").UTC)
```

---

## Django Signals Testing

### Testing that a signal fires

```python
import pytest
from unittest.mock import MagicMock, patch
from django.db.models.signals import post_save, pre_delete


@pytest.mark.django_db
def test_post_save_signal_fires(db):
    """Assert a signal receiver is called when a model is saved."""
    handler = MagicMock()
    post_save.connect(handler, sender=__import__("myapp.models", fromlist=["Article"]).Article)

    try:
        from myapp.models import Article
        article = Article.objects.create(title="Test", body="Body")
        handler.assert_called_once()
        call_kwargs = handler.call_args[1]
        assert call_kwargs["instance"] == article
        assert call_kwargs["created"] is True
    finally:
        post_save.disconnect(handler, sender=Article)


@pytest.mark.django_db
def test_signal_not_fired_on_unchanged_save(db):
    from myapp.models import Article
    from django.db.models.signals import post_save

    handler = MagicMock()
    post_save.connect(handler, sender=Article)
    try:
        article = Article.objects.create(title="Test", body="Body")
        handler.reset_mock()
        article.save()  # second save — created=False
        handler.assert_called_once()
        assert handler.call_args[1]["created"] is False
    finally:
        post_save.disconnect(handler, sender=Article)
```

### Mocking signal side effects

```python
@pytest.mark.django_db
def test_welcome_email_sent_via_signal(db, mailoutbox):
    """
    UserProfile.post_save signal triggers a welcome email.
    Using mailoutbox, we verify the email is sent without mocking the signal.
    """
    from django.contrib.auth import get_user_model
    User = get_user_model()
    User.objects.create_user(username="newbie", email="newbie@example.com", password="pass")
    assert len(mailoutbox) == 1
    assert mailoutbox[0].to == ["newbie@example.com"]


@pytest.mark.django_db
def test_signal_side_effect_suppressed(db):
    """Use mute_signals to prevent notification signal during bulk creation."""
    import factory.django
    from django.db.models.signals import post_save

    with factory.django.mute_signals(post_save):
        from myapp.models import Article
        Article.objects.create(title="Silent", body="No signal")

    # No side effects (emails, tasks, etc.) were triggered
    from django.core import mail
    assert len(mail.outbox) == 0
```

### Testing signal disconnection / reconnection

```python
@pytest.fixture
def disconnect_post_save():
    """Temporarily disconnect all post_save receivers for clean tests."""
    from myapp.signals import on_article_save
    from myapp.models import Article
    post_save.disconnect(on_article_save, sender=Article)
    yield
    post_save.connect(on_article_save, sender=Article)


@pytest.mark.django_db
def test_create_article_no_side_effects(db, disconnect_post_save):
    from myapp.models import Article
    article = Article.objects.create(title="Clean", body="No side effects")
    assert article.pk is not None
```

### Using `pytest-mock` for signal spying

```python
@pytest.mark.django_db
def test_signal_called_with_correct_args(db, mocker):
    """pytest-mock: pip install pytest-mock"""
    from myapp.models import Article

    mock_handler = mocker.Mock()
    post_save.connect(mock_handler, sender=Article)

    article = Article.objects.create(title="Test", body="Body")

    mock_handler.assert_called_once()
    _, kwargs = mock_handler.call_args
    assert kwargs["sender"] is Article
    assert kwargs["instance"].title == "Test"
    assert kwargs["created"] is True
```

---

## Management Commands Testing

### `call_command()` — basic usage

```python
import pytest
from django.core.management import call_command
from io import StringIO


@pytest.mark.django_db
def test_management_command_runs(db):
    """Test that a management command runs without errors."""
    call_command("my_custom_command")


@pytest.mark.django_db
def test_command_with_arguments(db):
    """Pass positional and keyword arguments."""
    call_command("import_data", "path/to/file.csv", verbosity=0, dry_run=True)


@pytest.mark.django_db
def test_command_stdout_output(db):
    """Capture stdout to assert printed output."""
    out = StringIO()
    call_command("my_custom_command", stdout=out)
    output = out.getvalue()
    assert "Success" in output
    assert "processed 0 records" in output.lower()


@pytest.mark.django_db
def test_command_stderr_on_error(db):
    """Capture stderr to assert error messages."""
    err = StringIO()
    with pytest.raises(SystemExit):
        call_command("my_custom_command", "--invalid-option", stderr=err)
    assert "error" in err.getvalue().lower()


@pytest.mark.django_db
def test_command_creates_objects(db):
    """Assert the command creates the expected DB state."""
    from myapp.models import Category

    assert Category.objects.count() == 0
    call_command("seed_categories", verbosity=0)
    assert Category.objects.count() > 0


@pytest.mark.django_db
def test_loaddata_fixture(db):
    """Use loaddata to insert fixture data in a test."""
    from myapp.models import Category
    call_command("loaddata", "categories.json", verbosity=0)
    assert Category.objects.filter(name="General").exists()


@pytest.mark.django_db
def test_command_idempotent(db):
    """Running the command twice should not cause errors or duplicates."""
    call_command("seed_categories", verbosity=0)
    count_after_first = __import__("myapp.models", fromlist=["Category"]).Category.objects.count()

    call_command("seed_categories", verbosity=0)
    count_after_second = __import__("myapp.models", fromlist=["Category"]).Category.objects.count()

    assert count_after_first == count_after_second
```

### Testing commands that use transactions

```python
@pytest.mark.django_db(transaction=True)
def test_command_uses_transaction(db):
    """Commands that commit mid-execution require transaction=True."""
    out = StringIO()
    call_command("process_orders", stdout=out)
    assert "Committed" in out.getvalue()
```

### Testing commands with mocked external services

```python
@pytest.mark.django_db
def test_sync_command_mocks_api(db, mocker):
    """Mock external HTTP calls inside management commands."""
    mock_get = mocker.patch("myapp.management.commands.sync_products.requests.get")
    mock_get.return_value.json.return_value = [
        {"id": 1, "name": "Product A", "price": "9.99"},
        {"id": 2, "name": "Product B", "price": "19.99"},
    ]
    mock_get.return_value.status_code = 200

    out = StringIO()
    call_command("sync_products", stdout=out)

    from myapp.models import Product
    assert Product.objects.count() == 2
    mock_get.assert_called_once()
```

---

## Async Django Views Testing

Django 4.1+ supports fully async views. pytest-django provides `async_client` and
`async_rf` for testing them. Use with `pytest-asyncio`.

```bash
pip install pytest-asyncio
```

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"   # automatically handle async test functions
```

### `async_client` — integration testing

```python
import pytest


@pytest.mark.django_db(transaction=True)
async def test_async_list_view(async_client):
    response = await async_client.get("/api/async/articles/")
    assert response.status_code == 200


@pytest.mark.django_db(transaction=True)
async def test_async_post_view(async_client, db):
    import json
    response = await async_client.post(
        "/api/async/articles/",
        data=json.dumps({"title": "Async Article", "body": "Written asynchronously"}),
        content_type="application/json",
    )
    assert response.status_code == 201
    data = response.json()
    assert data["title"] == "Async Article"


@pytest.mark.django_db(transaction=True)
async def test_async_auth_required(async_client):
    response = await async_client.get("/api/async/private/")
    assert response.status_code == 302  # redirect to login


@pytest.mark.django_db(transaction=True)
async def test_async_force_login(async_client, django_user_model):
    user = await django_user_model.objects.acreate_user(
        username="asyncuser",
        password="asyncpass",
    )
    await async_client.aforce_login(user)
    response = await async_client.get("/api/async/private/")
    assert response.status_code == 200
```

### `async_rf` — unit testing async views

```python
import pytest


@pytest.mark.asyncio
async def test_async_view_unit(async_rf, db):
    from myapp.views import AsyncArticleView

    request = async_rf.get("/articles/")
    # For DB access in async views, use transaction=True on the marker
    response = await AsyncArticleView.as_view()(request)
    assert response.status_code == 200


@pytest.mark.asyncio
async def test_async_view_with_user(async_rf, admin_user):
    from myapp.views import AsyncAdminView

    request = async_rf.get("/admin/async/")
    request.user = admin_user
    response = await AsyncAdminView.as_view()(request)
    assert response.status_code == 200
```

### Testing async ORM queries

```python
@pytest.mark.django_db(transaction=True)
async def test_async_orm(db):
    from myapp.models import Article

    await Article.objects.acreate(title="Async Created", body="Body", published=True)
    count = await Article.objects.acount()
    assert count == 1

    article = await Article.objects.aget(title="Async Created")
    assert article.published is True


@pytest.mark.django_db(transaction=True)
async def test_async_queryset(db):
    from myapp.models import Article

    await Article.objects.abulk_create([
        Article(title=f"Article {i}", body="Body", published=True)
        for i in range(5)
    ])
    articles = [a async for a in Article.objects.filter(published=True)]
    assert len(articles) == 5
```

### Django 4.1+ `AsyncClient`

```python
@pytest.mark.django_db(transaction=True)
async def test_streaming_response(async_client):
    """Test a streaming HTTP response from an async view."""
    response = await async_client.get("/api/async/stream/")
    assert response.status_code == 200
    content = b"".join([chunk async for chunk in response.streaming_content])
    assert len(content) > 0
```

---

## Query Count Testing and Optimization

### `django_assert_num_queries` — detecting N+1

```python
import pytest


@pytest.mark.django_db
def test_no_n_plus_one(client, django_assert_num_queries, db):
    """Verify select_related eliminates N+1 queries for article list."""
    from myapp.models import Author, Article
    author = Author.objects.create(name="Author A")
    Article.objects.bulk_create([
        Article(title=f"Article {i}", body="body", author=author, published=True)
        for i in range(10)
    ])

    with django_assert_num_queries(2):
        # Expected: 1 session/auth + 1 articles query with JOIN
        response = client.get("/api/articles/")
    assert response.status_code == 200


@pytest.mark.django_db
def test_prefetch_related_efficiency(db, django_assert_num_queries):
    """Verify prefetch_related for M2M relationships."""
    from myapp.models import Article, Tag
    tags = Tag.objects.bulk_create([Tag(name=f"tag{i}") for i in range(5)])
    articles = Article.objects.bulk_create([
        Article(title=f"Article {i}", body="body") for i in range(3)
    ])
    for article in articles:
        article.tags.set(tags[:3])

    with django_assert_num_queries(2):
        # 1 query for articles + 1 query for tags (prefetch_related)
        result = list(Article.objects.prefetch_related("tags").all())
        for article in result:
            _ = list(article.tags.all())  # no extra queries


@pytest.mark.django_db
def test_only_needed_fields(db, django_assert_num_queries):
    """Use .only() to fetch minimal columns."""
    from myapp.models import Article
    Article.objects.bulk_create([Article(title=f"A{i}", body="body") for i in range(3)])

    with django_assert_num_queries(1):
        titles = list(Article.objects.only("title").values_list("title", flat=True))
    assert len(titles) == 3
```

### `django_assert_max_num_queries` — bounded query budget

```python
@pytest.mark.django_db
def test_dashboard_query_budget(client, django_assert_max_num_queries, db):
    """Dashboard must never exceed 20 queries regardless of data size."""
    from tests.factories import ArticleFactory, UserFactory
    ArticleFactory.create_batch(50, published=True)

    with django_assert_max_num_queries(20):
        response = client.get("/dashboard/")
    assert response.status_code == 200


@pytest.mark.django_db
def test_api_serializer_efficiency(auth_api_client, django_assert_max_num_queries, db):
    from tests.factories import ArticleFactory
    ArticleFactory.create_batch(100)

    with django_assert_max_num_queries(5):
        response = auth_api_client.get("/api/v1/articles/?page=1&page_size=25")
    assert response.status_code == 200
```

### Using `connection` parameter for non-default DB

```python
@pytest.mark.django_db(databases=["default", "analytics"])
def test_analytics_query_count(db, django_assert_num_queries):
    from django.db import connections
    analytics_conn = connections["analytics"]

    with django_assert_num_queries(1, connection=analytics_conn):
        from myapp.models import AnalyticsEvent
        list(AnalyticsEvent.objects.using("analytics").filter(event_type="page_view"))
```

### Custom `assertNumQueries` context manager

```python
@pytest.mark.django_db
def test_nested_query_contexts(db, django_assert_num_queries):
    from myapp.models import Article

    # Outer context: total queries
    with django_assert_num_queries(3):
        # 1: get all articles
        articles = list(Article.objects.all())
        # 2: count
        count = Article.objects.count()
        # 3: get specific
        if articles:
            _ = Article.objects.get(pk=articles[0].pk)
```

### Capturing and inspecting queries

```python
@pytest.mark.django_db
def test_inspect_sql_queries(db):
    """Use connection.queries to print/inspect executed SQL."""
    from django.db import connection, reset_queries
    from django.test.utils import override_settings
    from myapp.models import Article

    reset_queries()
    with override_settings(DEBUG=True):
        Article.objects.create(title="Debug", body="Body")
        Article.objects.filter(published=True).select_related("author").all()[:5]
        for q in connection.queries:
            print(q["sql"])
        assert len(connection.queries) >= 1
```

---

## on_commit Callbacks — Advanced Patterns

### Celery task dispatch on commit

```python
@pytest.mark.django_db
def test_task_dispatched_after_order_creation(db, django_capture_on_commit_callbacks):
    from unittest.mock import patch

    with patch("myapp.tasks.process_order.delay") as mock_delay:
        with django_capture_on_commit_callbacks(execute=True):
            from myapp.services import create_order
            order = create_order(user_id=1, product_id=42, quantity=2)

    mock_delay.assert_called_once_with(order.pk)


@pytest.mark.django_db
def test_multiple_on_commit_callbacks(db, django_capture_on_commit_callbacks, mailoutbox):
    from django.db import transaction

    captured = []

    def callback_a():
        captured.append("a")

    def callback_b():
        captured.append("b")

    with django_capture_on_commit_callbacks(execute=True):
        with transaction.atomic():
            transaction.on_commit(callback_a)
            transaction.on_commit(callback_b)

    assert captured == ["a", "b"]


@pytest.mark.django_db
def test_nested_atomic_callbacks(db, django_capture_on_commit_callbacks):
    from django.db import transaction

    results = []

    with django_capture_on_commit_callbacks(execute=True):
        with transaction.atomic():
            with transaction.atomic():
                # Inner savepoint commit doesn't trigger on_commit
                transaction.on_commit(lambda: results.append("inner"))
            # Only the outermost commit triggers on_commit
        # After exiting the outermost atomic block, callbacks execute

    assert "inner" in results


@pytest.mark.django_db
def test_on_commit_with_specific_db(db, django_capture_on_commit_callbacks):
    """Capture on_commit callbacks for a specific database connection."""
    from django.db import transaction

    fired = []

    with django_capture_on_commit_callbacks(using="default", execute=True):
        with transaction.atomic(using="default"):
            transaction.on_commit(lambda: fired.append(True), using="default")

    assert fired == [True]
```

---

## Multi-Database Testing

### Enabling access to multiple databases

```python
@pytest.mark.django_db(databases=["default", "analytics"])
def test_cross_database_operation(db):
    from myapp.models import Order, AnalyticsEvent

    order = Order.objects.create(amount=100, status="completed")
    # Write an analytics event to the secondary database
    AnalyticsEvent.objects.using("analytics").create(
        event_type="order_completed",
        reference_id=order.pk,
    )
    assert AnalyticsEvent.objects.using("analytics").count() == 1


@pytest.mark.django_db(databases="__all__")
def test_all_databases_accessible():
    from django.db import connections
    for alias in connections:
        with connections[alias].cursor() as cursor:
            cursor.execute("SELECT 1")
            result = cursor.fetchone()
        assert result == (1,)
```

### Fixtures with multi-database factories

```python
@pytest.fixture
@pytest.mark.django_db(databases=["default", "analytics"])
def analytics_event(db):
    from myapp.models import AnalyticsEvent
    return AnalyticsEvent.objects.using("analytics").create(
        event_type="test_event",
        user_id=1,
    )


@pytest.mark.django_db(databases=["default", "analytics"])
def test_event_exists(analytics_event):
    from myapp.models import AnalyticsEvent
    assert AnalyticsEvent.objects.using("analytics").filter(event_type="test_event").exists()
```

---

## Custom Test Database Setup

### Override `django_db_setup` for session-scoped custom DB

```python
# conftest.py
import pytest


@pytest.fixture(scope="session")
def django_db_setup():
    """
    Use a pre-existing PostgreSQL database cloned from a template.
    No migrations are run — the DB is pre-populated.
    """
    import psycopg
    from django.conf import settings

    def run_sql(sql, params=None):
        with psycopg.connect("dbname=postgres") as conn:
            conn.autocommit = True
            conn.execute(sql, params)

    settings.DATABASES["default"]["NAME"] = "test_db_from_template"
    run_sql("DROP DATABASE IF EXISTS test_db_from_template")
    run_sql("CREATE DATABASE test_db_from_template TEMPLATE production_snapshot")

    yield

    from django.db import connections
    for conn in connections.all():
        conn.close()
    run_sql("DROP DATABASE test_db_from_template")
```

### Loading fixtures once per session

```python
@pytest.fixture(scope="session")
def django_db_setup(django_db_setup, django_db_blocker):
    """Run loaddata once per test session — much faster than per-test."""
    from django.core.management import call_command
    with django_db_blocker.unblock():
        call_command("loaddata", "initial_categories.json", verbosity=0)
        call_command("loaddata", "initial_tags.json", verbosity=0)
```

### External database (no migration)

```python
@pytest.fixture(scope="session")
def django_db_setup():
    from django.conf import settings
    settings.DATABASES["default"] = {
        "ENGINE": "django.db.backends.postgresql",
        "HOST": "test-db.internal",
        "PORT": "5432",
        "NAME": "integration_test_db",
        "USER": "test_reader",
        "PASSWORD": "readonly_password",
    }
```

### Read-only database (no create/teardown)

```python
@pytest.fixture(scope="session")
def django_db_setup():
    """Connect to an existing read-only DB; don't create or drop anything."""
    pass  # pytest-django skips DB setup/teardown when this is overridden


@pytest.fixture
def readonly_db_access(request, django_db_setup, django_db_blocker):
    """Grant test functions access to the read-only DB."""
    django_db_blocker.unblock()
    yield
    django_db_blocker.restore()


@pytest.mark.usefixtures("readonly_db_access")
def test_readonly_query():
    from myapp.models import Product
    assert Product.objects.count() > 0
```

### Randomising PostgreSQL sequences

```python
@pytest.fixture(scope="session")
def django_db_setup(django_db_setup, django_db_blocker):
    """Randomise PK sequences to catch tests that rely on specific IDs."""
    import random
    from django.db import connection

    with django_db_blocker.unblock():
        with connection.cursor() as cur:
            cur.execute("""
                SELECT sequence_name
                FROM information_schema.sequences
                WHERE sequence_schema = 'public'
            """)
            for (seq_name,) in cur.fetchall():
                start = random.randint(1000, 9999)
                cur.execute(f"ALTER SEQUENCE {seq_name} RESTART WITH %s", [start])
```

---

## Testcontainers with pytest-django

Use real Docker containers instead of SQLite for more production-faithful tests.

```bash
pip install testcontainers[postgres,redis]
```

### PostgreSQL via testcontainers

```python
# conftest.py
import pytest
from testcontainers.postgres import PostgresContainer


@pytest.fixture(scope="session")
def postgres_container():
    """Start a PostgreSQL Docker container once for the entire test session."""
    with PostgresContainer("postgres:16-alpine") as postgres:
        yield postgres


@pytest.fixture(scope="session")
def django_db_setup(postgres_container):
    """Wire the running container into Django's DATABASES setting."""
    from django.conf import settings
    from django.test.utils import setup_test_environment

    settings.DATABASES["default"] = {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": postgres_container.dbname,
        "USER": postgres_container.username,
        "PASSWORD": postgres_container.password,
        "HOST": postgres_container.get_container_host_ip(),
        "PORT": postgres_container.get_exposed_port(5432),
    }

    # Run migrations against the fresh container
    from django.core.management import call_command
    call_command("migrate", "--run-syncdb", verbosity=0)
```

### Redis via testcontainers

```python
from testcontainers.redis import RedisContainer


@pytest.fixture(scope="session")
def redis_container():
    with RedisContainer("redis:7-alpine") as redis:
        yield redis


@pytest.fixture(scope="session", autouse=True)
def configure_redis(redis_container):
    """Set Django's CACHES to use the test Redis container."""
    from django.conf import settings
    host = redis_container.get_container_host_ip()
    port = redis_container.get_exposed_port(6379)
    settings.CACHES = {
        "default": {
            "BACKEND": "django.core.cache.backends.redis.RedisCache",
            "LOCATION": f"redis://{host}:{port}/1",
        }
    }


@pytest.mark.django_db
def test_cache_with_real_redis(db, settings):
    from django.core.cache import cache
    cache.set("test_key", "test_value", timeout=30)
    assert cache.get("test_key") == "test_value"
    cache.delete("test_key")
    assert cache.get("test_key") is None
```

### Generic Docker container for any service

```python
from testcontainers.core.container import DockerContainer
from testcontainers.core.waiting_utils import wait_for_logs


@pytest.fixture(scope="session")
def elasticsearch_container():
    with (
        DockerContainer("docker.elastic.co/elasticsearch/elasticsearch:8.12.0")
        .with_env("discovery.type", "single-node")
        .with_env("xpack.security.enabled", "false")
        .with_exposed_ports(9200)
    ) as container:
        wait_for_logs(container, "started", timeout=60)
        yield container


@pytest.fixture(scope="session", autouse=True)
def configure_elasticsearch(elasticsearch_container):
    from django.conf import settings
    host = elasticsearch_container.get_container_host_ip()
    port = elasticsearch_container.get_exposed_port(9200)
    settings.ELASTICSEARCH_DSL = {
        "default": {"hosts": f"http://{host}:{port}"}
    }
```

---

## Celery Task Testing

### Strategy 1: Unit test — mock `.delay()` / `.apply_async()`

The preferred approach for unit tests. Verify the correct task is called with the
correct arguments without running a real worker.

```python
import pytest
from unittest.mock import patch, call


@pytest.mark.django_db
def test_order_enqueues_task(db, mocker):
    mock_delay = mocker.patch("myapp.tasks.process_order.delay")
    from myapp.services import create_order
    order = create_order(user_id=1, product_id=42, quantity=2)
    mock_delay.assert_called_once_with(order.pk)


@pytest.mark.django_db
def test_bulk_notification_tasks(db, mocker):
    mock_apply = mocker.patch("myapp.tasks.send_notification.apply_async")
    from myapp.services import notify_all_users
    notify_all_users(message="System maintenance tonight")
    # One task per user
    assert mock_apply.call_count == 3
```

### Strategy 2: `CELERY_TASK_ALWAYS_EAGER = True` — synchronous execution

Tasks run inline (in the same process, same thread) without a worker. Good for
basic integration tests but does not replicate real serialization/deserialization.

```python
@pytest.fixture(autouse=True)
def eager_celery(settings):
    settings.CELERY_TASK_ALWAYS_EAGER = True
    settings.CELERY_TASK_EAGER_PROPAGATES = True


@pytest.mark.django_db
def test_task_runs_eagerly(db):
    from myapp.tasks import update_article_stats
    from myapp.models import Article

    article = Article.objects.create(title="Test", body="Body")
    result = update_article_stats.delay(article.pk)

    assert result.successful()
    article.refresh_from_db()
    assert article.view_count == 1
```

### Strategy 3: `celery.contrib.pytest` — real embedded worker

Tests run against an actual Celery worker in a thread. Requires a broker.

```bash
pip install "celery[pytest]" redis
```

```python
# conftest.py
import pytest

pytest_plugins = ("celery.contrib.pytest",)


@pytest.fixture(scope="session")
def celery_config():
    return {
        "broker_url": "redis://localhost:6379/0",
        "result_backend": "redis://localhost:6379/0",
        "task_always_eager": False,
    }


@pytest.fixture(scope="session")
def celery_worker_parameters():
    return {"queues": ["default", "high_priority"]}
```

```python
@pytest.mark.django_db(transaction=True)
def test_task_with_real_worker(celery_worker, db):
    from myapp.tasks import process_order

    result = process_order.delay(order_id=1)
    result.get(timeout=10)  # block until task completes

    assert result.successful()


@pytest.mark.django_db(transaction=True)
def test_task_retry_behavior(celery_worker, db, mocker):
    from myapp.tasks import flaky_external_call

    mocker.patch("myapp.services.external_api.call", side_effect=[
        Exception("Temporary failure"),  # first call fails
        {"status": "ok"},               # second call succeeds
    ])

    result = flaky_external_call.delay()
    result.get(timeout=30)
    assert result.successful()
```

### Testing task failure handling

```python
@pytest.mark.django_db
def test_task_failure_updates_order_status(db, settings):
    settings.CELERY_TASK_ALWAYS_EAGER = True
    settings.CELERY_TASK_EAGER_PROPAGATES = False

    from myapp.tasks import process_payment
    from myapp.models import Order

    order = Order.objects.create(amount=100, status="pending")

    # Patch to simulate payment gateway failure
    with patch("myapp.services.payment.charge", side_effect=Exception("Card declined")):
        result = process_payment.delay(order.pk)

    order.refresh_from_db()
    assert order.status == "failed"
```

---

## Static Files and Media Files

### Static files in tests

```python
@pytest.fixture(autouse=True)
def no_staticfiles_finders(settings):
    """
    Prevent Django from searching for static files during tests.
    Tests generally don't need actual CSS/JS assets.
    """
    settings.STATICFILES_FINDERS = []


@pytest.mark.django_db
def test_static_url_in_template(client, settings):
    settings.STATIC_URL = "/static/"
    response = client.get("/")
    assert response.status_code == 200
    # Template uses {% static %} tag; just check the URL pattern
    assert "/static/" in response.content.decode()
```

### Media files — using `tmp_path`

```python
@pytest.fixture
def media_root(settings, tmp_path):
    """Isolate media files to a temporary directory per test."""
    media = tmp_path / "media"
    media.mkdir()
    settings.MEDIA_ROOT = str(media)
    settings.DEFAULT_FILE_STORAGE = "django.core.files.storage.FileSystemStorage"
    return media


@pytest.mark.django_db
def test_file_upload(client, user, media_root, db):
    from django.core.files.uploadedfile import SimpleUploadedFile

    client.force_login(user)
    file_content = b"Hello, this is a test file content."
    upload = SimpleUploadedFile("testfile.txt", file_content, content_type="text/plain")

    response = client.post("/api/upload/", {"file": upload})
    assert response.status_code == 201

    from myapp.models import UploadedFile
    obj = UploadedFile.objects.get()
    assert obj.file.read() == file_content


@pytest.mark.django_db
def test_image_upload(client, user, media_root, db):
    from io import BytesIO
    from PIL import Image
    from django.core.files.uploadedfile import InMemoryUploadedFile

    # Create an in-memory image
    img = Image.new("RGB", (100, 100), color="red")
    buf = BytesIO()
    img.save(buf, format="JPEG")
    buf.seek(0)
    upload = InMemoryUploadedFile(
        buf, "avatar", "test.jpg", "image/jpeg", buf.getbuffer().nbytes, None
    )

    client.force_login(user)
    response = client.post("/api/profile/avatar/", {"avatar": upload})
    assert response.status_code == 200

    from myapp.models import UserProfile
    profile = UserProfile.objects.get(user=user)
    assert profile.avatar.name.endswith(".jpg")
```

### Cleanup of media files after tests

```python
@pytest.fixture(autouse=True)
def clean_media(settings, tmp_path):
    """Use tmp_path so pytest auto-cleans media files after each test."""
    settings.MEDIA_ROOT = str(tmp_path / "media")
    (tmp_path / "media").mkdir(exist_ok=True)
    yield
    # tmp_path is automatically removed by pytest — no manual cleanup needed
```

### Testing file serving views

```python
@pytest.mark.django_db
def test_protected_file_download(client, user, media_root, db):
    from django.core.files.base import ContentFile
    from myapp.models import Document

    doc = Document.objects.create(owner=user)
    doc.file.save("secret.pdf", ContentFile(b"PDF content"))

    client.force_login(user)
    response = client.get(f"/api/documents/{doc.pk}/download/")
    assert response.status_code == 200
    # For X-Accel-Redirect or X-Sendfile patterns:
    # assert response["X-Accel-Redirect"] == f"/protected/{doc.file.name}"
```

---

## Django Channels WebSocket Testing

### Installation

```bash
pip install channels pytest-asyncio
```

### Testing with `WebsocketCommunicator`

```python
import pytest
from channels.testing import WebsocketCommunicator
from myapp.asgi import application  # your ASGI application


@pytest.mark.asyncio
async def test_websocket_connect():
    communicator = WebsocketCommunicator(application, "/ws/chat/room1/")
    connected, subprotocol = await communicator.connect()
    assert connected is True
    await communicator.disconnect()


@pytest.mark.asyncio
async def test_websocket_echo():
    communicator = WebsocketCommunicator(application, "/ws/echo/")
    connected, _ = await communicator.connect()
    assert connected

    await communicator.send_to(text_data="hello server")
    response = await communicator.receive_from()
    assert response == "echo: hello server"

    await communicator.disconnect()


@pytest.mark.asyncio
async def test_websocket_json_exchange():
    communicator = WebsocketCommunicator(application, "/ws/notifications/")
    await communicator.connect()

    await communicator.send_json_to({"type": "subscribe", "channel": "alerts"})
    response = await communicator.receive_json_from(timeout=2)

    assert response["type"] == "subscribed"
    assert response["channel"] == "alerts"
    await communicator.disconnect()


@pytest.mark.asyncio
async def test_websocket_disconnect_closes_cleanly():
    communicator = WebsocketCommunicator(application, "/ws/chat/")
    connected, _ = await communicator.connect()
    assert connected

    # Verify no messages are queued
    assert await communicator.receive_nothing(timeout=0.1) is True
    await communicator.disconnect()


@pytest.mark.asyncio
async def test_websocket_server_sends_message():
    """Test that the server pushes a message to the client."""
    communicator = WebsocketCommunicator(application, "/ws/live-updates/")
    await communicator.connect()

    # Trigger a server-side push (e.g., via channel layer)
    from channels.layers import get_channel_layer
    channel_layer = get_channel_layer()
    await channel_layer.group_send(
        "live_updates",
        {"type": "update.message", "text": "Stock price updated"},
    )

    response = await communicator.receive_from(timeout=2)
    assert "Stock price updated" in response
    await communicator.disconnect()
```

### Authenticated WebSocket connections

```python
@pytest.mark.asyncio
@pytest.mark.django_db(transaction=True)
async def test_authenticated_websocket(django_user_model):
    """
    Channels doesn't use Django's test client for auth.
    Inject auth data via the scope's 'user' key.
    """
    user = await django_user_model.objects.acreate_user(
        username="wsuser",
        password="wspass",
    )

    communicator = WebsocketCommunicator(
        application,
        "/ws/private/",
        headers=[(b"origin", b"http://localhost")],
    )
    # Inject the authenticated user directly into the scope
    communicator.scope["user"] = user

    connected, _ = await communicator.connect()
    assert connected

    await communicator.send_json_to({"type": "ping"})
    response = await communicator.receive_json_from(timeout=2)
    assert response["type"] == "pong"

    await communicator.disconnect()
```

### Testing consumer with `ApplicationCommunicator`

```python
@pytest.mark.asyncio
async def test_http_consumer():
    from channels.testing import HttpCommunicator
    from myapp.consumers import HealthCheckConsumer

    communicator = HttpCommunicator(HealthCheckConsumer.as_asgi(), "GET", "/health/")
    response = await communicator.get_response()
    assert response["status"] == 200
    assert b"ok" in response["body"]
```

### Using `ChannelsLiveServerTestCase`

```python
from channels.testing import ChannelsLiveServerTestCase


class WebSocketLiveTest(ChannelsLiveServerTestCase):
    serve_static = True  # serve static files via the test server

    def test_websocket_live(self):
        import websocket  # websocket-client library
        ws = websocket.create_connection(
            self.live_server_url.replace("http", "ws") + "/ws/echo/"
        )
        ws.send("hello")
        result = ws.recv()
        self.assertEqual(result, "echo: hello")
        ws.close()
```

---

## Complete Real-World conftest.py

A production-ready `conftest.py` combining all the patterns from this document.
This example is suitable for a Django + DRF + Celery + Channels project.

```python
# conftest.py
"""
Shared pytest fixtures for the entire test suite.

Scope hierarchy used here:
  session  → container startup, DB setup (expensive)
  module   → data shared across tests in a file
  function → default; each test gets a fresh copy
"""
import pytest


# ══════════════════════════════════════════════════════════════════════════════
# Performance: autouse fixtures applied to every test
# ══════════════════════════════════════════════════════════════════════════════

@pytest.fixture(autouse=True)
def fast_password_hasher(settings):
    """Replace PBKDF2/bcrypt with MD5 — makes create_user() ~100× faster."""
    settings.PASSWORD_HASHERS = ["django.contrib.auth.hashers.MD5PasswordHasher"]


@pytest.fixture(autouse=True)
def no_real_cache(settings):
    """Prevent tests from polluting or reading a shared cache."""
    settings.CACHES = {
        "default": {"BACKEND": "django.core.cache.backends.dummy.DummyCache"}
    }


@pytest.fixture(autouse=True)
def locmem_email(settings):
    """Capture emails in memory; never send real emails in tests."""
    settings.EMAIL_BACKEND = "django.core.mail.backends.locmem.EmailBackend"


@pytest.fixture(autouse=True)
def eager_celery(settings):
    """Run Celery tasks synchronously and propagate exceptions."""
    settings.CELERY_TASK_ALWAYS_EAGER = True
    settings.CELERY_TASK_EAGER_PROPAGATES = True


@pytest.fixture(autouse=True)
def no_throttling(settings):
    """Remove DRF throttling so tests don't hit rate limits."""
    settings.REST_FRAMEWORK = {
        **getattr(settings, "REST_FRAMEWORK", {}),
        "DEFAULT_THROTTLE_CLASSES": [],
        "DEFAULT_THROTTLE_RATES": {},
    }


@pytest.fixture(autouse=True)
def isolated_media(settings, tmp_path):
    """Each test gets its own media directory; auto-cleaned by pytest."""
    media_dir = tmp_path / "media"
    media_dir.mkdir()
    settings.MEDIA_ROOT = str(media_dir)
    settings.DEFAULT_FILE_STORAGE = "django.core.files.storage.FileSystemStorage"


# ══════════════════════════════════════════════════════════════════════════════
# User fixtures
# ══════════════════════════════════════════════════════════════════════════════

@pytest.fixture
def user(db, django_user_model):
    """A regular, active user."""
    return django_user_model.objects.create_user(
        username="testuser",
        email="testuser@example.com",
        password="testpass123",
        first_name="Test",
        last_name="User",
    )


@pytest.fixture
def inactive_user(db, django_user_model):
    """An inactive user (cannot log in)."""
    return django_user_model.objects.create_user(
        username="inactive",
        email="inactive@example.com",
        password="testpass123",
        is_active=False,
    )


@pytest.fixture
def superuser(db, django_user_model):
    """A Django superuser with all permissions."""
    return django_user_model.objects.create_superuser(
        username="admin",
        email="admin@example.com",
        password="adminpass123",
    )


@pytest.fixture
def staff_user(db, django_user_model):
    """A staff user with is_staff=True but not superuser."""
    return django_user_model.objects.create_user(
        username="staff",
        email="staff@example.com",
        password="staffpass123",
        is_staff=True,
    )


# ══════════════════════════════════════════════════════════════════════════════
# Django test client fixtures
# ══════════════════════════════════════════════════════════════════════════════

@pytest.fixture
def auth_client(client, user):
    """Django test client authenticated as a regular user."""
    client.force_login(user)
    return client


@pytest.fixture
def staff_client(client, staff_user):
    """Django test client authenticated as a staff user."""
    client.force_login(staff_user)
    return client


@pytest.fixture
def superuser_client(client, superuser):
    """Django test client authenticated as a superuser."""
    client.force_login(superuser)
    return client


# ══════════════════════════════════════════════════════════════════════════════
# DRF APIClient fixtures
# ══════════════════════════════════════════════════════════════════════════════

@pytest.fixture
def api_client():
    """Unauthenticated DRF APIClient."""
    from rest_framework.test import APIClient
    return APIClient()


@pytest.fixture
def auth_api_client(api_client, user):
    """DRF APIClient authenticated as a regular user (force_authenticate)."""
    api_client.force_authenticate(user=user)
    return api_client


@pytest.fixture
def staff_api_client(api_client, staff_user):
    """DRF APIClient authenticated as a staff user."""
    api_client.force_authenticate(user=staff_user)
    return api_client


@pytest.fixture
def admin_api_client(api_client, superuser):
    """DRF APIClient authenticated as a superuser."""
    api_client.force_authenticate(user=superuser)
    return api_client


@pytest.fixture
def token_api_client(api_client, user):
    """DRF APIClient using Token authentication (real HTTP header)."""
    from rest_framework.authtoken.models import Token
    token = Token.objects.create(user=user)
    api_client.credentials(HTTP_AUTHORIZATION=f"Token {token.key}")
    return api_client


# ══════════════════════════════════════════════════════════════════════════════
# Reusable domain fixtures (customize to your models)
# ══════════════════════════════════════════════════════════════════════════════

@pytest.fixture
def category(db):
    from myapp.models import Category
    return Category.objects.create(name="General", slug="general")


@pytest.fixture
def article(db, user, category):
    from myapp.models import Article
    return Article.objects.create(
        title="Test Article",
        body="This is the test body.",
        author=user,
        category=category,
        published=True,
    )


@pytest.fixture
def draft_article(db, user, category):
    from myapp.models import Article
    return Article.objects.create(
        title="Draft Article",
        body="Draft content.",
        author=user,
        category=category,
        published=False,
    )


# ══════════════════════════════════════════════════════════════════════════════
# Factory-based batch fixtures
# ══════════════════════════════════════════════════════════════════════════════

@pytest.fixture
def many_articles(db):
    """20 published articles from factory_boy for pagination/filter tests."""
    from tests.factories import ArticleFactory
    return ArticleFactory.create_batch(20, published=True)


@pytest.fixture
def many_users(db):
    """10 regular users for bulk operation tests."""
    from tests.factories import UserFactory
    return UserFactory.create_batch(10)


# ══════════════════════════════════════════════════════════════════════════════
# Async client fixtures
# ══════════════════════════════════════════════════════════════════════════════

@pytest.fixture
async def async_auth_client(async_client, django_user_model, db):
    """Async Django test client authenticated as a regular user."""
    user = await django_user_model.objects.acreate_user(
        username="asyncuser",
        email="asyncuser@example.com",
        password="asyncpass123",
    )
    await async_client.aforce_login(user)
    return async_client


# ══════════════════════════════════════════════════════════════════════════════
# Session-scoped fixtures (run once per test session)
# ══════════════════════════════════════════════════════════════════════════════

@pytest.fixture(scope="session")
def django_db_setup(django_db_setup, django_db_blocker):
    """
    Load global reference data once.
    Override django_db_setup from pytest-django; then chain to the default.
    """
    from django.core.management import call_command
    with django_db_blocker.unblock():
        # Load reference/lookup tables used by many tests
        call_command("loaddata", "countries.json", verbosity=0)
        call_command("loaddata", "currencies.json", verbosity=0)


# ══════════════════════════════════════════════════════════════════════════════
# Utility fixtures
# ══════════════════════════════════════════════════════════════════════════════

@pytest.fixture
def freeze_time():
    """
    Freeze time for a test using freezegun (pip install freezegun).
    Usage: with freeze_time("2024-01-15 12:00:00"): ...
    Or as a fixture:
        def test_something(freeze_time):
            with freeze_time("2024-06-01"):
                ...
    """
    from freezegun import freeze_time as _freeze_time
    return _freeze_time


@pytest.fixture
def mock_requests(requests_mock):
    """
    Fixture that provides a requests-mock session (pip install requests-mock).
    Usage:
        def test_external_call(mock_requests):
            mock_requests.get("https://api.example.com/data", json={"key": "val"})
    """
    return requests_mock


@pytest.fixture
def capture_queries(db):
    """
    Context manager that captures and exposes all SQL queries executed.
    Usage:
        def test_queries(capture_queries):
            with capture_queries() as q:
                MyModel.objects.all()
            print(q.queries)
    """
    from django.test.utils import CaptureQueriesContext
    from django.db import connection

    class _Capture:
        def __call__(self):
            return CaptureQueriesContext(connection)

    return _Capture()
```
