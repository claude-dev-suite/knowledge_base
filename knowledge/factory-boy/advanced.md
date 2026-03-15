# factory_boy — Advanced Patterns

> **Source:** https://factoryboy.readthedocs.io/en/stable/
> **Prerequisite:** Read `basics.md` first.

---

## Complex `SubFactory` Chains

When you have deeply nested model relationships (e.g., Author → Book → Chapter),
each level delegates to its own factory and overrides cascade via double-underscore
notation.

```python
import factory
from factory.django import DjangoModelFactory
from myapp.models import Author, Book, Chapter

class AuthorFactory(DjangoModelFactory):
    class Meta:
        model = Author

    first_name = factory.Faker("first_name")
    last_name  = factory.Faker("last_name")
    email      = factory.LazyAttribute(
        lambda o: f"{o.first_name.lower()}.{o.last_name.lower()}@example.com"
    )

class BookFactory(DjangoModelFactory):
    class Meta:
        model = Book

    title  = factory.Faker("sentence", nb_words=5)
    author = factory.SubFactory(AuthorFactory)
    year   = factory.Faker("year")

class ChapterFactory(DjangoModelFactory):
    class Meta:
        model = Chapter

    title    = factory.Faker("sentence", nb_words=4)
    number   = factory.Sequence(lambda n: n + 1)
    book     = factory.SubFactory(BookFactory)
    word_count = factory.Faker("pyint", min_value=500, max_value=8000)

# --- Usage examples ---

# Create a chapter; book and author are created automatically
chapter = ChapterFactory.create()

# Override at any depth
chapter = ChapterFactory.create(
    book__title="Clean Code",
    book__author__first_name="Robert",
    book__author__last_name="Martin",
)

# Reuse an existing book for multiple chapters (shares FK, avoids extra inserts)
book = BookFactory.create()
chapters = ChapterFactory.create_batch(5, book=book)
```

---

## M2M Relations with `@post_generation`

Many-to-many relationships cannot be set in `objects.create()`, so they must
be handled after the object exists. The `@post_generation` hook is the
canonical way.

```python
from django.contrib.auth.models import Group, Permission

class GroupFactory(DjangoModelFactory):
    class Meta:
        model = Group

    name = factory.Sequence(lambda n: f"group_{n}")

class UserFactory(DjangoModelFactory):
    class Meta:
        model = "auth.User"

    username = factory.Faker("user_name")
    email    = factory.Faker("email")
    password = factory.PostGenerationMethodCall("set_password", "testpass123")

    @factory.post_generation
    def groups(obj, create, extracted, **kwargs):
        """
        Usage:
          UserFactory()                          → no groups
          UserFactory(groups=[g1, g2])           → explicit groups
          UserFactory(groups=True)               → auto-create 2 random groups
        """
        if not create:
            return
        if extracted is True:
            # Auto-create groups when caller passes groups=True
            obj.groups.add(GroupFactory(), GroupFactory())
        elif extracted:
            obj.groups.set(extracted)

    @factory.post_generation
    def permissions(obj, create, extracted, **kwargs):
        if not create or not extracted:
            return
        obj.user_permissions.set(extracted)

# Usage:
staff_group = GroupFactory(name="staff")
user = UserFactory(groups=[staff_group])

# Using kwargs to pass to a nested factory via post_generation:
class ArticleFactory(DjangoModelFactory):
    class Meta:
        model = Article

    @factory.post_generation
    def tags(obj, create, extracted, **kwargs):
        if not create:
            return
        if extracted:
            obj.tags.set(extracted)
        else:
            obj.tags.set(TagFactory.create_batch(2))
```

---

## `RelatedFactory` with Multiple Related Objects

Use `RelatedFactoryList` (or multiple `RelatedFactory` declarations) to
generate a collection of related objects after the parent is created.

```python
class PublisherFactory(DjangoModelFactory):
    class Meta:
        model = Publisher

    name = factory.Faker("company")

    # Always create 3 associated books
    books = factory.RelatedFactoryList(
        BookFactory,
        factory_related_name="publisher",
        size=3,
    )

# Variable size via callable:
class CatalogFactory(DjangoModelFactory):
    class Meta:
        model = Catalog

    products = factory.RelatedFactoryList(
        ProductFactory,
        factory_related_name="catalog",
        size=lambda: random.randint(2, 10),
    )

# Multiple distinct RelatedFactory declarations:
class TeamFactory(DjangoModelFactory):
    class Meta:
        model = Team

    captain  = factory.RelatedFactory(
        UserFactory, factory_related_name="captained_team", role="captain"
    )
    co_pilot = factory.RelatedFactory(
        UserFactory, factory_related_name="co_piloted_team", role="co-pilot"
    )
```

