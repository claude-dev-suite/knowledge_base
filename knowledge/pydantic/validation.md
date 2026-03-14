# Pydantic - Validation

> Official Documentation: https://docs.pydantic.dev/latest/concepts/validators/

## Overview

Pydantic provides multiple layers of validation: type checking with coercion, field-level validators for custom field logic, model-level validators for cross-field logic, and a rich `ValidationError` system that collects all errors in a single pass. Validators can be attached via the `@field_validator` / `@model_validator` decorators or inline via `Annotated` types.

---

## Table of Contents

1. [ValidationError](#validationerror)
2. [Field Validators — @field_validator](#field-validators--field_validator)
3. [Annotated Validators](#annotated-validators)
4. [Model Validators — @model_validator](#model-validators--model_validator)
5. [Raising Validation Errors](#raising-validation-errors)
6. [ValidationInfo & Context](#validationinfo--context)
7. [Validator Execution Order](#validator-execution-order)
8. [Special Validator Utilities](#special-validator-utilities)
9. [Common Validation Patterns](#common-validation-patterns)

---

## ValidationError

Pydantic collects **all** validation errors and raises a single `ValidationError` at the end. It never short-circuits after the first error.

```python
from pydantic import BaseModel, ValidationError

class User(BaseModel):
    id: int
    name: str
    age: int

try:
    user = User(id="not_int", name=123, age=-5)
except ValidationError as e:
    print(e)
    # 3 validation errors for User
    # id: Input should be a valid integer, unable to parse string as an integer
    # age: Input should be greater than 0

    # Access individual errors
    print(e.error_count())   # 2 (id and age; name coerced successfully)
    print(e.errors())        # List of error dicts

    for error in e.errors():
        print(f"Field: {error['loc']}")
        print(f"Type:  {error['type']}")
        print(f"Msg:   {error['msg']}")
        print(f"Input: {error['input']}")
        print()
```

### Error Dict Structure

Each error in `e.errors()` contains:

| Key | Type | Description |
|-----|------|-------------|
| `loc` | tuple | Location — field names / indices path, e.g. `("address", "city")` |
| `type` | str | Error type identifier, e.g. `"int_parsing"`, `"missing"`, `"value_error"` |
| `msg` | str | Human-readable error message |
| `input` | Any | The invalid input value |
| `url` | str | Link to Pydantic error docs for this error type |
| `ctx` | dict | Additional context (e.g. `{"gt": 0}` for a `gt` constraint) |

```python
try:
    User(id="bad", name="Alice", age=-1)
except ValidationError as e:
    errors = e.errors(include_url=False)
    print(errors)
    # [
    #   {"type": "int_parsing", "loc": ("id",), "msg": "...", "input": "bad", ...},
    #   {"type": "greater_than", "loc": ("age",), "msg": "...", "ctx": {"gt": 0}, ...}
    # ]
```

### Catching and Re-raising

```python
from pydantic import ValidationError
import json

def parse_user(data: dict):
    try:
        return User(**data)
    except ValidationError as e:
        # Convert to a clean dict for API responses
        return {
            "status": "error",
            "errors": [
                {"field": ".".join(str(loc) for loc in err["loc"]), "message": err["msg"]}
                for err in e.errors(include_url=False)
            ]
        }
```

---

## Field Validators — @field_validator

### Basic Usage

```python
from pydantic import BaseModel, field_validator

class Product(BaseModel):
    name: str
    price: float
    sku: str

    @field_validator("name")
    @classmethod
    def name_must_not_be_blank(cls, v: str) -> str:
        if not v.strip():
            raise ValueError("Name must not be blank")
        return v.strip().title()

    @field_validator("price")
    @classmethod
    def price_must_be_positive(cls, v: float) -> float:
        if v <= 0:
            raise ValueError("Price must be greater than 0")
        return round(v, 2)

    @field_validator("sku")
    @classmethod
    def sku_format(cls, v: str) -> str:
        import re
        v = v.upper()
        if not re.match(r"^[A-Z]{2,4}-\d{4,8}$", v):
            raise ValueError("SKU must match format: XX-1234")
        return v
```

### Multiple Fields at Once

```python
@field_validator("first_name", "last_name", "middle_name")
@classmethod
def name_must_be_non_empty(cls, v: str) -> str:
    if not v.strip():
        raise ValueError("Name field must not be empty")
    return v.strip()

# Apply to ALL fields
@field_validator("*")
@classmethod
def strip_strings(cls, v):
    if isinstance(v, str):
        return v.strip()
    return v
```

### Validator Modes

```python
from pydantic import BaseModel, field_validator

class Config(BaseModel):
    tags: list[str]
    port: int
    timeout: float

    # mode="before": receives raw input before type parsing
    @field_validator("tags", mode="before")
    @classmethod
    def parse_tags(cls, v):
        if isinstance(v, str):
            return [t.strip() for t in v.split(",") if t.strip()]
        return v

    # mode="after" (default): receives parsed, type-correct value
    @field_validator("port", mode="after")
    @classmethod
    def validate_port(cls, v: int) -> int:
        if not (1 <= v <= 65535):
            raise ValueError(f"Port {v} is outside valid range 1-65535")
        return v

    # mode="wrap": wrap around Pydantic's own validation
    @field_validator("timeout", mode="wrap")
    @classmethod
    def validate_timeout(cls, v, handler):
        if v is None:
            return 30.0  # Default timeout
        try:
            result = handler(v)
            if result <= 0:
                raise ValueError("Timeout must be positive")
            return result
        except Exception as e:
            raise ValueError(f"Invalid timeout: {e}")

cfg = Config(tags="python, pydantic, fast", port=8080, timeout=None)
# tags=["python", "pydantic", "fast"], port=8080, timeout=30.0
```

### Accessing Previously Validated Fields

Use `ValidationInfo` to access already-validated sibling fields:

```python
from pydantic import BaseModel, field_validator, ValidationInfo

class PasswordModel(BaseModel):
    password: str
    password_repeat: str

    @field_validator("password")
    @classmethod
    def password_strength(cls, v: str) -> str:
        if len(v) < 8:
            raise ValueError("Password must be at least 8 characters")
        if not any(c.isupper() for c in v):
            raise ValueError("Password must contain an uppercase letter")
        if not any(c.isdigit() for c in v):
            raise ValueError("Password must contain a digit")
        return v

    @field_validator("password_repeat", mode="after")
    @classmethod
    def passwords_must_match(cls, v: str, info: ValidationInfo) -> str:
        # info.data contains already-validated fields
        if "password" in info.data and v != info.data["password"]:
            raise ValueError("Passwords do not match")
        return v
```

---

## Annotated Validators

Annotated validators are reusable, composable, and keep the model class clean. They're defined as standalone functions or classes attached via `Annotated`.

```python
from typing import Annotated, Any
from pydantic import BaseModel, AfterValidator, BeforeValidator, WrapValidator, PlainValidator
from pydantic_core.core_schema import ValidatorFunctionWrapHandler

# AfterValidator: runs after Pydantic's type validation
def ensure_uppercase(v: str) -> str:
    return v.upper()

# BeforeValidator: runs before Pydantic's type validation
def coerce_to_list(v: Any) -> Any:
    if isinstance(v, str):
        return [item.strip() for item in v.split(",")]
    if not isinstance(v, list):
        return [v]
    return v

# WrapValidator: most flexible
def clamp_value(v: Any, handler: ValidatorFunctionWrapHandler) -> float:
    result = handler(v)
    return max(0.0, min(100.0, result))

# PlainValidator: replaces all other validation for this type
def parse_bool(v: Any) -> bool:
    if isinstance(v, bool):
        return v
    if isinstance(v, str):
        return v.lower() in ("true", "1", "yes", "on")
    return bool(v)

# Compose into types
UpperStr = Annotated[str, AfterValidator(ensure_uppercase)]
CsvList = Annotated[list[str], BeforeValidator(coerce_to_list)]
ClampedFloat = Annotated[float, WrapValidator(clamp_value)]
FlexBool = Annotated[bool, PlainValidator(parse_bool)]

class Report(BaseModel):
    title: UpperStr
    categories: CsvList
    completion: ClampedFloat
    is_published: FlexBool

r = Report(
    title="my report",
    categories="tech, science, art",
    completion=150.0,   # clamped to 100.0
    is_published="yes"
)
print(r)
# title='MY REPORT' categories=['tech', 'science', 'art'] completion=100.0 is_published=True
```

---

## Model Validators — @model_validator

### After Mode (instance method)

Runs after all fields are validated. Receives and returns `self`:

```python
from pydantic import BaseModel, model_validator
from typing_extensions import Self
from datetime import date

class BookingRequest(BaseModel):
    check_in: date
    check_out: date
    guests: int
    room_type: str
    total_price: float | None = None

    @model_validator(mode="after")
    def validate_dates_and_price(self) -> Self:
        if self.check_out <= self.check_in:
            raise ValueError("check_out must be after check_in")

        nights = (self.check_out - self.check_in).days
        rate = {"single": 100, "double": 150, "suite": 300}.get(self.room_type, 100)

        # Computed field logic in validator
        if self.total_price is None:
            self.total_price = nights * rate * self.guests

        return self
```

### Before Mode (classmethod)

Runs before individual field validation. Receives raw input (dict or any type):

```python
from pydantic import BaseModel, model_validator
from typing import Any

class NormalizedUser(BaseModel):
    name: str
    email: str
    role: str = "user"

    @model_validator(mode="before")
    @classmethod
    def normalize_input(cls, data: Any) -> Any:
        if not isinstance(data, dict):
            return data
        # Rename legacy fields
        if "full_name" in data:
            data["name"] = data.pop("full_name")
        # Normalize email
        if "email" in data:
            data["email"] = data["email"].lower().strip()
        # Inject defaults from external source
        if "role" not in data:
            data["role"] = "user"
        return data

# Both work:
NormalizedUser(name="Alice", email="Alice@Example.COM")
NormalizedUser(full_name="Bob", email="  Bob@EXAMPLE.COM  ")
```

### Wrap Mode

Most flexible — wraps the entire validation pipeline:

```python
from pydantic import BaseModel, model_validator, ValidationError
from pydantic import ModelWrapValidatorHandler
from typing_extensions import Self
import logging

class MonitoredModel(BaseModel):
    value: int
    label: str

    @model_validator(mode="wrap")
    @classmethod
    def monitor_validation(
        cls, data: Any, handler: ModelWrapValidatorHandler[Self]
    ) -> Self:
        try:
            result = handler(data)
            logging.info("Validated %s successfully", cls.__name__)
            return result
        except ValidationError as exc:
            logging.warning(
                "Validation failed for %s with data %s: %s",
                cls.__name__, data, exc
            )
            raise
```

---

## Raising Validation Errors

### ValueError (most common)

```python
from pydantic import BaseModel, field_validator

class Model(BaseModel):
    score: int

    @field_validator("score")
    @classmethod
    def check_score(cls, v: int) -> int:
        if v < 0:
            raise ValueError("Score must be non-negative")
        return v
```

### AssertionError

```python
@field_validator("age")
@classmethod
def check_age(cls, v: int) -> int:
    assert v >= 0, "Age cannot be negative"
    assert v <= 150, f"Age {v} is unrealistically high"
    return v
```

Note: `AssertionError` is ignored when Python runs with the `-O` (optimize) flag. Prefer `ValueError` for production validators.

### PydanticCustomError

Provides custom error types and structured context for better error responses:

```python
from pydantic import BaseModel, field_validator
from pydantic_core import PydanticCustomError

class CreditCard(BaseModel):
    number: str

    @field_validator("number")
    @classmethod
    def validate_card(cls, v: str) -> str:
        digits = v.replace(" ", "").replace("-", "")
        if not digits.isdigit():
            raise PydanticCustomError(
                "card_non_numeric",
                "Card number must contain only digits, got: {value}",
                {"value": v}
            )
        if not cls._luhn_check(digits):
            raise PydanticCustomError(
                "card_invalid_luhn",
                "Card number {number} failed Luhn validation",
                {"number": digits}
            )
        return digits

    @classmethod
    def _luhn_check(cls, number: str) -> bool:
        total = 0
        reverse_digits = number[::-1]
        for i, d in enumerate(reverse_digits):
            n = int(d)
            if i % 2 == 1:
                n *= 2
                if n > 9:
                    n -= 9
            total += n
        return total % 10 == 0

try:
    CreditCard(number="1234-5678-9012-3456")
except Exception as e:
    for err in e.errors(include_url=False):
        print(err["type"])   # "card_invalid_luhn"
        print(err["msg"])    # "Card number ... failed Luhn validation"
        print(err["ctx"])    # {"number": "..."}
```

### PydanticUseDefault

Instruct Pydantic to use the field's default value instead of raising an error:

```python
from typing import Annotated, Any
from pydantic import BaseModel, BeforeValidator
from pydantic_core import PydanticUseDefault

def use_default_if_none(v: Any) -> Any:
    if v is None:
        raise PydanticUseDefault()
    return v

class Model(BaseModel):
    name: Annotated[str, BeforeValidator(use_default_if_none)] = "anonymous"
    score: Annotated[int, BeforeValidator(use_default_if_none)] = 0

m = Model(name=None, score=None)
# name="anonymous", score=0
```

---

## ValidationInfo & Context

### ValidationInfo in Field Validators

```python
from pydantic import BaseModel, field_validator, ValidationInfo

class FormModel(BaseModel):
    country: str
    postal_code: str
    city: str

    @field_validator("postal_code", mode="after")
    @classmethod
    def validate_postal_code(cls, v: str, info: ValidationInfo) -> str:
        country = info.data.get("country", "")
        if country == "US":
            import re
            if not re.match(r"^\d{5}(-\d{4})?$", v):
                raise ValueError("US postal codes must be 5 digits (or ZIP+4)")
        elif country == "UK":
            if len(v) < 5 or len(v) > 8:
                raise ValueError("UK postcodes must be 5-8 characters")
        return v.upper()
```

### Validation Context

Pass extra context at validation time:

```python
from pydantic import BaseModel, field_validator, ValidationInfo

class Content(BaseModel):
    text: str
    language: str = "en"

    @field_validator("text", mode="after")
    @classmethod
    def filter_stopwords(cls, v: str, info: ValidationInfo) -> str:
        if not isinstance(info.context, dict):
            return v
        stopwords = info.context.get("stopwords", set())
        words = [w for w in v.split() if w.lower() not in stopwords]
        return " ".join(words)

# Pass context to validation
content = Content.model_validate(
    {"text": "This is an example document", "language": "en"},
    context={"stopwords": {"this", "is", "an"}}
)
print(content.text)  # "example document"
```

### Available ValidationInfo Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `data` | dict | Already-validated fields (field validators only) |
| `context` | Any | User-provided context passed to `model_validate()` |
| `field_name` | str | Name of the field being validated |
| `mode` | str | Validation mode: `"python"`, `"json"`, or `"strings"` |
| `config` | ConfigWrapper | Model's config |

---

## Validator Execution Order

When multiple validators/annotations are applied to a field, execution order is:

1. **Before validators**: right-to-left (last `Annotated` item runs first)
2. **Pydantic core validation** (type checking, coercion)
3. **After validators**: left-to-right (first `Annotated` item runs first)

```python
from typing import Annotated
from pydantic import BaseModel, AfterValidator, BeforeValidator, WrapValidator

def before_1(v): print("before_1"); return v
def before_2(v): print("before_2"); return v
def after_1(v): print("after_1"); return v
def after_2(v): print("after_2"); return v

class Model(BaseModel):
    name: Annotated[
        str,
        AfterValidator(after_1),   # runs 3rd
        AfterValidator(after_2),   # runs 4th
        BeforeValidator(before_2), # runs 2nd
        BeforeValidator(before_1), # runs 1st
    ]
# Output order: before_1, before_2, [core], after_1, after_2
```

For `@field_validator` decorators, execution order follows field declaration order.

---

## Special Validator Utilities

### InstanceOf

```python
from pydantic import BaseModel, InstanceOf

class Fruit:
    def __init__(self, name: str):
        self.name = name

class Basket(BaseModel):
    fruits: list[InstanceOf[Fruit]]  # Each item must be a Fruit instance

apple = Fruit("apple")
Basket(fruits=[apple])  # OK
Basket(fruits=["apple"])  # ValidationError: not a Fruit instance
```

### SkipValidation

```python
from pydantic import BaseModel, SkipValidation
from typing import Annotated

class TrustedInput(BaseModel):
    # Skip validation entirely for these fields
    raw_data: SkipValidation[dict]
    fast_list: list[SkipValidation[str]]
```

---

## Common Validation Patterns

### Email Validation (without email-validator package)

```python
from typing import Annotated
from pydantic import BaseModel, Field

Email = Annotated[
    str,
    Field(
        pattern=r"^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$",
        max_length=254,
        to_lower=True,
    )
]
```

### URL Slug Validation

```python
from typing import Annotated
from pydantic import AfterValidator

def normalize_slug(v: str) -> str:
    import re
    v = v.lower().strip()
    v = re.sub(r"[^\w\s-]", "", v)
    v = re.sub(r"[\s_]+", "-", v)
    v = re.sub(r"-+", "-", v).strip("-")
    if not v:
        raise ValueError("Slug cannot be empty after normalization")
    return v

Slug = Annotated[str, AfterValidator(normalize_slug)]
```

### Conditional Required Fields

```python
from pydantic import BaseModel, model_validator
from typing_extensions import Self

class Address(BaseModel):
    street: str
    city: str
    state: str | None = None
    country: str

    @model_validator(mode="after")
    def state_required_for_us(self) -> Self:
        if self.country == "US" and self.state is None:
            raise ValueError("State is required for US addresses")
        return self
```

### Cross-Model Validation

```python
from pydantic import BaseModel, model_validator
from typing_extensions import Self

class Budget(BaseModel):
    total: float
    spent: float
    reserved: float

    @model_validator(mode="after")
    def check_budget(self) -> Self:
        if self.spent + self.reserved > self.total:
            raise ValueError(
                f"Spent ({self.spent}) + reserved ({self.reserved}) "
                f"exceeds total budget ({self.total})"
            )
        return self

    @property
    def available(self) -> float:
        return self.total - self.spent - self.reserved
```

### Dynamic Validation with Context

```python
from pydantic import BaseModel, field_validator, ValidationInfo

class FeatureRequest(BaseModel):
    feature_name: str
    priority: int

    @field_validator("feature_name", mode="after")
    @classmethod
    def validate_feature_exists(cls, v: str, info: ValidationInfo) -> str:
        allowed_features = (info.context or {}).get("allowed_features", set())
        if allowed_features and v not in allowed_features:
            raise ValueError(
                f"Feature '{v}' is not in allowed features: {allowed_features}"
            )
        return v

# At runtime, inject context
available = {"feature_a", "feature_b", "feature_c"}
req = FeatureRequest.model_validate(
    {"feature_name": "feature_a", "priority": 1},
    context={"allowed_features": available}
)
```

### Validator That Transforms Data

```python
from pydantic import BaseModel, field_validator
import re

class SearchQuery(BaseModel):
    q: str
    filters: dict[str, str] = {}

    @field_validator("q", mode="before")
    @classmethod
    def parse_query_string(cls, v: str) -> str:
        v = v.strip()
        if not v:
            raise ValueError("Search query cannot be empty")
        # Sanitize potentially dangerous characters
        v = re.sub(r"[<>\"';&]", "", v)
        return v

    @field_validator("filters", mode="before")
    @classmethod
    def parse_filters_string(cls, v):
        if isinstance(v, str):
            result = {}
            for part in v.split(";"):
                if "=" in part:
                    k, _, val = part.partition("=")
                    result[k.strip()] = val.strip()
            return result
        return v
```

### Full Example: User Registration

```python
from pydantic import BaseModel, field_validator, model_validator, Field, ValidationInfo
from typing import Annotated
from typing_extensions import Self
import re

Email = Annotated[str, Field(pattern=r"^[^@]+@[^@]+\.[^@]+$", max_length=254)]
Username = Annotated[str, Field(min_length=3, max_length=30, pattern=r"^[a-zA-Z0-9_]+$")]

class UserRegistration(BaseModel):
    username: Username
    email: Email
    password: str = Field(min_length=8, max_length=128)
    password_confirm: str
    age: int = Field(ge=13, le=120)
    invite_code: str | None = None

    @field_validator("username", mode="after")
    @classmethod
    def username_not_reserved(cls, v: str) -> str:
        reserved = {"admin", "root", "system", "superuser"}
        if v.lower() in reserved:
            raise ValueError(f"Username '{v}' is reserved")
        return v.lower()

    @field_validator("password", mode="after")
    @classmethod
    def password_complexity(cls, v: str) -> str:
        errors = []
        if not any(c.isupper() for c in v):
            errors.append("at least one uppercase letter")
        if not any(c.islower() for c in v):
            errors.append("at least one lowercase letter")
        if not any(c.isdigit() for c in v):
            errors.append("at least one digit")
        if not any(c in "!@#$%^&*()_+-=[]{}|;':\",./<>?" for c in v):
            errors.append("at least one special character")
        if errors:
            raise ValueError(f"Password requires: {', '.join(errors)}")
        return v

    @field_validator("password_confirm", mode="after")
    @classmethod
    def passwords_match(cls, v: str, info: ValidationInfo) -> str:
        if "password" in info.data and v != info.data["password"]:
            raise ValueError("Passwords do not match")
        return v

    @model_validator(mode="after")
    def validate_invite_if_required(self) -> Self:
        # Use context to pass invite requirement
        return self
```
