# Python Code Quality

Complete guide to Python static analysis with ruff, mypy, and pyright.

## Tool Stack

| Tool | Purpose | Speed | Recommendation |
|------|---------|-------|----------------|
| **ruff** | Linting + Formatting | Fastest | Primary tool |
| **mypy** | Type checking | Good | CI gate |
| **pyright** | Type checking | Fast | IDE (Pylance) |

**Recommended setup**: ruff + pyright (IDE) + mypy (CI)

## Ruff

Ruff is an extremely fast Python linter and formatter.

### Installation

```bash
uv add --dev ruff
# Or globally
uv tool install ruff
```

### Configuration

```toml
# pyproject.toml
[tool.ruff]
line-length = 88
target-version = "py312"
src = ["src", "tests"]
exclude = [
    ".git",
    ".venv",
    "__pycache__",
    "build",
    "dist",
]

[tool.ruff.lint]
select = [
    "E",      # pycodestyle errors
    "W",      # pycodestyle warnings
    "F",      # Pyflakes
    "I",      # isort
    "B",      # flake8-bugbear
    "C4",     # flake8-comprehensions
    "UP",     # pyupgrade
    "ARG",    # flake8-unused-arguments
    "SIM",    # flake8-simplify
    "TCH",    # flake8-type-checking
    "PTH",    # flake8-use-pathlib
    "ERA",    # eradicate (commented code)
    "PL",     # pylint
    "RUF",    # Ruff-specific
    "S",      # flake8-bandit (security)
    "PERF",   # perflint
]
ignore = [
    "E501",   # line too long (formatter handles)
    "PLR0913", # too many arguments
]

[tool.ruff.lint.per-file-ignores]
"tests/*" = ["S101", "ARG001", "PLR2004"]
"__init__.py" = ["F401"]
"conftest.py" = ["ARG001"]

[tool.ruff.lint.isort]
known-first-party = ["my_package"]
force-single-line = false
combine-as-imports = true

[tool.ruff.lint.pylint]
max-args = 8

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
skip-magic-trailing-comma = false
docstring-code-format = true
```

### Commands

```bash
# Lint
ruff check .
ruff check src/ tests/

# Lint with auto-fix
ruff check --fix .

# Format
ruff format .
ruff format --check .  # Check only

# Combined (CI)
ruff check . && ruff format --check .

# Show what would change
ruff check --diff .
ruff format --diff .
```

### Common Rules

| Rule | Description |
|------|-------------|
| `E` | pycodestyle errors |
| `W` | pycodestyle warnings |
| `F` | Pyflakes (undefined names, unused imports) |
| `I` | isort (import sorting) |
| `B` | flake8-bugbear (common bugs) |
| `S` | flake8-bandit (security) |
| `C4` | flake8-comprehensions |
| `UP` | pyupgrade (modern syntax) |
| `RUF` | Ruff-specific rules |

### Inline Ignores

```python
# Ignore specific rule
x = 1  # noqa: E501

# Ignore multiple rules
x = eval(code)  # noqa: S307, B301

# Ignore for block (ruff)
# ruff: noqa: S101
assert condition

# Type: ignore equivalent
x: int = "string"  # type: ignore[assignment]
```

## mypy

### Installation

```bash
uv add --dev mypy
# With stubs
uv add --dev types-requests types-redis
```

### Configuration

```toml
# pyproject.toml
[tool.mypy]
python_version = "3.12"
strict = true
warn_return_any = true
warn_unused_ignores = true
warn_redundant_casts = true
warn_unused_configs = true
disallow_untyped_defs = true
disallow_incomplete_defs = true
check_untyped_defs = true
disallow_untyped_decorators = true
no_implicit_optional = true
strict_equality = true
show_error_codes = true
show_column_numbers = true
pretty = true

# Error codes to enable
enable_error_code = [
    "ignore-without-code",
    "truthy-bool",
    "redundant-expr",
]

# Per-module overrides
[[tool.mypy.overrides]]
module = "tests.*"
disallow_untyped_defs = false
disallow_incomplete_defs = false

[[tool.mypy.overrides]]
module = "third_party.*"
ignore_missing_imports = true

[[tool.mypy.overrides]]
module = "legacy.*"
ignore_errors = true
```

### Commands

```bash
# Check
mypy src/
mypy src/ tests/

# Strict mode
mypy --strict src/

# Generate stubs
stubgen -p my_package -o stubs/

# Daemon mode (faster)
dmypy start
dmypy run -- src/
dmypy stop
```

### Common Error Codes

| Code | Description |
|------|-------------|
| `assignment` | Incompatible types in assignment |
| `arg-type` | Wrong argument type |
| `return-value` | Wrong return type |
| `call-overload` | No matching overload |
| `union-attr` | Attribute access on union type |
| `no-untyped-def` | Function without type annotations |
| `name-defined` | Name not defined |
| `import` | Import error |

