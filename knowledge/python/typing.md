# Python Type Hints

Complete guide to Python type hints, including modern PEP 695 syntax.

## Version Compatibility

| Feature | Python Version |
|---------|----------------|
| Basic type hints | 3.5+ |
| Built-in generics (`list[T]`) | 3.9+ |
| Union with `\|` | 3.10+ |
| PEP 695 type parameters | 3.12+ |
| Type aliases with `type` | 3.12+ |

## Basic Types

```python
# Primitives
name: str = "Alice"
age: int = 30
price: float = 19.99
is_active: bool = True
data: bytes = b"hello"

# None type
result: None = None

# Any (avoid when possible)
from typing import Any
unknown: Any = get_dynamic_value()
```

## Collections (Python 3.9+)

```python
# Use built-in types directly
names: list[str] = ["Alice", "Bob"]
scores: dict[str, int] = {"math": 95}
coords: tuple[float, float] = (1.0, 2.0)
unique: set[int] = {1, 2, 3}
frozen: frozenset[str] = frozenset(["a", "b"])

# Variable-length tuple
args: tuple[int, ...] = (1, 2, 3, 4, 5)

# Nested collections
matrix: list[list[int]] = [[1, 2], [3, 4]]
users: dict[str, dict[str, Any]] = {"user1": {"name": "Alice"}}
```

## Union Types (Python 3.10+)

```python
# Modern syntax with |
def process(value: str | int | None) -> str:
    if value is None:
        return "empty"
    return str(value)

# Optional is just T | None
def find(id: int) -> User | None:
    return db.get(id)

# Multiple types
ID = str | int
```

## PEP 695 - Type Parameters (Python 3.12+)

### Generic Functions

```python
# New syntax - cleaner, no TypeVar needed
def first[T](items: list[T]) -> T | None:
    return items[0] if items else None

def merge[K, V](d1: dict[K, V], d2: dict[K, V]) -> dict[K, V]:
    return {**d1, **d2}

# Bounded type parameter
def stringify[T: (int, float, str)](value: T) -> str:
    return str(value)

# Constrained to protocol
from typing import SupportsLessThan

def min_item[T: SupportsLessThan](items: list[T]) -> T:
    return min(items)
```

### Generic Classes

```python
class Stack[T]:
    def __init__(self) -> None:
        self._items: list[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        if not self._items:
            raise IndexError("Stack is empty")
        return self._items.pop()

    def peek(self) -> T | None:
        return self._items[-1] if self._items else None

# Multiple type parameters
class Pair[T, U]:
    def __init__(self, first: T, second: U) -> None:
        self.first = first
        self.second = second

    def swap(self) -> "Pair[U, T]":
        return Pair(self.second, self.first)
```

### Type Aliases (Python 3.12+)

```python
# Simple alias
type UserID = int
type JSON = dict[str, Any]

# Generic alias
type Vector[T] = list[T]
type Mapping[K, V] = dict[K, V]

# Complex alias
type Handler[T] = Callable[[T], None]
type Result[T, E] = Ok[T] | Err[E]
type Middleware = Callable[[Request], Awaitable[Response]]

# Usage
def process(ids: Vector[UserID]) -> None:
    pass
```

## Callable Types

```python
from collections.abc import Callable, Awaitable

# Function that takes int and returns str
Converter = Callable[[int], str]

# Function with multiple args
Comparator = Callable[[int, int], bool]

# Function with no args
Factory = Callable[[], User]

# Async function
AsyncHandler = Callable[[Request], Awaitable[Response]]

# Usage
def apply(fn: Callable[[int], int], value: int) -> int:
    return fn(value)

# With ParamSpec for decorators (Python 3.10+)
from typing import ParamSpec, TypeVar

P = ParamSpec("P")
R = TypeVar("R")

def logged(fn: Callable[P, R]) -> Callable[P, R]:
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        print(f"Calling {fn.__name__}")
        return fn(*args, **kwargs)
    return wrapper
```

## Protocol (Structural Typing)

```python
from typing import Protocol, runtime_checkable

class Drawable(Protocol):
    def draw(self) -> None: ...

class Serializable(Protocol):
    def to_json(self) -> str: ...
    @classmethod
    def from_json(cls, data: str) -> "Self": ...

# Any class with draw() method satisfies Drawable
class Circle:
    def draw(self) -> None:
        print("Drawing circle")

def render(item: Drawable) -> None:
    item.draw()

render(Circle())  # Works - Circle has draw()

# Runtime checkable protocol
@runtime_checkable
class Sized(Protocol):
    def __len__(self) -> int: ...

assert isinstance([1, 2, 3], Sized)  # True
```

## TypedDict

