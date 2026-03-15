# factory_boy — Basics

> **Source:** https://factoryboy.readthedocs.io/en/stable/

factory_boy is a fixture replacement library for Python. It provides a
declarative syntax for defining test object factories that generate realistic,
varied instances of your data models, replacing fragile hand-written setup
code with concise, composable declarations.

---

## Installation

```bash
pip install factory-boy
# Faker is bundled as a dependency — no separate install needed.
```

For Django integration nothing extra is required; the `factory.django`
sub-package ships with the main package.

---

## The Base Factory

Every factory subclasses `factory.Factory` and declares its target class in
an inner `Meta` class.

```python
import factory
from myapp.models import User

class UserFactory(factory.Factory):
    class Meta:
        model = User          # the class that will be instantiated

    first_name = "John"
    last_name  = "Doe"
    email      = "john.doe@example.com"
```

Calling `UserFactory()` is equivalent to `User(first_name="John", ...)`.
Static values are shared across every instance unless you override them.

---

## Field Declarations

### Static / literal values

The simplest declaration — a plain Python value used as-is for every object.

```python
class ArticleFactory(factory.Factory):
    class Meta:
        model = Article

    status = "draft"
    views  = 0
```

---

### `factory.Sequence` — sequential, unique values

Use when a field must be unique (e.g. usernames, email addresses, order IDs).
The counter starts at 0 and increments automatically.

```python
class UserFactory(factory.Factory):
    class Meta:
        model = User

    username = factory.Sequence(lambda n: f"user_{n}")
    email    = factory.Sequence(lambda n: f"user_{n}@example.com")

# UserFactory() → username="user_0"
# UserFactory() → username="user_1"
```

Decorator form:

```python
class UserFactory(factory.Factory):
    @factory.sequence
    def username(n):
        return f"user_{n:04d}"
```

Reset the counter explicitly:

```python
UserFactory.reset_sequence()        # resets to 0
UserFactory.reset_sequence(10)      # resets to 10
UserFactory.reset_sequence(force=True)  # forces reset even on subclasses
```

Force a specific counter value for one call:

```python
UserFactory(__sequence=42)   # this instance uses n=42; global counter unchanged
```

---

### `factory.LazyFunction` — no-arg callable

Call a zero-argument function to compute a fresh value each time.

```python
import datetime, factory

class LogEntryFactory(factory.Factory):
    class Meta:
        model = LogEntry

    # datetime.now() called fresh on every build
    timestamp = factory.LazyFunction(datetime.datetime.now)

    # Copy a mutable default without sharing state
    allowed_ips = factory.LazyFunction(lambda: list(DEFAULT_IPS))
```

---

### `factory.LazyAttribute` — computed from sibling fields

Receives the partially-built object (a `Resolver`) and returns the value.

```python
class UserFactory(factory.Factory):
    class Meta:
        model = User

    first_name = "John"
    last_name  = "Doe"
    email      = factory.LazyAttribute(
        lambda o: f"{o.first_name.lower()}.{o.last_name.lower()}@example.com"
    )
```

Decorator form (useful for multi-line logic):

```python
class UserFactory(factory.Factory):
    class Meta:
        model = User

    first_name = "Ève"

    @factory.lazy_attribute
    def email(self):
        import unicodedata
        clean = unicodedata.normalize("NFKD", self.first_name)
        return f"{clean.lower()}@example.com"
```

`LazyAttributeSequence` combines both — receives `(object, n)`:

```python
class UserFactory(factory.Factory):
    login = "john"
    email = factory.LazyAttributeSequence(
        lambda o, n: f"{o.login}+{n}@example.com"
    )
```

---

### `factory.Faker` — realistic data via the Faker library