### Handling Errors

```python
# Ignore specific error
x: int = "string"  # type: ignore[assignment]

# Ignore with comment
result = untyped_function()  # type: ignore[no-untyped-call]  # TODO: add types

# Cast (use sparingly)
from typing import cast
x = cast(int, unknown_value)

# Assert type
from typing import assert_type
assert_type(x, int)  # Fails if x is not int

# reveal_type (debugging)
reveal_type(x)  # Shows type in mypy output
```

## pyright

### Configuration

```toml
# pyproject.toml
[tool.pyright]
include = ["src"]
exclude = ["**/node_modules", "**/__pycache__", ".venv"]
typeCheckingMode = "strict"
pythonVersion = "3.12"
pythonPlatform = "Linux"

reportMissingImports = true
reportMissingTypeStubs = false
reportUnusedImport = true
reportUnusedVariable = true
reportUnusedClass = true
reportUnusedFunction = true
reportDuplicateImport = true
reportPrivateUsage = true
reportConstantRedefinition = true
reportIncompatibleMethodOverride = true
reportIncompatibleVariableOverride = true
reportUntypedFunctionDecorator = true
```

Or JSON (pyrightconfig.json):
```json
{
  "include": ["src"],
  "typeCheckingMode": "strict",
  "pythonVersion": "3.12"
}
```

### Commands

```bash
# Install
uv add --dev pyright

# Check
pyright
pyright src/

# With specific config
pyright --project pyrightconfig.json
```

## Type Stubs

### Installing Stubs

```bash
# Common stubs
uv add --dev types-requests
uv add --dev types-redis
uv add --dev types-PyYAML
uv add --dev types-python-dateutil
uv add --dev types-setuptools

# Multiple stubs
uv add --dev types-requests types-redis types-PyYAML
```

### Creating Stubs

```python
# my_package/stubs/external_lib.pyi

def external_function(x: int) -> str: ...

class ExternalClass:
    value: int
    def method(self, arg: str) -> None: ...
```

Add to pyproject.toml:
```toml
[tool.mypy]
mypy_path = "stubs"
```

## Pre-commit Integration

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.14.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.14.0
    hooks:
      - id: mypy
        additional_dependencies:
          - types-requests
          - types-redis
        args: [--strict]
```

```bash
# Install
uv add --dev pre-commit
pre-commit install

# Run manually
pre-commit run --all-files
```

## CI/CD Integration

### GitHub Actions

```yaml
name: Quality

on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v4

      - name: Install dependencies
        run: uv sync --dev

      - name: Lint
        run: uv run ruff check .

      - name: Format check
        run: uv run ruff format --check .

      - name: Type check (mypy)
        run: uv run mypy src/

      - name: Type check (pyright)
        run: uv run pyright
```

## Strict Mode Migration

### Phase 1: Basic types

```toml
[tool.mypy]
check_untyped_defs = true
```

### Phase 2: Require annotations

```toml
[tool.mypy]
disallow_untyped_defs = true
disallow_incomplete_defs = true
```

### Phase 3: Full strict

```toml
[tool.mypy]
strict = true
```

### Gradual migration

```toml
# Enable strict globally
[tool.mypy]
strict = true

# Relax for legacy code
[[tool.mypy.overrides]]
module = "legacy.*"
disallow_untyped_defs = false
ignore_errors = true

# Strict for new code
[[tool.mypy.overrides]]
module = "new_feature.*"
# Inherits strict from global
```

## Common Issues and Solutions

### "Module has no attribute X"

```python
# Problem: Missing stubs
import requests
requests.get(...)  # Error: Module has no attribute "get"

# Solution: Install stubs
# uv add --dev types-requests
```

### "Incompatible types in assignment"

```python
# Problem
x: int = "string"  # Error

# Solution: Fix type or use Union
x: int | str = "string"
```

### "Argument of type X is not assignable to parameter of type Y"

```python
# Problem
def process(items: list[str]) -> None: ...
process(["a", 1])  # Error: int not assignable to str

# Solution: Fix data or use broader type
def process(items: list[str | int]) -> None: ...
```

### "Cannot infer type"

```python
# Problem
x = []  # Cannot infer type

# Solution: Add annotation
x: list[int] = []
```

## Best Practices

1. **Use ruff** for linting and formatting (replaces black, isort, flake8)
2. **Enable strict mode** in mypy for new projects
3. **Add type stubs** for untyped dependencies
4. **Use pre-commit** for automatic checking
5. **Run in CI** to catch regressions
6. **Migrate gradually** for existing projects
7. **Use specific ignores** with error codes

## References

- [Ruff documentation](https://docs.astral.sh/ruff/)
- [mypy documentation](https://mypy.readthedocs.io/)
- [pyright documentation](https://microsoft.github.io/pyright/)
- [typeshed (type stubs)](https://github.com/python/typeshed)
