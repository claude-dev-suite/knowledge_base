# Pydantic - Models

> Official Documentation: https://docs.pydantic.dev/latest/concepts/models/

## Overview

Pydantic models are Python classes that inherit from `BaseModel` and define fields as annotated attributes. They provide automatic data parsing, validation, and serialization. Validation runs on instantiation and raises a `ValidationError` if data is invalid.

---

## Table of Contents

1. [Defining a Model](#defining-a-model)
2. [Field Definition with Field()](#field-definition-with-field)
3. [Model Configuration (model_config)](#model-configuration-model_config)
4. [Field Validators](#field-validators)
5. [Model Validators](#model-validators)
6. [Computed Fields](#computed-fields)
7. [Nested Models](#nested-models)
8. [Model Methods](#model-methods)
9. [Serialization](#serialization)
10. [Advanced Patterns](#advanced-patterns)

---

## Defining a Model

```python
from pydantic import BaseModel

class User(BaseModel):
    id: int
    name: str
    email: str
    age: int | None = None   # Optional field with default None
    is_active: bool = True   # Field with default value

# Instantiation validates data
user = User(id=1, name="Alice", email="alice@example.com")
print(user)
# id=1 name='Alice' email='alice@example.com' age=None is_active=True

# Type coercion (lax mode by default)
user2 = User(id="42", name="Bob", email="bob@example.com")
assert user2.id == 42  # "42" coerced to int
```

### Accessing Fields

```python
user.name           # "Alice"
user.model_fields   # dict of field names to FieldInfo objects
user.__dict__       # {"id": 1, "name": "Alice", ...}
```

---

## Field Definition with Field()

`Field()` provides rich metadata and constraints for model fields.

```python
from pydantic import BaseModel, Field
from uuid import uuid4

class Product(BaseModel):
    id: str = Field(default_factory=lambda: uuid4().hex)
    name: str = Field(
        ...,                    # ... means required (no default)
        min_length=1,
        max_length=100,
        description="Product display name",
        title="Product Name",
        examples=["Widget Pro", "Gadget Plus"]
    )
    price: float = Field(gt=0, le=1_000_000, description="Price in USD")
    quantity: int = Field(default=0, ge=0)
    sku: str = Field(alias="product_sku")   # Input uses "product_sku", field named "sku"
    tags: list[str] = Field(default_factory=list)
```

### Field Parameters Reference

| Parameter | Type | Description |
|-----------|------|-------------|
| `default` | Any | Static default value |
| `default_factory` | callable | Function called to generate default |
| `alias` | str | Alias for both validation and serialization |
| `validation_alias` | str/AliasPath/AliasChoices | Alias only for validation input |
| `serialization_alias` | str | Alias only for serialization output |
| `title` | str | JSON Schema title |
| `description` | str | JSON Schema description |
| `examples` | list | JSON Schema examples |
| `gt` | number | Greater than (exclusive) |
| `ge` | number | Greater than or equal (inclusive) |
| `lt` | number | Less than (exclusive) |
| `le` | number | Less than or equal (inclusive) |
| `multiple_of` | number | Must be multiple of this value |
| `min_length` | int | Minimum string/collection length |
| `max_length` | int | Maximum string/collection length |
| `pattern` | str | Regex pattern for strings |
| `strict` | bool | Enable strict mode for this field |
| `frozen` | bool | Prevent reassignment after instantiation |
| `exclude` | bool | Exclude from `model_dump()` |
| `repr` | bool | Include in `__repr__` (default True) |
| `validate_default` | bool | Validate the default value |
| `discriminator` | str | Union discriminator field name |
| `deprecated` | bool/str | Mark field as deprecated |

### Alias Patterns

```python
from pydantic import BaseModel, Field, AliasPath, AliasChoices

class Response(BaseModel):
    # Accept "firstName" or "first_name" in input
    first_name: str = Field(
        validation_alias=AliasChoices("firstName", "first_name")
    )
    # Access nested JSON: {"user": {"id": 123}}
    user_id: int = Field(validation_alias=AliasPath("user", "id"))
    # Output uses "fullName"
    full_name: str = Field(serialization_alias="fullName")

resp = Response.model_validate({"firstName": "Alice", "user": {"id": 1}, "full_name": "Alice Smith"})
print(resp.model_dump(by_alias=True))
# {"first_name": "Alice", "user_id": 1, "fullName": "Alice Smith"}
```

---

## Model Configuration (model_config)

```python
from pydantic import BaseModel, ConfigDict

class StrictUser(BaseModel):
    model_config = ConfigDict(
        # Validation behavior
        strict=False,              # Enable strict mode globally
        coerce_numbers_to_str=False,
        validate_default=False,    # Validate default values
        validate_assignment=True,  # Validate on field assignment (user.name = ...)
        revalidate_instances="never",  # "never", "always", "subclass-instances"

        # Extra fields
        extra="ignore",            # "ignore" (default), "forbid", "allow"

        # Serialization
        populate_by_name=True,     # Allow both field name and alias for input
        use_enum_values=True,      # Store enum values, not enum instances

        # JSON / ORM
        from_attributes=True,      # Enable ORM mode (read from object attributes)
        json_encoders={},          # Custom JSON encoders (deprecated in v2, use serializers)

        # Documentation
        title="User Model",
        str_max_length=500,
        str_min_length=1,
        str_strip_whitespace=True,
        str_to_lower=False,
        str_to_upper=False,

        # Performance
        frozen=False,              # Make model immutable (hashable)
    )

    name: str
    age: int
```

### Common Config Patterns

```python
# Immutable / hashable model
class ImmutablePoint(BaseModel):
    model_config = ConfigDict(frozen=True)
    x: float
    y: float

p = ImmutablePoint(x=1.0, y=2.0)
# p.x = 3.0  # Raises ValidationError

# ORM model (read from SQLAlchemy objects, etc.)
class UserSchema(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    name: str

# Usage with SQLAlchemy ORM object
# user_schema = UserSchema.model_validate(db_user)

# Forbid extra fields
class StrictSchema(BaseModel):
    model_config = ConfigDict(extra="forbid")
    name: str
    age: int

# StrictSchema(name="Alice", age=30, extra_field="x")  # Raises ValidationError
```

---

## Field Validators

Validators run custom logic for individual fields.

```python
from pydantic import BaseModel, field_validator, ValidationInfo

class User(BaseModel):
    name: str
    age: int
    email: str
    password: str
    password_confirm: str

    @field_validator("name")
    @classmethod
    def name_must_not_be_empty(cls, v: str) -> str:
        v = v.strip()
        if not v:
            raise ValueError("Name cannot be empty")
        return v.title()  # Return transformed value

    @field_validator("age")
    @classmethod
    def age_must_be_positive(cls, v: int) -> int:
        if v < 0:
            raise ValueError("Age must be non-negative")
        if v > 150:
            raise ValueError("Age seems unrealistic")
        return v

    @field_validator("email")
    @classmethod
    def email_must_be_valid(cls, v: str) -> str:
        if "@" not in v:
            raise ValueError("Invalid email format")
        return v.lower()

    # Validate multiple fields at once
    @field_validator("password", "password_confirm")
    @classmethod
    def password_min_length(cls, v: str) -> str:
        if len(v) < 8:
            raise ValueError("Password must be at least 8 characters")
        return v

    # Access previously validated fields via info
    @field_validator("password_confirm", mode="after")
    @classmethod
    def passwords_match(cls, v: str, info: ValidationInfo) -> str:
        if "password" in info.data and v != info.data["password"]:
            raise ValueError("Passwords do not match")
        return v
```

### Validator Modes

| Mode | When it runs | Input type |
|------|-------------|------------|
| `"before"` | Before Pydantic parsing | Raw input (any type) |
| `"after"` (default) | After Pydantic parsing | Already validated type |
| `"wrap"` | Wraps Pydantic validation | Raw input + handler |
| `"plain"` | Replaces Pydantic validation | Raw input (any type) |

```python
from pydantic import BaseModel, field_validator

class Model(BaseModel):
    tags: list[str]

    # Before validator: handle comma-separated strings
    @field_validator("tags", mode="before")
    @classmethod
    def parse_tags(cls, v):
        if isinstance(v, str):
            return [t.strip() for t in v.split(",")]
        return v

m = Model(tags="python, pydantic, fast")
# m.tags == ["python", "pydantic", "fast"]
```

---

## Model Validators

Model validators operate on the entire model data, useful for cross-field validation.

```python
from pydantic import BaseModel, model_validator
from typing_extensions import Self

class DateRange(BaseModel):
    start_date: str
    end_date: str
    label: str | None = None

    # Runs AFTER all fields are validated
    @model_validator(mode="after")
    def check_date_order(self) -> Self:
        if self.start_date > self.end_date:
            raise ValueError("start_date must be before end_date")
        if self.label is None:
            self.label = f"{self.start_date} to {self.end_date}"
        return self

    # Runs BEFORE individual field validation (receives raw dict)
    @model_validator(mode="before")
    @classmethod
    def preprocess_input(cls, data: dict) -> dict:
        if isinstance(data, dict) and "date_from" in data:
            data["start_date"] = data.pop("date_from")
        return data
```

### Wrap Model Validator

```python
import logging
from pydantic import BaseModel, model_validator, ValidationError
from pydantic import ModelWrapValidatorHandler
from typing_extensions import Self

class LoggedModel(BaseModel):
    name: str
    value: int

    @model_validator(mode="wrap")
    @classmethod
    def log_validation(cls, data, handler: ModelWrapValidatorHandler[Self]) -> Self:
        try:
            result = handler(data)
            logging.info("Validation succeeded for %s", cls.__name__)
            return result
        except ValidationError as e:
            logging.error("Validation failed: %s", e)
            raise
```

---

## Computed Fields

Computed fields are properties that appear in `model_dump()` and JSON schema:

```python
from pydantic import BaseModel, computed_field
from functools import cached_property
import math

class Circle(BaseModel):
    radius: float

    @computed_field
    @property
    def area(self) -> float:
        return math.pi * self.radius ** 2

    @computed_field
    @property
    def circumference(self) -> float:
        return 2 * math.pi * self.radius

class UserProfile(BaseModel):
    first_name: str
    last_name: str
    birth_year: int

    @computed_field
    @property
    def full_name(self) -> str:
        return f"{self.first_name} {self.last_name}"

    @computed_field
    @cached_property
    def age(self) -> int:
        from datetime import date
        return date.today().year - self.birth_year

c = Circle(radius=5.0)
print(c.model_dump())
# {"radius": 5.0, "area": 78.53..., "circumference": 31.41...}
```

---

## Nested Models

```python
from pydantic import BaseModel
from typing import Optional

class Address(BaseModel):
    street: str
    city: str
    country: str
    postal_code: str | None = None

class Company(BaseModel):
    name: str
    address: Address

class Person(BaseModel):
    name: str
    age: int
    address: Address
    employer: Optional[Company] = None
    tags: list[str] = []
    metadata: dict[str, str] = {}

# Nested dict is automatically coerced to nested model
person = Person(
    name="Alice",
    age=30,
    address={"street": "123 Main St", "city": "NYC", "country": "US"},
    employer={"name": "Acme Corp", "address": {"street": "1 Corp Ave", "city": "NYC", "country": "US"}},
    tags=["developer", "python"],
)

print(person.address.city)  # "NYC"
print(person.employer.name) # "Acme Corp"
```

---

## Model Methods

### Validation / Construction

```python
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    price: float

# From dict
item = Item.model_validate({"name": "Widget", "price": 9.99})

# From JSON string
item = Item.model_validate_json('{"name": "Widget", "price": 9.99}')

# From ORM object (requires from_attributes=True in config)
# item = Item.model_validate(db_item)

# Strict validation (no coercion)
try:
    item = Item.model_validate({"name": "Widget", "price": "9.99"}, strict=True)
except Exception as e:
    print(e)  # price must be float, not str

# Skip validation (use with care — no guarantee of correctness)
item = Item.model_construct(name="Widget", price=9.99)
```

### Serialization

```python
item = Item(name="Widget", price=9.99)

# To dict
d = item.model_dump()
# {"name": "Widget", "price": 9.99}

# To JSON string
j = item.model_dump_json()
# '{"name":"Widget","price":9.99}'

# Selective serialization
item.model_dump(include={"name"})
item.model_dump(exclude={"price"})
item.model_dump(exclude_none=True)
item.model_dump(exclude_unset=True)   # Only fields explicitly set
item.model_dump(by_alias=True)        # Use serialization aliases
item.model_dump(mode="json")          # JSON-compatible types (e.g., datetime → str)
```

### Copy

```python
# Shallow copy
item_copy = item.model_copy()

# Deep copy
item_deep = item.model_copy(deep=True)

# Copy with updates
item_updated = item.model_copy(update={"price": 19.99})
```

### Schema

```python
# JSON Schema
schema = Item.model_json_schema()
print(schema)
# {"title": "Item", "type": "object", "properties": {...}}

# Rebuild schema (for forward references)
Item.model_rebuild()
```

---

## Serialization

```python
from pydantic import BaseModel, field_serializer, model_serializer
from datetime import datetime

class Event(BaseModel):
    name: str
    timestamp: datetime
    price_cents: int

    # Custom field serializer
    @field_serializer("timestamp")
    def serialize_timestamp(self, value: datetime) -> str:
        return value.strftime("%Y-%m-%d %H:%M:%S")

    @field_serializer("price_cents")
    def serialize_price(self, value: int, info) -> float:
        return value / 100  # Convert cents to dollars

event = Event(name="Launch", timestamp=datetime(2024, 6, 1, 10, 0), price_cents=1999)
print(event.model_dump())
# {"name": "Launch", "timestamp": "2024-06-01 10:00:00", "price_cents": 19.99}
```

---

## Advanced Patterns

### Generic Models

```python
from pydantic import BaseModel
from typing import Generic, TypeVar

T = TypeVar("T")

class PaginatedResponse(BaseModel, Generic[T]):
    items: list[T]
    total: int
    page: int
    page_size: int

    @property
    def total_pages(self) -> int:
        return (self.total + self.page_size - 1) // self.page_size

class User(BaseModel):
    id: int
    name: str

response = PaginatedResponse[User](
    items=[{"id": 1, "name": "Alice"}, {"id": 2, "name": "Bob"}],
    total=100, page=1, page_size=2
)
print(response.total_pages)  # 50
```

### RootModel (for top-level non-dict types)

```python
from pydantic import RootModel

# Validate a list of strings
Tags = RootModel[list[str]]
tags = Tags(["python", "pydantic"])
print(tags.root)      # ["python", "pydantic"]
print(tags.model_dump())  # ["python", "pydantic"]

# Validate a dict
Config = RootModel[dict[str, int]]
cfg = Config({"timeout": 30, "retries": 3})
```

### Dynamic Model Creation

```python
from pydantic import create_model, Field

# Create model at runtime
DynamicUser = create_model(
    "DynamicUser",
    name=(str, ...),            # (type, default) — ... means required
    age=(int, 0),               # default = 0
    score=(float, Field(ge=0, le=100, default=50.0)),
)

user = DynamicUser(name="Alice", age=30)
print(DynamicUser.model_fields.keys())
```

### Private Attributes

```python
from pydantic import BaseModel, PrivateAttr

class APIClient(BaseModel):
    base_url: str
    api_key: str
    _session: object = PrivateAttr(default=None)    # Not validated, not serialized
    _request_count: int = PrivateAttr(default=0)

    def model_post_init(self, __context) -> None:
        """Hook called after __init__ and validation."""
        import requests
        self._session = requests.Session()
        self._session.headers["Authorization"] = f"Bearer {self.api_key}"

    def get(self, path: str) -> dict:
        self._request_count += 1
        return self._session.get(f"{self.base_url}{path}").json()
```

### Class Variables

```python
from typing import ClassVar
from pydantic import BaseModel

class Product(BaseModel):
    # ClassVar is NOT included in the model schema or serialization
    _registry: ClassVar[dict] = {}
    MAX_PRICE: ClassVar[float] = 10_000.0

    id: int
    name: str
    price: float

    def model_post_init(self, __context) -> None:
        Product._registry[self.id] = self
```

### Discriminated Unions

```python
from typing import Annotated, Union, Literal
from pydantic import BaseModel, Field

class Cat(BaseModel):
    type: Literal["cat"]
    name: str
    indoor: bool = True

class Dog(BaseModel):
    type: Literal["dog"]
    name: str
    breed: str

class Bird(BaseModel):
    type: Literal["bird"]
    name: str
    can_fly: bool = True

Pet = Annotated[Union[Cat, Dog, Bird], Field(discriminator="type")]

class Owner(BaseModel):
    name: str
    pet: Pet

owner = Owner(name="Alice", pet={"type": "cat", "name": "Whiskers"})
print(type(owner.pet))  # <class 'Cat'>
```

### Forward References

```python
from __future__ import annotations
from pydantic import BaseModel

class TreeNode(BaseModel):
    value: int
    children: list[TreeNode] = []

# Must call rebuild for forward references
TreeNode.model_rebuild()

root = TreeNode(value=1, children=[TreeNode(value=2), TreeNode(value=3)])
```