---

## `Trait` with `Meta.exclude` for Computed Fields

Combine traits with `Meta.exclude` when you need intermediate values that
should influence multiple fields but must not be passed to the model
constructor.

```python
import datetime, factory
from factory.django import DjangoModelFactory

class OrderFactory(DjangoModelFactory):
    class Meta:
        model  = Order
        exclude = ("now",)   # "now" is used in declarations but not sent to Order()

    # Helper field — excluded from the model
    now = factory.LazyFunction(datetime.datetime.utcnow)

    # Fields derived from "now"
    created_at = factory.LazyAttribute(lambda o: o.now - datetime.timedelta(days=7))
    updated_at = factory.LazyAttribute(lambda o: o.now - datetime.timedelta(hours=1))
    state      = "pending"
    shipped_at = None

    class Params:
        shipped = factory.Trait(
            state      = "shipped",
            shipped_at = factory.LazyAttribute(
                lambda o: o.now - datetime.timedelta(days=1)
            ),
        )
        cancelled = factory.Trait(
            state = "cancelled",
        )
        old = factory.Trait(
            # Override "now" to simulate an old record
            now = factory.LazyFunction(
                lambda: datetime.datetime.utcnow() - datetime.timedelta(days=365)
            ),
        )

# An old shipped order:
order = OrderFactory(old=True, shipped=True)
```

---

## Factory Inheritance and Abstract Factories

Use abstract factories to share common declarations (e.g., audit fields)
without instantiating them directly.

```python
class TimestampedFactory(factory.Factory):
    """Abstract base that adds created_at / updated_at to any model."""

    class Meta:
        abstract = True   # no model — cannot be called directly

    created_at = factory.LazyFunction(datetime.datetime.utcnow)
    updated_at = factory.LazyFunction(datetime.datetime.utcnow)


class UserFactory(TimestampedFactory):
    class Meta:
        model = User   # inherits created_at, updated_at

    username = factory.Faker("user_name")
    email    = factory.Faker("email")


class ArticleFactory(TimestampedFactory):
    class Meta:
        model = Article

    title = factory.Faker("sentence")


# Concrete factory inheritance — override specific fields:
class AdminUserFactory(UserFactory):
    """Inherits all UserFactory fields but forces staff/admin flags."""
    is_staff     = True
    is_superuser = True


class InactiveUserFactory(UserFactory):
    is_active = False
    deactivated_at = factory.LazyFunction(datetime.datetime.utcnow)
```

Sequence counters are shared between parent and child factories by default.
To reset the child factory's counter independently:

```python
AdminUserFactory.reset_sequence(0, force=True)
```

---

## Overriding Factories in Tests

Any declared field can be overridden at call time. This is the primary
mechanism for test-specific customization.

```python
# Override simple fields
user = UserFactory(username="alice", email="alice@test.com")

# Override sub-factory fields
chapter = ChapterFactory(book__author__first_name="Alice")

# Override to None to skip a SubFactory
article = ArticleFactory(author=None)   # author is left unset

# Override a RelatedFactory to prevent related object creation
country = CountryFactory(capital_city=None)

# Combine trait + overrides
order = OrderFactory(shipped=True, shipped_at=datetime.datetime(2024, 6, 1))

# Activate multiple traits
user = UserFactory(is_staff=True, is_verified=True)   # two Params traits
```

---

## `factory.random_seed()` for Reproducible Tests

Seed the factory_boy RNG so that `Faker` values, `FuzzyChoice` picks, and
all other random declarations produce the same output on every run.

```python
import factory

# Module-level or in a fixture:
factory.random.reseed_random(12345)

# Verify:
u1 = UserFactory()
factory.random.reseed_random(12345)
u2 = UserFactory()
assert u1.first_name == u2.first_name   # always True

# Save/restore state around a block:
state = factory.random.get_random_state()
# ... run test code ...
factory.random.set_random_state(state)
```

