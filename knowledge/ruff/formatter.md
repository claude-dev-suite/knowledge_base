# Ruff - Formatter

> Official Documentation: https://docs.astral.sh/ruff/formatter/

## Overview

`ruff format` is an extremely fast Python code formatter designed as a drop-in replacement for Black. Over 99.9% of Black-formatted code is formatted identically by Ruff. It is orders of magnitude faster and requires no separate installation.

---

## Table of Contents

1. [ruff format vs Black](#ruff-format-vs-black)
2. [Basic Usage](#basic-usage)
3. [Formatter Options](#formatter-options)
4. [skip-magic-trailing-comma](#skip-magic-trailing-comma)
5. [quote-style](#quote-style)
6. [indent-style](#indent-style)
7. [line-ending](#line-ending)
8. [Docstring Code Formatting](#docstring-code-formatting)
9. [Format Suppression](#format-suppression)
10. [Differences from Black](#differences-from-black)
11. [Disabling Conflicting Lint Rules](#disabling-conflicting-lint-rules)
12. [Exit Codes](#exit-codes)

---

## ruff format vs Black

| Feature | ruff format | Black |
|---------|------------|-------|
| Speed | 10-100x faster | Baseline |
| Black compatibility | >99.9% identical output | Reference |
| Configuration | `pyproject.toml` / `ruff.toml` | `pyproject.toml` |
| F-string formatting | Yes (formats expressions inside `{}`) | No |
| Markdown code blocks | Yes (preview mode) | No |
| Fluent chain layout | Yes (preview mode) | No |
| Separate install | No (part of ruff) | Yes |

Both tools share the same core philosophy: opinionated formatting with minimal configuration, making style debates irrelevant.

---

## Basic Usage

```bash
# Format all Python files in current directory (recursive)
ruff format

# Format specific path
ruff format src/
ruff format mymodule.py

# Check if files need formatting (no modification)
ruff format --check

# Show what would change without modifying
ruff format --diff

# Format from stdin
echo "x=1+2" | ruff format -

# Format specific extension
ruff format --extension py src/
```

### In CI Pipelines

```bash
# Returns exit code 1 if any files would be reformatted
ruff format --check
if [ $? -ne 0 ]; then
    echo "Files need formatting. Run 'ruff format' locally."
    exit 1
fi
```

---

## Formatter Options

All formatter options live under `[tool.ruff.format]` (or `[format]` in `ruff.toml`):

```toml
[tool.ruff.format]
quote-style = "double"
indent-style = "space"
skip-magic-trailing-comma = false
line-ending = "auto"
docstring-code-format = false
docstring-code-line-length = "dynamic"
```

---

## skip-magic-trailing-comma

The **magic trailing comma** is a formatting idiom: if you add a trailing comma to the last item in a collection, formatters will keep it expanded across multiple lines even if it fits on one line.

```python
# Without magic trailing comma — formatter may collapse to one line:
x = [1, 2, 3]

# With magic trailing comma — formatter keeps it expanded:
x = [
    1,
    2,
    3,  # <- trailing comma forces multi-line
]
```

### `skip-magic-trailing-comma = false` (default, matches Black)

Respects the trailing comma. Your explicit choice to expand is preserved:

```python
# Input
x = [1, 2, 3,]

# Output (skip-magic-trailing-comma = false)
x = [
    1,
    2,
    3,
]
```

### `skip-magic-trailing-comma = true`

Ignores trailing commas; formats purely based on line length:

```python
# Input
x = [
    1,
    2,
    3,
]

# Output (skip-magic-trailing-comma = true, if fits on one line)
x = [1, 2, 3]
```

```toml
[tool.ruff.format]
skip-magic-trailing-comma = true  # Format like Prettier, not Black
```

---

## quote-style

Controls the preferred quote character for strings.

### `quote-style = "double"` (default)

```python
# Input
x = 'hello'
y = "world"

# Output
x = "hello"
y = "world"
```

### `quote-style = "single"`

```python
# Output
x = 'hello'
y = 'world'
```

### `quote-style = "preserve"`

Keeps the original quotes unchanged:

```python
# Output — unchanged
x = 'hello'
y = "world"
```

### Quote avoidance

Ruff avoids unnecessary escape sequences by using the opposite quote when the string contains the preferred quote character:

```python
# With quote-style = "double"
x = "it's fine"     # no change needed
y = 'say "hello"'   # kept as single to avoid escaping
# would NOT become: y = "say \"hello\""
```

---

## indent-style

Controls indentation character.

### `indent-style = "space"` (default)

Uses spaces for indentation (PEP 8 standard):

```python
def greet(name):
    print(f"Hello, {name}")  # 4 spaces
```

### `indent-style = "tab"`

Uses tabs for indentation:

```python
def greet(name):
	print(f"Hello, {name}")  # 1 tab
```

The `indent-width` setting (default: 4) controls how many spaces constitute one level when `indent-style = "space"`.

```toml
[tool.ruff]
indent-width = 2  # 2-space indentation

[tool.ruff.format]
indent-style = "space"
```

---

## line-ending

Controls line ending characters in formatted output.

| Value | Line Ending | Description |
|-------|-------------|-------------|
| `"auto"` (default) | Detected | Preserves existing line endings |
| `"lf"` | `\n` | Unix/Linux/macOS (recommended for cross-platform) |
| `"cr-lf"` | `\r\n` | Windows |
| `"cr"` | `\r` | Legacy Mac OS |

```toml
[tool.ruff.format]
line-ending = "lf"  # Enforce Unix line endings
```

---

## Docstring Code Formatting

Ruff can format Python code examples inside docstrings. Supports:
- Python doctest format (`>>>`)
- CommonMark fenced code blocks (` ```python `)
- reStructuredText literal blocks and directives

### Enable

```toml
[tool.ruff.format]
docstring-code-format = true
```

### Example

```python
def add(a: int, b: int) -> int:
    """Add two numbers.

    Example:
        >>> add(  1,
        ...       2)
        3

        ```python
        result = add(1,
                     2)
        ```
    """
    return a + b
```

After formatting:

```python
def add(a: int, b: int) -> int:
    """Add two numbers.

    Example:
        >>> add(1, 2)
        3

        ```python
        result = add(1, 2)
        ```
    """
    return a + b
```

### Docstring code line length

```toml
[tool.ruff.format]
docstring-code-format = true
docstring-code-line-length = 72   # Fixed limit for docstring code
# or:
docstring-code-line-length = "dynamic"  # Same as surrounding code (default)
```

---

## Format Suppression

### Disable for a range of code

```python
x = 1
# fmt: off
y   =  2    # this block will not be formatted
z=3
# fmt: on
a = 4  # this will be formatted
```

### Disable for a specific statement

```python
# fmt: skip
matrix = [[1, 0, 0],
          [0, 1, 0],
          [0, 0, 1]]
```

### YAPF compatibility

Ruff also respects `# yapf: disable` and `# yapf: enable` pragmas (treated equivalently to `# fmt: off/on`).

---

## Differences from Black

While 99.9%+ of output is identical, there are intentional differences:

### 1. F-string Expression Formatting

Ruff formats expressions inside f-string `{}` braces:

```python
# Input
x = f"{'hello':>10}"
y = f"{value    +    1}"

# Black output (unchanged)
x = f"{'hello':>10}"
y = f"{value    +    1}"

# Ruff output (formats expressions)
x = f"{'hello':>10}"
y = f"{value + 1}"
```

### 2. Fluent Method Chain Layout (preview)

Ruff in preview mode can format long method chains differently:

```python
# Ruff (preview)
result = (
    df
    .filter(condition)
    .groupby("column")
    .agg({"value": "sum"})
    .reset_index()
)
```

### 3. Markdown Code Block Support (preview)

Ruff can format Python code inside Markdown files (`.md`) when using preview mode. Black does not support this.

### 4. Magic Trailing Comma

Both tools respect magic trailing commas by default, but Ruff provides `skip-magic-trailing-comma = true` to override this — Black does not offer this option.

### 5. Preview Mode Features

Ruff has a `preview = true` flag that enables upcoming formatter features before they stabilize:

```toml
[tool.ruff.format]
preview = true
```

---

## Disabling Conflicting Lint Rules

When using `ruff format`, certain lint rules conflict with the formatter's output. Disable them:

```toml
[tool.ruff.lint]
ignore = [
    # Formatter handles these
    "E101",   # Indentation contains mixed spaces/tabs
    "E111",   # Indentation not multiple of indent-width
    "E114",   # Indentation not multiple of indent-width (comment)
    "E117",   # Over-indented
    "E203",   # Whitespace before ':'
    "E501",   # Line too long (formatter wraps lines)
    "W191",   # Indentation contains tabs

    # isort conflicts (use ruff's isort integration instead)
    # These are only needed if you also run standalone isort

    # Trailing comma conflicts with formatter
    "COM812", # Missing trailing comma
    "COM819", # Prohibited trailing comma

    # Quote conflicts
    "Q000",   # Single quotes over double (or vice versa)
    "Q001",   # Single quote multiline
    "Q002",   # Single quote docstring
    "Q003",   # Avoidable escaped quote

    # String concatenation
    "ISC001", # Implicitly concatenated string literals on one line
    "ISC002", # Implicitly concatenated string literals over continuation
]
```

The canonical list of formatter-conflicting rules can be shown with:

```bash
ruff format --help
```

---

## Exit Codes

| Exit Code | Scenario |
|-----------|----------|
| `0` | Formatting succeeded (files may have been modified) |
| `1` | Files would be reformatted (with `--check`); or `--exit-non-zero-on-format` was set and files were formatted |
| `2` | Configuration error, invalid CLI argument, or internal error |

```bash
# Use in CI to enforce formatting
ruff format --check
echo "Exit code: $?"
# 0 = already formatted
# 1 = needs formatting
# 2 = error
```
