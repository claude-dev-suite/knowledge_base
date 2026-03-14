# Ruff - Configuration

> Official Documentation: https://docs.astral.sh/ruff/configuration/

## Overview

Ruff can be configured via `pyproject.toml`, `ruff.toml`, or `.ruff.toml`. All formats support the same schema. Configuration supports inheritance, file-pattern-specific overrides, and per-rule customization.

---

## Table of Contents

1. [Configuration File Formats](#configuration-file-formats)
2. [File Discovery and Precedence](#file-discovery-and-precedence)
3. [Top-level Settings](#top-level-settings)
4. [Lint Configuration](#lint-configuration)
5. [Rule Selection Patterns](#rule-selection-patterns)
6. [per-file-ignores](#per-file-ignores)
7. [Formatter Configuration](#formatter-configuration)
8. [Plugin-specific Settings](#plugin-specific-settings)
9. [Configuration Inheritance](#configuration-inheritance)
10. [Full Example Configurations](#full-example-configurations)

---

## Configuration File Formats

All three formats are equivalent. Ruff searches for configuration files starting from the current directory and walking up.

### pyproject.toml

```toml
[tool.ruff]
line-length = 88
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I"]
ignore = ["E501"]

[tool.ruff.format]
quote-style = "double"
```

### ruff.toml

```toml
line-length = 88
target-version = "py311"

[lint]
select = ["E", "F", "I"]
ignore = ["E501"]

[format]
quote-style = "double"
```

### .ruff.toml

Identical schema to `ruff.toml`. `.ruff.toml` takes precedence over `ruff.toml` if both exist in the same directory.

---

## File Discovery and Precedence

1. `.ruff.toml` (highest priority per directory)
2. `ruff.toml`
3. `pyproject.toml` with `[tool.ruff]`
4. Inherited via `extend` from parent config

CLI flags always override configuration file settings.

```bash
# CLI overrides config file
ruff check --select E501 --line-length 100 .
```

---

## Top-level Settings

```toml
[tool.ruff]

# Target Python version for syntax and rule compatibility
# Options: "py37" | "py38" | "py39" | "py310" | "py311" | "py312" | "py313" | "py314" | "py315"
target-version = "py311"

# Maximum line length (default: 88, matches Black)
line-length = 88

# Indentation width for indent-based rules (default: 4)
indent-width = 4

# Files/directories to completely exclude from analysis
exclude = [
    ".git",
    ".hg",
    ".mypy_cache",
    ".tox",
    ".venv",
    "venv",
    "__pycache__",
    "_build",
    "buck-out",
    "build",
    "dist",
    "migrations",
    "node_modules",
]

# Additional exclusions beyond defaults
extend-exclude = ["vendor/", "legacy/"]

# File patterns to include (default: *.py, *.pyi, *.ipynb)
include = ["*.py", "*.pyi"]

# Additional file patterns to include
extend-include = ["*.pyc"]

# Respect .gitignore and other ignore files (default: true)
respect-gitignore = true

# Force exclusions even for directly passed files
force-exclude = true

# Show fixes in output (default: false)
show-fixes = true

# Output format
output-format = "concise"  # "text" | "json" | "junit" | "github" | "gitlab" | "pylint" | "rdjson" | "concise"
```

---

## Lint Configuration

All linting-specific settings go under `[tool.ruff.lint]` (or `[lint]` in ruff.toml):

```toml
[tool.ruff.lint]

# Rules to enable (default: ["E4", "E7", "E9", "F"])
select = ["E", "F", "W", "I", "N", "UP", "B"]

# Rules to disable (even if selected)
ignore = [
    "E501",   # line too long (handled by formatter)
    "E203",   # whitespace before ':' (conflicts with Black)
    "W503",   # line break before binary operator (conflicts with Black)
]

# Add rules beyond the selected set (union with select)
extend-select = ["ANN", "S"]

# Which selected rules support auto-fixing (default: ["ALL"])
fixable = ["ALL"]

# Prevent specific rules from being auto-fixed
unfixable = ["F401"]  # Don't auto-remove unused imports (may be re-exports)

# Regex pattern for dummy/ignored variable names (default: "^(_+|(_+[a-zA-Z0-9_]*[a-zA-Z0-9]+_*)|dummy|ignored)$")
dummy-variable-rgx = "^(_+|dummy|unused_).*$"

# Allow unused imports in __init__.py (re-exports pattern)
# Better approach: use extend-per-file-ignores below
```

---

## Rule Selection Patterns

Rules use prefix codes. You can select at multiple levels of granularity:

```toml
[tool.ruff.lint]

# Select entire categories
select = ["E", "F", "W"]

# Select specific sub-categories
select = ["E4", "E7", "E9", "F"]

# Select individual rules
select = ["E501", "F401", "I001"]

# Enable ALL rules (with ignores for conflicts)
select = ["ALL"]
ignore = [
    "ANN101",  # Missing type annotation for self
    "ANN102",  # Missing type annotation for cls
    "D",       # All docstring rules (if not using docstrings)
    "COM812",  # Trailing comma missing (conflicts with formatter)
    "ISC001",  # Implicitly concatenated string (conflicts with formatter)
]
```

### extend-select vs select

```toml
# select REPLACES the default rule set
select = ["E", "F"]  # Only E and F; default rules are gone

# extend-select ADDS to the default rule set
extend-select = ["UP", "B"]  # Default rules + UP and B
```

---

## per-file-ignores

Override rules for specific file patterns:

```toml
[tool.ruff.lint.per-file-ignores]

# Tests can use assert statements and don't need docstrings
"tests/**/*.py" = ["S101", "ANN", "D"]

# __init__.py files often have re-exports (unused imports are intentional)
"**/__init__.py" = ["F401"]

# Migration files generated by Django/Alembic
"**/migrations/*.py" = ["E501", "RUF012"]

# Scripts may use print
"scripts/**/*.py" = ["T201"]

# Conftest can have fixtures that look unused
"conftest.py" = ["F401", "F811"]

# Type stub files
"*.pyi" = ["E302", "E501", "F401"]

# Specific file
"src/legacy.py" = ["ALL"]
```

### extend-per-file-ignores

Add to existing per-file-ignores without replacing them:

```toml
[tool.ruff.lint]
extend-per-file-ignores = {
    "tests/**" = ["S101"],
    "scripts/**" = ["T201"],
}
```

---

## Formatter Configuration

```toml
[tool.ruff.format]

# Quote style: "double" (default, like Black) or "single"
quote-style = "double"

# Indentation: "space" (default) or "tab"
indent-style = "space"

# Skip magic trailing comma (default: false)
# If false: respects trailing commas to prevent collapsing to single line
# If true: ignores trailing commas and formats purely on line length
skip-magic-trailing-comma = false

# Line ending: "auto" (default) | "lf" | "cr-lf" | "cr"
line-ending = "auto"

# Format code examples in docstrings (default: false)
docstring-code-format = true

# Line length limit for code in docstrings
# "dynamic" = same as surrounding code, or integer
docstring-code-line-length = "dynamic"
```

---

## Plugin-specific Settings

Many rule categories have their own sub-configuration:

### isort (I)

```toml
[tool.ruff.lint.isort]
# Force single-line imports
force-single-line = false

# Known first-party packages
known-first-party = ["mypackage", "mylib"]

# Known third-party packages
known-third-party = ["requests", "pandas"]

# Section order
section-order = ["future", "standard-library", "third-party", "first-party", "local-folder"]

# Lines between sections
lines-between-sections = 1

# Lines after imports
lines-after-imports = 2

# Force sort within sections
force-sort-within-sections = false

# Split on trailing comma
split-on-trailing-comma = false
```

### pydocstyle (D)

```toml
[tool.ruff.lint.pydocstyle]
# Convention: "google" | "numpy" | "pep257"
convention = "google"
```

### flake8-quotes (Q)

```toml
[tool.ruff.lint.flake8-quotes]
docstring-quotes = "double"
inline-quotes = "double"
multiline-quotes = "double"
```

### flake8-builtins (A)

```toml
[tool.ruff.lint.flake8-builtins]
builtins-ignorelist = ["id", "type"]
```

### flake8-type-checking (TCH)

```toml
[tool.ruff.lint.flake8-type-checking]
# Move runtime-evaluated annotations to TYPE_CHECKING block
runtime-evaluated-base-classes = ["pydantic.BaseModel", "sqlalchemy.orm.DeclarativeBase"]
```

### mccabe (C90) - complexity

```toml
[tool.ruff.lint.mccabe]
# Maximum allowed cyclomatic complexity (default: 10)
max-complexity = 12
```

### pylint (PL)

```toml
[tool.ruff.lint.pylint]
max-args = 5
max-returns = 3
max-branches = 12
max-statements = 50
```

---

## Configuration Inheritance

Use `extend` to inherit from a base configuration:

```toml
# In a sub-package's pyproject.toml or ruff.toml
extend = "../../pyproject.toml"  # Path to parent config

[tool.ruff.lint]
# These are merged with the parent's settings
extend-ignore = ["ANN"]
```

This is useful in monorepos where each package can override specific rules while inheriting the base config.

---

## Full Example Configurations

### Library / Open Source Project

```toml
[tool.ruff]
target-version = "py310"
line-length = 88

[tool.ruff.lint]
select = [
    "E",    # pycodestyle errors
    "W",    # pycodestyle warnings
    "F",    # pyflakes
    "I",    # isort
    "N",    # pep8-naming
    "UP",   # pyupgrade
    "B",    # flake8-bugbear
    "ANN",  # flake8-annotations
    "D",    # pydocstyle
    "RUF",  # ruff-specific rules
]
ignore = [
    "E501",    # line too long (handled by formatter)
    "ANN101",  # Missing type annotation for self
    "ANN102",  # Missing type annotation for cls
    "D203",    # 1 blank line required before class docstring (conflicts with D211)
    "D213",    # Multi-line docstring summary should start at the second line (conflicts with D212)
    "COM812",  # Trailing comma missing (formatter conflict)
]

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["ANN", "D", "S101"]
"**/__init__.py" = ["F401", "D104"]

[tool.ruff.lint.pydocstyle]
convention = "google"

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
skip-magic-trailing-comma = false

[tool.ruff.lint.isort]
known-first-party = ["mylib"]
```

### FastAPI / Web Application

```toml
[tool.ruff]
target-version = "py311"
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "W", "I", "UP", "B", "S", "N"]
ignore = [
    "E501",
    "S101",   # Allow assert in tests
    "B008",   # Do not perform function call in default arguments (FastAPI uses Depends())
]
fixable = ["ALL"]
unfixable = ["F401"]

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101", "S105", "S106"]
"alembic/**" = ["E501"]

[tool.ruff.format]
quote-style = "double"
```

### Data Science / Notebooks

```toml
[tool.ruff]
target-version = "py310"
line-length = 100
extend-include = ["*.ipynb"]

[tool.ruff.lint]
select = ["E", "F", "I", "UP"]
ignore = [
    "E501",
    "F401",   # Unused imports common in notebooks
    "E402",   # Module-level imports not at top of file
]
per-file-ignores = { "*.ipynb" = ["E402", "F401"] }

[tool.ruff.format]
skip-magic-trailing-comma = true
```

### Strict Production Configuration

```toml
[tool.ruff]
target-version = "py312"
line-length = 88

[tool.ruff.lint]
select = ["ALL"]
ignore = [
    # Formatter conflicts
    "COM812", "COM819",
    "D206", "D300",
    "E111", "E114", "E117",
    "ISC001", "ISC002",
    "Q000", "Q001", "Q002", "Q003",
    "W191",
    # Docstring style choice
    "D203",  # conflicts with D211
    "D213",  # conflicts with D212
    # Overly strict
    "ANN101", "ANN102",
    "FIX002",  # Allow TODO comments
    "TD003",   # Missing issue link for TODO
]

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["D", "ANN", "S101", "PLR2004"]
"**/migrations/**" = ["D", "ANN", "E501", "RUF012"]

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
skip-magic-trailing-comma = false
line-ending = "lf"
```