In pytest, use an autouse fixture:

```python
# conftest.py
import pytest, factory

@pytest.fixture(autouse=True)
def reset_factory_seed():
    factory.random.reseed_random(42)
    yield
    # seed is reset before every test automatically
```

---

## pytest-factoryboy Integration

[pytest-factoryboy](https://pytest-factoryboy.readthedocs.io/) turns
factories into pytest fixtures with automatic dependency resolution.

### Installation

```bash
pip install pytest-factoryboy
```

### `register()` — expose a factory as fixtures

```python
# conftest.py
import pytest
from pytest_factoryboy import register
from myapp.factories import AuthorFactory, BookFactory, ArticleFactory

# Registers two fixtures per factory:
#   author_factory  → the factory class
#   author          → an Author instance
register(AuthorFactory)
register(BookFactory)

# Custom fixture name:
register(AuthorFactory, "second_author")
```

Decorator form:

```python
from pytest_factoryboy import register
import factory

@register
class TagFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = "myapp.Tag"
    name = factory.Sequence(lambda n: f"tag-{n}")
```

### Using auto-registered fixtures

```python
# tests/test_models.py

def test_author_created(author):
    assert author.pk is not None

def test_book_has_author(book):
    # book fixture auto-creates an author too (SubFactory dependency resolved)
    assert book.author is not None

def test_factory_fixture(author_factory):
    custom_author = author_factory(first_name="Tolkien")
    assert custom_author.first_name == "Tolkien"
```

### Overriding field values via `parametrize`

pytest-factoryboy exposes each factory attribute as an overridable fixture
named `modelname__fieldname`.

```python
@pytest.mark.parametrize("author__first_name", ["Agatha"])
def test_author_name(author):
    assert author.first_name == "Agatha"

@pytest.mark.parametrize("book__title", ["Dune"])
@pytest.mark.parametrize("author__first_name", ["Frank"])
def test_book_and_author(book):
    assert book.title == "Dune"
    assert book.author.first_name == "Frank"
```

### `LazyFixture` — inject other fixtures as factory arguments

```python
from pytest_factoryboy import register, LazyFixture

register(AuthorFactory)
register(BookFactory)
register(AuthorFactory, "another_author")

@pytest.mark.parametrize(
    "book__author",
    [LazyFixture("another_author")]   # use the "another_author" fixture
)
def test_book_uses_other_author(book, another_author):
    assert book.author == another_author

# Callable form:
@pytest.mark.parametrize(
    "book__author",
    [LazyFixture(lambda another_author: another_author)]
)
def test_lazy_callable(book, another_author):
    assert book.author == another_author
```

### Partial specialization — multiple fixture flavors

```python
# conftest.py
register(AuthorFactory, "male_author",   gender="M", first_name="John")
register(AuthorFactory, "female_author", gender="F")

@pytest.fixture
def female_author__first_name():
    return "Jane"

@pytest.mark.parametrize("male_author__age", [35])
def test_two_authors(male_author, female_author):
    assert male_author.first_name == "John"
    assert male_author.age == 35
    assert female_author.first_name == "Jane"
    assert female_author.gender == "F"
```

### Complete `conftest.py` example

```python
# conftest.py
import pytest
import factory
from pytest_factoryboy import register, LazyFixture
from factory.django import DjangoModelFactory
from myapp.models import User, Article, Tag

# --- Factories ---

class UserFactory(DjangoModelFactory):
    class Meta:
        model = User
        django_get_or_create = ("username",)

    username   = factory.Sequence(lambda n: f"user_{n}")
    email      = factory.LazyAttribute(lambda o: f"{o.username}@example.com")
    password   = factory.PostGenerationMethodCall("set_password", "pass")
    is_active  = True

    @factory.post_generation
    def groups(obj, create, extracted, **kwargs):
        if create and extracted:
            obj.groups.set(extracted)

class TagFactory(DjangoModelFactory):
    class Meta:
        model = Tag
    name = factory.Sequence(lambda n: f"tag-{n}")

class ArticleFactory(DjangoModelFactory):
    class Meta:
        model = Article

    title  = factory.Faker("sentence", nb_words=6)
    body   = factory.Faker("text")
    author = factory.SubFactory(UserFactory)

    @factory.post_generation
    def tags(obj, create, extracted, **kwargs):
        if not create:
            return
        if extracted:
            obj.tags.set(extracted)

# --- Register fixtures ---
register(UserFactory)
register(TagFactory)
register(ArticleFactory)
register(UserFactory, "admin_user", is_staff=True, is_superuser=True)

# --- Seed randomness for reproducibility ---
@pytest.fixture(autouse=True)
def reset_factory_seed():
    factory.random.reseed_random(0)
```

---

## Factory for Celery Task Payloads

Generate realistic task argument dictionaries for unit-testing Celery tasks
without touching the broker.

```python
import factory, datetime
from myapp.tasks import send_invoice_email, process_payment

class InvoicePayloadFactory(factory.Factory):
    """Builds the kwargs dict that send_invoice_email expects."""

    class Meta:
        model = dict   # factory.Factory works for plain dicts

    invoice_id = factory.Sequence(lambda n: n + 1000)
    user_email = factory.Faker("email")
    amount     = factory.Faker("pydecimal", left_digits=4, right_digits=2, positive=True)
    due_date   = factory.Faker("future_date")
    currency   = factory.Iterator(["USD", "EUR", "GBP"])

class PaymentPayloadFactory(factory.Factory):
    class Meta:
        model = dict

    order_id      = factory.Sequence(lambda n: f"ORD-{n:06d}")
    amount_cents  = factory.Faker("pyint", min_value=100, max_value=100000)
    payment_method = factory.FuzzyChoice(["card", "bank", "crypto"])

# --- Test ---

def test_send_invoice_email_task():
    payload = InvoicePayloadFactory()
    result = send_invoice_email.apply(kwargs=payload)
    assert result.successful()

def test_process_payment_retry(mocker):
    mocker.patch("myapp.payment_gateway.charge", side_effect=TimeoutError)
    payload = PaymentPayloadFactory()
    result = process_payment.apply(kwargs=payload)
    assert result.state == "FAILURE"
```

---

## Factory for GraphQL Mutation Inputs

```python
import factory

class CreateUserInputFactory(factory.Factory):
    class Meta:
        model = dict

    username  = factory.Faker("user_name")
    email     = factory.Faker("email")
    firstName = factory.Faker("first_name")
    lastName  = factory.Faker("last_name")
    password  = "TestPass123!"
    role      = factory.Iterator(["VIEWER", "EDITOR", "ADMIN"])

class UpdateArticleInputFactory(factory.Factory):
    class Meta:
        model = dict

    id      = factory.Sequence(lambda n: str(n + 1))
    title   = factory.Faker("sentence", nb_words=6)
    content = factory.Faker("text", max_nb_chars=500)
    tags    = factory.List([factory.Faker("word") for _ in range(3)])

# In a GraphQL test:
def test_create_user_mutation(graphql_client):
    input_data = CreateUserInputFactory()
    response = graphql_client.execute(
        CREATE_USER_MUTATION,
        variables={"input": input_data}
    )
    assert response["data"]["createUser"]["username"] == input_data["username"]
```

---

## Custom Declarations

Subclass `factory.declarations.BaseDeclaration` to create reusable custom
declaration types.

```python
import factory
from factory.declarations import BaseDeclaration

class ChoiceDeclaration(BaseDeclaration):
    """Picks a random element from a list of choices (like FuzzyChoice, but
    using factory_boy's seeded RNG for reproducibility)."""

    def __init__(self, choices):
        super().__init__()
        self.choices = list(choices)

    def evaluate(self, instance, step, extra):
        return factory.random.randgen.choice(self.choices)


class UpperCaseDeclaration(BaseDeclaration):
    """Wraps another declaration and upper-cases the result."""

    def __init__(self, declaration):
        super().__init__()
        self.declaration = declaration

    def evaluate(self, instance, step, extra):
        value = self.declaration.evaluate(instance, step, extra)
        return str(value).upper()


# Usage in a factory:
class ProductFactory(factory.Factory):
    class Meta:
        model = Product

    status   = ChoiceDeclaration(["active", "inactive", "pending"])
    sku_code = UpperCaseDeclaration(factory.Faker("bothify", text="??-####"))
```

---

## Factory `_create` Override for Read-Only / API-backed Models

When your model does not use the standard ORM `save()` / `objects.create()`
pattern, override `_create` on the factory.

```python
import factory

class APIUserFactory(factory.Factory):
    class Meta:
        model = dict   # or a dataclass / NamedTuple

    username = factory.Faker("user_name")
    email    = factory.Faker("email")

    @classmethod
    def _create(cls, model_class, *args, **kwargs):
        """Send a POST to an external API instead of hitting the database."""
        import httpx
        resp = httpx.post("https://api.example.com/users", json=kwargs)
        resp.raise_for_status()
        return resp.json()

class ReadOnlyModelFactory(factory.Factory):
    """For models that are populated by migrations / fixtures, not create()."""

    class Meta:
        model = ReadOnlyModel

    @classmethod
    def _create(cls, model_class, *args, **kwargs):
        # Fetch an existing instance instead of creating a new one
        return model_class.objects.filter(**kwargs).first() or \
               model_class.objects.order_by("?").first()
```

Custom `_build` for non-standard constructors:

```python
class ImmutableRecordFactory(factory.Factory):
    class Meta:
        model = ImmutableRecord   # a frozen dataclass / NamedTuple

    @classmethod
    def _build(cls, model_class, *args, **kwargs):
        return model_class(**kwargs)   # already fine for dataclasses

class NamedTupleFactory(factory.Factory):
    class Meta:
        model = Point  # collections.namedtuple

    x = factory.Faker("pyfloat", left_digits=2, right_digits=4)
    y = factory.Faker("pyfloat", left_digits=2, right_digits=4)

    @classmethod
    def _build(cls, model_class, *args, **kwargs):
        return model_class(x=kwargs["x"], y=kwargs["y"])
```

---

## Performance: `create_batch` vs Multiple `create` Calls

`create_batch(n)` is idiomatic and slightly preferred over a Python loop, but
the main cost driver is the number of database round-trips, not factory
invocation overhead.

```python
# Preferred — single expression, same DB round-trips
users = UserFactory.create_batch(100)

# Equivalent but verbose
users = [UserFactory.create() for _ in range(100)]

# For heavy INSERT loads, use bulk_create manually:
from myapp.models import User

objs = UserFactory.build_batch(10_000)   # no DB hits
User.objects.bulk_create(objs, batch_size=500)

# When using DjangoModelFactory, you can override _create to bulk-insert:
class FastUserFactory(DjangoModelFactory):
    class Meta:
        model = User

    username = factory.Sequence(lambda n: f"user_{n}")

    @classmethod
    def _create(cls, model_class, *args, **kwargs):
        # Fall back to standard create; override for bulk scenarios
        return super()._create(model_class, *args, **kwargs)
```

For read-heavy test suites, prefer `build_batch` (no DB) over `create_batch`
whenever persistence is not required by the assertion under test.

---

## `_adjust_kwargs` — Late Field Tuning

Called after all declarations are resolved and before the model constructor.
Use it to transform or validate the resolved kwargs.

```python
class UserFactory(factory.Factory):
    class Meta:
        model = User

    last_name = factory.Faker("last_name")

    @classmethod
    def _adjust_kwargs(cls, **kwargs):
        # Enforce uppercase last names in the model
        kwargs["last_name"] = kwargs["last_name"].upper()
        return kwargs
```

---

## Summary: Patterns at a Glance

| Pattern | Mechanism | Use Case |
|---|---|---|
| Unique fields | `Sequence` | PK-like values, usernames, emails |
| Computed fields | `LazyAttribute` | Derived from sibling fields |
| Realistic data | `Faker` | Human-readable test data |
| Nested models | `SubFactory` | ForeignKey relations |
| Reverse relations | `RelatedFactory` / `RelatedFactoryList` | Reverse FK / reverse O2O |
| M2M | `@post_generation` | Django ManyToManyField |
| Toggle groups | `Trait` | Named object states (shipped, cancelled) |
| Helper values | `Meta.exclude` | Intermediate computations |
| Reproducibility | `reseed_random(n)` | Deterministic test suites |
| pytest DI | `pytest-factoryboy register()` | Fixture-based test setup |
| Task payloads | `Factory(model=dict)` | Celery / GraphQL inputs |
| Custom persistence | `_create` override | API clients, read-only models |