Wraps the [Faker](https://faker.readthedocs.io/) library. The first argument
is the provider method name; any remaining arguments are forwarded.

```python
import factory

class PersonFactory(factory.Factory):
    class Meta:
        model = Person

    # Basic providers
    first_name   = factory.Faker("first_name")
    last_name    = factory.Faker("last_name")
    email        = factory.Faker("email")
    phone_number = factory.Faker("phone_number")

    # Address providers
    street  = factory.Faker("street_address")
    city    = factory.Faker("city")
    country = factory.Faker("country_code")

    # Internet providers
    username  = factory.Faker("user_name")
    url       = factory.Faker("url")
    ipv4      = factory.Faker("ipv4_private")

    # Date / time providers
    birthdate  = factory.Faker("date_of_birth", minimum_age=18, maximum_age=90)
    created_at = factory.Faker("date_time_this_decade")
    arrival    = factory.Faker(
        "date_between_dates",
        date_start=datetime.date(2024, 1, 1),
        date_end=datetime.date(2024, 12, 31),
    )

    # Text providers
    bio       = factory.Faker("paragraph")
    slug      = factory.Faker("slug")
    uuid      = factory.Faker("uuid4")
    color_hex = factory.Faker("color")

    # Numbers
    price = factory.Faker("pydecimal", left_digits=4, right_digits=2, positive=True)
    score = factory.Faker("pyint", min_value=0, max_value=100)

    # Locale-specific
    french_name = factory.Faker("name", locale="fr_FR")
```

Override the default locale globally or temporarily:

```python
# Permanent override for a factory attribute
class FrenchUserFactory(factory.Factory):
    class Meta:
        model = User
    name = factory.Faker("name", locale="fr_FR")

# Temporary override via context manager
with factory.Faker.override_default_locale("ja_JP"):
    user = UserFactory()

# Register a custom Faker provider
from my_providers import ColorProvider
factory.Faker.add_provider(ColorProvider)
```

---

### `factory.SubFactory` — nested / related objects

Instantiates another factory as an attribute value. The build strategy of the
parent propagates to the sub-factory (i.e., `build` stays `build`,
`create` stays `create`).

```python
class AddressFactory(factory.Factory):
    class Meta:
        model = Address

    street = factory.Faker("street_address")
    city   = factory.Faker("city")

class UserFactory(factory.Factory):
    class Meta:
        model = User

    name    = factory.Faker("name")
    address = factory.SubFactory(AddressFactory)
```

Override sub-factory fields using `__` double-underscore notation:

```python
user = UserFactory(address__city="Berlin")
```

Provide a pre-built object to skip sub-factory execution entirely:

```python
berlin = Address(city="Berlin")
user = UserFactory(address=berlin)
```

Handle circular imports with a dotted string path:

```python
group = factory.SubFactory("myapp.factories.GroupFactory")
```

Pass constant overrides directly to SubFactory:

```python
class CompanyFactory(factory.Factory):
    class Meta:
        model = Company

    owner = factory.SubFactory(UserFactory, first_name="Jack")
```

---

### `factory.RelatedFactory` — reverse / post-creation relations

Creates a *related* object **after** the main object has been built, linking
the two by setting `factory_related_name` on the child factory.

```python
class CityFactory(factory.Factory):
    class Meta:
        model = City

    name       = factory.Faker("city")
    capital_of = None   # will be set by CountryFactory

class CountryFactory(factory.Factory):
    class Meta:
        model = Country

    name         = factory.Faker("country")
    capital_city = factory.RelatedFactory(
        CityFactory,
        factory_related_name="capital_of",  # sets City.capital_of = <Country>
        name="Paris",
    )
```

Override via double-underscore:

```python
CountryFactory(capital_city__name="London")
```

Disable the related object entirely by passing an explicit value:

```python
CountryFactory(capital_city=None)
```

Generate *multiple* related objects with `RelatedFactoryList`:

```python
class AuthorFactory(factory.Factory):
    class Meta:
        model = Author

    books = factory.RelatedFactoryList(
        BookFactory,
        factory_related_name="author",
        size=3,                              # generate 3 books
    )

# Dynamic size:
books = factory.RelatedFactoryList(BookFactory, size=lambda: random.randint(1, 5))
```

---

### `factory.SelfAttribute` — reference sibling or parent fields

Access another declared field on the same factory (or on a parent factory
via `..` traversal).

```python
class UserFactory(factory.Factory):
    class Meta:
        model = User

    birthdate   = factory.Faker("date_of_birth")
    birth_month = factory.SelfAttribute("birthdate.month")   # attribute chaining

class CompanyFactory(factory.Factory):
    class Meta:
        model = Company

    country = factory.SubFactory(CountryFactory)
    owner   = factory.SubFactory(
        UserFactory,
        # Access parent factory's field with ".."
        language=factory.SelfAttribute("..country.language"),
    )
```

Equivalent `LazyAttribute` form:

```python
language = factory.LazyAttribute(lambda user: user.factory_parent.country.language)
```

---

### `factory.Iterator` — cycle through a sequence

Cycles through an iterable indefinitely. Useful for distributing values like
languages, roles, or statuses evenly.

```python
class UserFactory(factory.Factory):
    class Meta:
        model = User

    lang = factory.Iterator(["en", "fr", "es", "de", "it"])
    role = factory.Iterator(
        User.ROLE_CHOICES,
        getter=lambda c: c[0],   # extract value from (value, label) tuples
    )
```

Reset the cycle:

```python
UserFactory.lang.reset()
```

Decorator form (generator-based, evaluated lazily):

```python
class UserFactory(factory.Factory):
    @factory.iterator
    def tier():
        with open("test/data/tiers.txt") as f:
            for line in f:
                yield line.strip()
```

---

### `factory.Dict` — dict-typed fields with declarations

Values inside the dict can themselves be any factory declaration.

```python
class UserFactory(factory.Factory):
    class Meta:
        model = User

    is_staff = False
    permissions = factory.Dict({
        "can_edit":   True,
        "can_delete": False,
        "can_publish": factory.Iterator([True, False]),
        "is_admin":   factory.SelfAttribute("..is_staff"),  # ".." = factory root
    })

# Override individual dict keys with "__":
user = UserFactory(permissions__can_publish=True)
```

---

### `factory.List` — list-typed fields with declarations

Individual list elements can be overridden by index.

```python
class UserFactory(factory.Factory):
    class Meta:
        model = User

    roles = factory.List(["viewer", "commenter", "editor"])

# Override by zero-based index:
user = UserFactory(roles__0="admin")   # replaces "viewer" with "admin"
```

---

### `factory.Maybe` — conditional declarations

Applies one of two declarations depending on a boolean field.

```python
import datetime, factory
from django.utils import timezone

class UserFactory(factory.Factory):
    class Meta:
        model = User

    is_active        = True
    deactivation_date = factory.Maybe(
        "is_active",
        yes_declaration=None,                          # active → no deactivation date
        no_declaration=factory.LazyFunction(
            lambda: timezone.now() - datetime.timedelta(days=30)
        ),
    )

# Create an inactive user:
inactive_user = UserFactory(is_active=False)
# inactive_user.deactivation_date is set automatically
```

---

### `@factory.post_generation` — post-creation hooks

Runs **after** the object has been created. Receives
`(obj, create, extracted, **kwargs)`:

- `obj` — the generated object
- `create` — `True` if `create()` strategy was used, `False` for `build()`
- `extracted` — value passed explicitly at call time (or `None`)
- `**kwargs` — any `field__subkey=value` overrides

```python
class UserFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = User

    username = factory.Faker("user_name")

    @factory.post_generation
    def groups(obj, create, extracted, **kwargs):
        if not create:
            return   # build() strategy — skip DB operations
        if extracted:
            for group in extracted:
                obj.groups.add(group)

# Usage:
admin_group = Group.objects.get(name="admin")
user = UserFactory.create(groups=[admin_group])
```

`PostGeneration` function form:

```python
class UserFactory(factory.Factory):
    make_mbox = factory.PostGeneration(
        lambda obj, create, extracted, **kwargs:
            os.makedirs(extracted or f"/tmp/mbox/{obj.username}", exist_ok=True)
    )
```

`PostGenerationMethodCall` — calls a method on the generated object:

```python
class UserFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = User

    username = factory.Faker("user_name")
    # Calls obj.set_password("defaultpass") after creation
    password = factory.PostGenerationMethodCall("set_password", "defaultpass")

# Override:
user = UserFactory(password="secret123")
```

---

## Traits

A `Trait` is a reusable bundle of field overrides that can be toggled with a
single boolean parameter. Traits live inside a `Params` inner class.

```python
class OrderFactory(factory.Factory):
    class Meta:
        model = Order

    state      = "pending"
    shipped_on = None
    shipped_by = None

    class Params:
        shipped = factory.Trait(
            state      = "shipped",
            shipped_on = factory.LazyFunction(datetime.date.today),
            shipped_by = factory.SubFactory(EmployeeFactory),
        )
        cancelled = factory.Trait(
            state = "cancelled",
        )

# Usage:
order = OrderFactory(shipped=True)
```

Traits can reference each other (chaining):

```python
class Params:
    shipped = factory.Trait(state="shipped", shipped_on=datetime.date.today())
    received = factory.Trait(
        shipped=True,       # activates the shipped trait first
        state="received",
        received_on=datetime.date.today(),
    )
```

Traits can use `Meta.exclude` to prevent helper fields from being passed to
the model constructor:

```python
class UserFactory(factory.Factory):
    class Meta:
        model = User
        exclude = ("now",)

    now        = factory.LazyFunction(datetime.datetime.utcnow)
    created_at = factory.LazyAttribute(lambda o: o.now - datetime.timedelta(hours=1))
    updated_at = factory.LazyAttribute(lambda o: o.now - datetime.timedelta(minutes=5))
```

---

## Django Integration — `DjangoModelFactory`

`DjangoModelFactory` replaces `factory.Factory` for Django ORM models.
Its `create()` calls `Model.objects.create()` automatically.

```python
import factory
from factory.django import DjangoModelFactory
from myapp.models import Article, Tag

class TagFactory(DjangoModelFactory):
    class Meta:
        model = Tag

    name = factory.Sequence(lambda n: f"tag-{n}")

class ArticleFactory(DjangoModelFactory):
    class Meta:
        model = Article
        django_get_or_create = ("slug",)   # use get_or_create instead of create

    title  = factory.Faker("sentence", nb_words=6)
    slug   = factory.LazyAttribute(lambda o: o.title.lower().replace(" ", "-"))
    author = factory.SubFactory("myapp.factories.UserFactory")

    @factory.post_generation
    def tags(obj, create, extracted, **kwargs):
        if not create:
            return
        if extracted:
            obj.tags.set(extracted)

# Usage:
article = ArticleFactory()
article_with_tags = ArticleFactory(tags=[TagFactory(), TagFactory()])
```

Key `Meta` options for `DjangoModelFactory`:

| Option | Description |
|---|---|
| `model` | The Django model class or `'app.Model'` string |
| `database` | Named DB alias (default: `"default"`) |
| `django_get_or_create` | Tuple of field names for `get_or_create` lookup |
| `skip_postgeneration_save` | Avoid the extra `.save()` after post-generation hooks |

Handling Django passwords:

```python
from factory.django import DjangoModelFactory
from django.contrib.auth.hashers import make_password

class UserFactory(DjangoModelFactory):
    class Meta:
        model = "auth.User"

    username = factory.Faker("user_name")
    password = factory.PostGenerationMethodCall("set_password", "testpass123")
    email    = factory.Faker("email")
```

Suppressing signals:

```python
from django.db.models import signals

@factory.django.mute_signals(signals.pre_save, signals.post_save)
class ProfileFactory(DjangoModelFactory):
    class Meta:
        model = Profile
    # signals are silenced during factory operations
```

---

## SQLAlchemy Integration — `SQLAlchemyModelFactory`

```python
import factory
from factory.alchemy import SQLAlchemyModelFactory
from myapp import models
from myapp.test import common   # provides a scoped_session

class UserFactory(SQLAlchemyModelFactory):
    class Meta:
        model          = models.User
        sqlalchemy_session             = common.Session      # scoped session
        sqlalchemy_session_persistence = "commit"            # flush/commit/None

    name  = factory.Sequence(lambda n: f"User {n}")
    email = factory.LazyAttribute(lambda o: f"{o.name.lower()}@example.com")
```

Session factory approach (preferred for concurrency):

```python
class ArticleFactory(SQLAlchemyModelFactory):
    class Meta:
        model                    = models.Article
        sqlalchemy_session_factory = lambda: Session()

    title = factory.Faker("sentence")
```

Test teardown:

```python
def teardown_function():
    common.Session.rollback()
    common.Session.remove()
```

---

## Build Strategies

| Method | Description |
|---|---|
| `UserFactory.build()` | Instantiates model without saving (`__init__` only) |
| `UserFactory.create()` | Instantiates **and** persists (calls `save()` / `objects.create()`) |
| `UserFactory.stub()` | Returns a `StubObject` — plain attribute bag, no model class needed |
| `UserFactory.build_batch(n)` | Returns a list of `n` built instances |
| `UserFactory.create_batch(n)` | Returns a list of `n` created/saved instances |
| `UserFactory.stub_batch(n)` | Returns a list of `n` stubs |

```python
# Single instances
user  = UserFactory.build()    # not saved
user  = UserFactory.create()   # saved
stub  = UserFactory.stub()     # StubObject, no model

# Batches
users = UserFactory.build_batch(3)
users = UserFactory.create_batch(3)

# Override fields for all in a batch:
users = UserFactory.create_batch(5, is_active=False)

# Module-level shortcuts
from factory import build, create, stub
u = build(User, first_name="Alice")
u = create(User, first_name="Alice")
```

`build()` vs `create()` semantics:

- `build()` — only calls `Model.__init__()`. Sub-factories also call `build()`.
  No DB interaction. `@post_generation` hooks run but `create=False`.
- `create()` — calls `Model._create()` (default: `objects.create()`). Sub-factories
  also use `create()`. `@post_generation` hooks run with `create=True`.

---

## Fuzzy Attributes

The `factory.fuzzy` module provides random-value generators.
Most can be replaced by `factory.Faker`, but they remain useful for ranges.

> **Note:** Fuzzy attributes use `factory.random.randgen` (a seeded RNG) for
> reproducibility. Set the seed with `factory.random.reseed_random(42)`.

```python
import factory.fuzzy as fuzzy
import datetime, decimal
```

### `FuzzyChoice`

Picks a random item from an iterable. The iterable is evaluated lazily
(safe to pass a Django queryset).

```python
class ArticleFactory(factory.Factory):
    status = fuzzy.FuzzyChoice(["draft", "published", "archived"])
    author = fuzzy.FuzzyChoice(User.objects.filter(is_active=True))
```

### `FuzzyInteger`

```python
class ProductFactory(factory.Factory):
    quantity = fuzzy.FuzzyInteger(1, 100)          # inclusive range
    rating   = fuzzy.FuzzyInteger(0, 10, step=2)   # even numbers only
```

### `FuzzyFloat`

```python
class MeasurementFactory(factory.Factory):
    value = fuzzy.FuzzyFloat(0.0, 9.99)
```

### `FuzzyDecimal`

```python
class OrderFactory(factory.Factory):
    total = fuzzy.FuzzyDecimal(0.01, 9999.99, precision=2)
```

### `FuzzyDate`

```python
class EventFactory(factory.Factory):
    date = fuzzy.FuzzyDate(
        datetime.date(2020, 1, 1),
        datetime.date(2025, 12, 31),   # optional; defaults to today
    )
```

### `FuzzyDateTime`

Produces timezone-aware datetimes. Supports `force_*` arguments to pin
specific components.

```python
from django.utils.timezone import utc

class EventFactory(factory.Factory):
    starts_at = fuzzy.FuzzyDateTime(
        datetime.datetime(2024, 1, 1, tzinfo=utc),
        datetime.datetime(2024, 12, 31, 23, 59, tzinfo=utc),
        force_hour=9,       # always at 9 AM
    )
```

### `FuzzyNaiveDateTime`

Like `FuzzyDateTime` but produces timezone-naive datetimes.

### `FuzzyText`

```python
class TagFactory(factory.Factory):
    name = fuzzy.FuzzyText(length=8, prefix="tag_", chars="abcdefghijklmnopqrstuvwxyz0123456789")
```

### `FuzzyAttribute`

Accepts any callable — most flexible option.

```python
import random

class ProductFactory(factory.Factory):
    weight_kg = fuzzy.FuzzyAttribute(lambda: round(random.uniform(0.1, 50.0), 2))
```

### Randomness and reproducibility

```python
# Seed for reproducible tests:
factory.random.reseed_random(42)

# Save and restore state:
state = factory.random.get_random_state()
# ... run tests ...
factory.random.set_random_state(state)
```

---

## `Meta` Options Reference

| Option | Description |
|---|---|
| `model` | Target class to instantiate |
| `abstract` | If `True`, factory is not used directly (no model required) |
| `exclude` | Fields passed to declarations but NOT forwarded to the model |
| `rename` | Dict mapping factory field names → model parameter names |
| `inline_args` | Tuple of field names to pass as positional args to `__init__` |
| `strategy` | Default strategy: `factory.BUILD_STRATEGY` / `factory.CREATE_STRATEGY` |
| `django_get_or_create` | (DjangoModelFactory) Fields for get-or-create |
| `database` | (DjangoModelFactory) Named DB alias |
| `sqlalchemy_session` | (SQLAlchemyModelFactory) Session object |
| `sqlalchemy_session_persistence` | (SQLAlchemyModelFactory) `None`, `"flush"`, or `"commit"` |

---

## Debug Helper

```python
with factory.debug():
    user = UserFactory.create()
# Prints detailed factory resolution log to stderr
```