```python
from typing import TypedDict, NotRequired, Required

class UserDict(TypedDict):
    name: str
    email: str
    age: NotRequired[int]  # Optional key

# All keys required by default
class ConfigDict(TypedDict, total=True):
    host: str
    port: int

# All keys optional
class PartialConfig(TypedDict, total=False):
    timeout: int
    retries: int

# Usage
def create_user(data: UserDict) -> User:
    return User(name=data["name"], email=data["email"])
```

## Literal and Final

```python
from typing import Literal, Final

# Literal - specific values only
Status = Literal["active", "inactive", "pending"]

def set_status(status: Status) -> None:
    pass

set_status("active")   # OK
set_status("unknown")  # Error

# Literal with numbers
Direction = Literal[1, -1, 0]

# Final - cannot be reassigned
MAX_SIZE: Final = 100
API_URL: Final[str] = "https://api.example.com"

class Config:
    DEBUG: Final = False  # Cannot be overridden in subclass
```

## TypeGuard and TypeIs

```python
from typing import TypeGuard, TypeIs

# TypeGuard narrows type in if branch
def is_string_list(val: list[object]) -> TypeGuard[list[str]]:
    return all(isinstance(x, str) for x in val)

def process(items: list[object]) -> None:
    if is_string_list(items):
        # items is now list[str]
        print(items[0].upper())

# TypeIs (Python 3.13+) - stricter, preserves type
def is_int(val: object) -> TypeIs[int]:
    return isinstance(val, int)
```

## Self Type

```python
from typing import Self

class Builder:
    def set_name(self, name: str) -> Self:
        self.name = name
        return self

    def set_value(self, value: int) -> Self:
        self.value = value
        return self

class ExtendedBuilder(Builder):
    def set_extra(self, extra: str) -> Self:
        self.extra = extra
        return self

# Method chaining works with correct types
builder = ExtendedBuilder().set_name("test").set_extra("data")
# builder is ExtendedBuilder, not Builder
```

## Variance

```python
from typing import TypeVar

# Invariant (default) - T must match exactly
T = TypeVar("T")

# Covariant - can use subtypes (for read-only)
T_co = TypeVar("T_co", covariant=True)

# Contravariant - can use supertypes (for write-only)
T_contra = TypeVar("T_contra", contravariant=True)

# Example: Producer is covariant
class Producer[T_co]:
    def get(self) -> T_co: ...

# Producer[Dog] can be used where Producer[Animal] expected

# Example: Consumer is contravariant
class Consumer[T_contra]:
    def accept(self, item: T_contra) -> None: ...

# Consumer[Animal] can be used where Consumer[Dog] expected
```

## Overload

```python
from typing import overload

@overload
def process(value: str) -> str: ...

@overload
def process(value: int) -> int: ...

@overload
def process(value: list[str]) -> list[str]: ...

def process(value: str | int | list[str]) -> str | int | list[str]:
    if isinstance(value, str):
        return value.upper()
    elif isinstance(value, int):
        return value * 2
    else:
        return [v.upper() for v in value]
```

## Common Patterns

### Optional Parameters

```python
def greet(name: str, greeting: str | None = None) -> str:
    if greeting is None:
        greeting = "Hello"
    return f"{greeting}, {name}!"
```

### Factory Functions

```python
from typing import TypeVar

T = TypeVar("T")

def create_instance[T](cls: type[T], **kwargs: Any) -> T:
    return cls(**kwargs)
```

### Decorator with Type Preservation

```python
from typing import ParamSpec, TypeVar, Callable
from functools import wraps

P = ParamSpec("P")
R = TypeVar("R")

def retry(times: int) -> Callable[[Callable[P, R]], Callable[P, R]]:
    def decorator(fn: Callable[P, R]) -> Callable[P, R]:
        @wraps(fn)
        def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
            for _ in range(times):
                try:
                    return fn(*args, **kwargs)
                except Exception:
                    continue
            return fn(*args, **kwargs)
        return wrapper
    return decorator
```

## Best Practices

1. **Use modern syntax** - `list[T]` not `List[T]`, `X | Y` not `Union[X, Y]`
2. **Use PEP 695** for generics in Python 3.12+
3. **Prefer Protocol** over ABC for duck typing
4. **Use Final** for constants
5. **Avoid Any** - use proper types or generics
6. **Add return type** to all functions
7. **Use TypedDict** for structured dicts

## References

- [PEP 484 - Type Hints](https://peps.python.org/pep-0484/)
- [PEP 585 - Type Hinting Generics](https://peps.python.org/pep-0585/)
- [PEP 604 - Union with |](https://peps.python.org/pep-0604/)
- [PEP 695 - Type Parameter Syntax](https://peps.python.org/pep-0695/)
- [typing module docs](https://docs.python.org/3/library/typing.html)
