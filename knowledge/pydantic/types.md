# Pydantic - Types

> Official Documentation: https://docs.pydantic.dev/latest/concepts/types/

## Overview

Pydantic supports a wide range of Python types out of the box, plus a rich library of constrained and specialized types. Types control what values are accepted and how coercion works. Annotated types let you attach validation metadata directly to type hints, making constraints reusable and composable.

---

## Table of Contents

1. [Standard Python Types](#standard-python-types)
2. [Numeric Constrained Types](#numeric-constrained-types)
3. [String & Bytes Types](#string--bytes-types)
4. [Collection Types](#collection-types)
5. [Date & Time Types](#date--time-types)
6. [Network & URL Types](#network--url-types)
7. [Secret Types](#secret-types)
8. [UUID Types](#uuid-types)
9. [Path Types](#path-types)
10. [Special Types](#special-types)
11. [Annotated Types Pattern](#annotated-types-pattern)
12. [Strict Mode](#strict-mode)
13. [Custom Types](#custom-types)

---

## Standard Python Types

Pydantic supports all standard Python primitive and collection types with automatic coercion in lax mode:

```python
from pydantic import BaseModel
from datetime import date, datetime, time, timedelta
from decimal import Decimal
from pathlib import Path
from uuid import UUID
from enum import Enum
import ipaddress

class Color(str, Enum):
    RED = "red"
    GREEN = "green"
    BLUE = "blue"

class AllTypes(BaseModel):
    # Primitives
    int_field: int
    float_field: float
    str_field: str
    bool_field: bool
    bytes_field: bytes
    decimal_field: Decimal

    # Collections
    list_field: list[int]
    tuple_field: tuple[int, str, float]   # Fixed-length typed tuple
    set_field: set[str]
    frozenset_field: frozenset[int]
    dict_field: dict[str, int]

    # Optional / Union
    optional_field: int | None = None
    union_field: int | str

    # Date/Time
    date_field: date
    datetime_field: datetime
    time_field: time
    timedelta_field: timedelta

    # Others
    uuid_field: UUID
    path_field: Path
    enum_field: Color
```

### Coercion Examples

```python
from pydantic import BaseModel

class Model(BaseModel):
    x: int
    y: float
    z: bool

m = Model(x="42", y="3.14", z="true")
# x=42, y=3.14, z=True  (strings coerced to correct types)

m2 = Model(x=1, y=1, z=1)
# x=1, y=1.0, z=True    (int to float, int to bool)
```

---

## Numeric Constrained Types

### Convenience Types

```python
from pydantic import (
    BaseModel,
    PositiveInt,
    NegativeInt,
    NonNegativeInt,
    NonPositiveInt,
    PositiveFloat,
    NegativeFloat,
    NonNegativeFloat,
    NonPositiveFloat,
    FiniteFloat,
)

class Metrics(BaseModel):
    count: PositiveInt          # int > 0
    deficit: NegativeInt        # int < 0
    score: NonNegativeInt       # int >= 0
    penalty: NonPositiveInt     # int <= 0
    temperature: FiniteFloat    # float, excludes inf and nan
    ratio: PositiveFloat        # float > 0
```

### Constrained Integers

```python
from typing import Annotated
from pydantic import BaseModel, Field

# Using Annotated + Field (recommended)
class Order(BaseModel):
    quantity: Annotated[int, Field(ge=1, le=999)]        # 1 <= quantity <= 999
    priority: Annotated[int, Field(ge=0, le=5)]          # 0 <= priority <= 5
    batch_size: Annotated[int, Field(multiple_of=10)]    # Must be multiple of 10
```

### Constrained Floats / Decimals

```python
from decimal import Decimal
from typing import Annotated
from pydantic import BaseModel, Field

class Price(BaseModel):
    amount: Annotated[float, Field(gt=0, lt=1_000_000)]
    tax_rate: Annotated[float, Field(ge=0.0, le=1.0)]
    precise: Annotated[Decimal, Field(decimal_places=2, max_digits=10)]
```

---

## String & Bytes Types

### StrictStr and StrictBytes

```python
from pydantic import BaseModel, StrictStr, StrictBytes

class StrictModel(BaseModel):
    name: StrictStr    # Only accepts str, no coercion
    data: StrictBytes  # Only accepts bytes
```

### Constrained Strings

```python
from typing import Annotated
from pydantic import BaseModel, Field, StringConstraints

class User(BaseModel):
    username: Annotated[
        str,
        StringConstraints(
            min_length=3,
            max_length=50,
            pattern=r"^[a-zA-Z0-9_]+$",
            strip_whitespace=True,
            to_lower=True,
        )
    ]
    # Or using Field shorthand
    email: Annotated[str, Field(min_length=5, max_length=254, pattern=r"^.+@.+\..+$")]
    bio: Annotated[str, Field(max_length=500)] = ""
```

### ImportString

```python
from pydantic import BaseModel, ImportString

class Config(BaseModel):
    encoder_class: ImportString  # Imports Python object from dotted path

config = Config(encoder_class="json.JSONEncoder")
# config.encoder_class is the actual json.JSONEncoder class
```

---

## Collection Types

### Constrained Collections

```python
from typing import Annotated
from pydantic import BaseModel, Field

class Data(BaseModel):
    # List with size constraints
    tags: Annotated[list[str], Field(min_length=1, max_length=10)]
    # Set with size constraints
    permissions: Annotated[set[str], Field(min_length=1)]
    # Items with their own constraints
    scores: Annotated[
        list[Annotated[int, Field(ge=0, le=100)]],
        Field(min_length=1)
    ]
```

### Typed Dict

```python
from typing import TypedDict
from pydantic import BaseModel, TypeAdapter

class Config(TypedDict):
    host: str
    port: int

class AppSettings(BaseModel):
    database: Config

app = AppSettings(database={"host": "localhost", "port": 5432})
```

### JsonValue

Recursive type for any JSON-serializable value:

```python
from pydantic import BaseModel
from pydantic.types import JsonValue

class FlexibleModel(BaseModel):
    metadata: JsonValue   # Can be dict, list, str, int, float, bool, or None

m = FlexibleModel(metadata={"key": [1, 2, {"nested": True}]})
```

---

## Date & Time Types

### Constrained Date Types

```python
from pydantic import BaseModel, PastDate, FutureDate, AwareDatetime, NaiveDatetime, PastDatetime, FutureDatetime
from datetime import datetime, timezone

class Event(BaseModel):
    birth_date: PastDate          # date in the past
    expiry_date: FutureDate       # date in the future
    scheduled_at: AwareDatetime   # datetime with timezone info
    logged_at: NaiveDatetime      # datetime WITHOUT timezone info
    starts: FutureDatetime        # datetime in the future
    ended: PastDatetime           # datetime in the past
```

### Datetime Parsing

```python
from pydantic import BaseModel
from datetime import datetime

class Log(BaseModel):
    created_at: datetime

# Pydantic accepts multiple formats:
Log(created_at="2024-01-15T10:30:00")           # ISO string
Log(created_at="2024-01-15T10:30:00+00:00")     # With timezone
Log(created_at=1705312200)                       # Unix timestamp (seconds)
Log(created_at=1705312200000)                    # Unix timestamp (milliseconds)
```

---

## Network & URL Types

```python
from pydantic import BaseModel, AnyUrl, AnyHttpUrl, HttpUrl, AnyWebsocketUrl, IPvAnyAddress, EmailStr
from pydantic.networks import PostgresDsn, RedisDsn, MongoDsn, AmqpDsn

class ServiceConfig(BaseModel):
    api_url: HttpUrl             # Must be http:// or https://
    webhook: AnyHttpUrl          # Any HTTP URL
    ws_url: AnyWebsocketUrl      # ws:// or wss://
    client_ip: IPvAnyAddress     # IPv4 or IPv6
    contact: EmailStr            # Valid email format (requires email-validator package)

class DatabaseConfig(BaseModel):
    postgres_url: PostgresDsn    # postgresql://...
    redis_url: RedisDsn          # redis://...
    mongo_url: MongoDsn          # mongodb://...

config = ServiceConfig(
    api_url="https://api.example.com/v1",
    webhook="http://hooks.example.com",
    ws_url="wss://ws.example.com",
    client_ip="192.168.1.1",
    contact="user@example.com"
)
# config.api_url is a Url object with .scheme, .host, .path, etc.
print(config.api_url.host)  # "api.example.com"
```

---

## Secret Types

Secret types mask sensitive values in `repr()` and serialization:

```python
from pydantic import BaseModel, SecretStr, SecretBytes

class Credentials(BaseModel):
    username: str
    password: SecretStr
    private_key: SecretBytes

creds = Credentials(username="alice", password="s3cr3t!", private_key=b"key_data")

print(creds)
# username='alice' password=SecretStr('**********') private_key=SecretBytes(b'**********')

print(creds.password)           # SecretStr('**********')
print(creds.password.get_secret_value())  # "s3cr3t!"  (reveals actual value)

# NOT included in model_dump() by default
creds.model_dump()
# {"username": "alice", "password": SecretStr('**********'), ...}
```

### Generic Secret

```python
from pydantic import BaseModel, Secret

class Config(BaseModel):
    token: Secret[int]    # Secret wrapping any type

cfg = Config(token=12345)
print(cfg.token)                    # Secret(12345) — masked
print(cfg.token.get_secret_value()) # 12345
```

---

## UUID Types

```python
from pydantic import BaseModel, UUID1, UUID3, UUID4, UUID5
from uuid import UUID

class Resource(BaseModel):
    id: UUID4      # Must be a version-4 UUID
    parent_id: UUID1 | None = None

r = Resource(id="550e8400-e29b-41d4-a716-446655440000")
```

---

## Path Types

```python
from pydantic import BaseModel, FilePath, DirectoryPath, NewPath

class FileConfig(BaseModel):
    input_file: FilePath          # Must exist and be a file
    output_dir: DirectoryPath     # Must exist and be a directory
    log_file: NewPath             # Must NOT exist (for creating new files)
    any_path: Path                # Any path, no existence check
```

---

## Special Types

### ByteSize

Converts human-readable byte sizes to integers:

```python
from pydantic import BaseModel, ByteSize

class Config(BaseModel):
    max_upload: ByteSize
    buffer: ByteSize

cfg = Config(max_upload="50MB", buffer="512KiB")
print(cfg.max_upload)          # 52428800 (bytes)
print(cfg.max_upload.human_readable())  # "50.0MB"
```

### PaymentCardNumber

```python
from pydantic import BaseModel, PaymentCardNumber

class Payment(BaseModel):
    card_number: PaymentCardNumber  # Validates via Luhn algorithm

payment = Payment(card_number="4111111111111111")
print(payment.card_number.brand)   # "Visa"
```

### Base64 Types

```python
from pydantic import BaseModel, Base64Bytes, Base64Str, Base64UrlStr

class Encoded(BaseModel):
    data: Base64Bytes   # Accepts base64 string, decodes to bytes
    text: Base64Str     # Accepts base64 string, decodes to str
    url: Base64UrlStr   # URL-safe base64

enc = Encoded(data="SGVsbG8=", text="SGVsbG8=", url="SGVsbG8=")
print(enc.data)  # b"Hello"
print(enc.text)  # "Hello"
```

### Json Type

Parse a JSON string and validate the result:

```python
from pydantic import BaseModel, Json

class Request(BaseModel):
    config: Json[dict[str, int]]  # Accepts JSON string, parses and validates

req = Request(config='{"timeout": 30, "retries": 3}')
print(req.config)  # {"timeout": 30, "retries": 3}  (already a dict)
```

### FailFast

Stop validation at first error (for collections):

```python
from typing import Annotated
from pydantic import BaseModel, FailFast

class Batch(BaseModel):
    items: Annotated[list[int], FailFast()]  # Raises at first invalid item
```

### OnErrorOmit

Silently omit invalid items from a collection:

```python
from typing import Annotated
from pydantic import BaseModel, OnErrorOmit

class Flexible(BaseModel):
    values: list[Annotated[int, OnErrorOmit]]

m = Flexible(values=[1, "bad", 3, None, 5])
print(m.values)  # [1, 3, 5]  — invalid items omitted
```

---

## Annotated Types Pattern

`Annotated` lets you attach validators and constraints to types, making them reusable across models:

```python
from typing import Annotated
from pydantic import BaseModel, Field, AfterValidator, BeforeValidator, PlainSerializer, TypeAdapter

# Reusable constrained types
PositiveInt = Annotated[int, Field(gt=0)]
NonEmptyStr = Annotated[str, Field(min_length=1, strip_whitespace=True)]
Percentage = Annotated[float, Field(ge=0.0, le=100.0)]
Email = Annotated[str, Field(pattern=r"^[^@]+@[^@]+\.[^@]+$")]

# Types with transformations
def to_upper(v: str) -> str:
    return v.upper()

UpperStr = Annotated[str, AfterValidator(to_upper)]

def parse_csv_list(v) -> list[str]:
    if isinstance(v, str):
        return [item.strip() for item in v.split(",")]
    return v

CsvList = Annotated[list[str], BeforeValidator(parse_csv_list)]

# Types with custom serialization
TruncatedFloat = Annotated[
    float,
    AfterValidator(lambda x: round(x, 2)),
    PlainSerializer(lambda x: f"{x:.2f}", return_type=str),
]

# Reuse across multiple models
class Product(BaseModel):
    name: NonEmptyStr
    price: PositiveInt
    discount: Percentage
    owner_email: Email

class User(BaseModel):
    name: NonEmptyStr
    tags: CsvList
    score: TruncatedFloat

# Validate standalone types
ta = TypeAdapter(PositiveInt)
ta.validate_python(5)     # 5
ta.validate_python(-1)    # raises ValidationError

# Named type aliases (Python 3.12+)
type ProductName = Annotated[str, Field(min_length=1, max_length=100)]
```

---

## Strict Mode

Strict mode disables all automatic type coercion.

### Global Strict Mode

```python
from pydantic import BaseModel, ConfigDict

class StrictUser(BaseModel):
    model_config = ConfigDict(strict=True)
    id: int
    name: str

StrictUser(id=1, name="Alice")   # OK
StrictUser(id="1", name="Alice") # ValidationError: id must be int
```

### Per-Field Strict Mode

```python
from typing import Annotated
from pydantic import BaseModel, Field, Strict, StrictInt, StrictStr

class Mixed(BaseModel):
    strict_id: StrictInt                               # Strict int
    strict_name: StrictStr                             # Strict str
    flexible_age: int                                  # Normal coercion
    annotated_strict: Annotated[float, Strict()]       # Using Strict marker
    strict_but_overridden: int = Field(strict=False)   # Override global strict

    model_config = ConfigDict(strict=True)  # Global strict, per-field can override
```

### Per-Validation Strict Mode

```python
from pydantic import BaseModel, ValidationError

class Model(BaseModel):
    value: int

# Normal: coerces "42" to 42
m = Model.model_validate({"value": "42"})

# Strict: rejects "42"
try:
    m = Model.model_validate({"value": "42"}, strict=True)
except ValidationError as e:
    print(e)  # Input should be a valid integer
```

### Strict Mode Behavior Table

| Python input | Target type | Lax mode | Strict mode |
|-------------|-------------|----------|-------------|
| `"42"` | `int` | `42` | Error |
| `42` | `str` | `"42"` | Error |
| `1` | `bool` | `True` | Error |
| `"2024-01-01"` | `date` | `date(2024,1,1)` | Error (string not date) |
| `1705312200` | `datetime` | `datetime(...)` | Error |

---

## Custom Types

### Class-based Custom Type

```python
from pydantic import GetCoreSchemaHandler
from pydantic_core import core_schema

class PhoneNumber:
    """Custom type that validates and normalizes phone numbers."""

    def __init__(self, value: str):
        self.value = self._normalize(value)

    def _normalize(self, v: str) -> str:
        import re
        digits = re.sub(r"\D", "", v)
        if len(digits) == 10:
            return f"+1{digits}"
        elif len(digits) == 11 and digits[0] == "1":
            return f"+{digits}"
        else:
            raise ValueError(f"Invalid phone number: {v}")

    def __str__(self) -> str:
        return self.value

    def __repr__(self) -> str:
        return f"PhoneNumber({self.value!r})"

    @classmethod
    def __get_pydantic_core_schema__(
        cls, source_type, handler: GetCoreSchemaHandler
    ):
        return core_schema.no_info_plain_validator_function(
            cls,
            serialization=core_schema.to_string_ser_schema(),
        )

from pydantic import BaseModel

class Contact(BaseModel):
    name: str
    phone: PhoneNumber

c = Contact(name="Alice", phone="555-867-5309")
print(c.phone)  # PhoneNumber('+15558675309')
```

### Annotation-based Custom Validator

```python
from typing import Annotated, Any
from pydantic import GetCoreSchemaHandler
from pydantic_core import core_schema
from dataclasses import dataclass

@dataclass(frozen=True)
class Gt:
    """Custom annotation: greater than."""
    value: float

    def __get_pydantic_core_schema__(
        self, source_type: Any, handler: GetCoreSchemaHandler
    ):
        schema = handler(source_type)
        return core_schema.with_info_after_validator_function(
            lambda v, info: v if v > self.value else (_ for _ in ()).throw(
                ValueError(f"Value must be > {self.value}")
            ),
            schema,
        )

BigInt = Annotated[int, Gt(100)]

from pydantic import BaseModel

class Stats(BaseModel):
    high_score: BigInt

Stats(high_score=150)  # OK
Stats(high_score=50)   # ValidationError
```
