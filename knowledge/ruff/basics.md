# Ruff - Basics

> Official Documentation: https://docs.astral.sh/ruff/

## Overview

Ruff is an extremely fast Python linter and code formatter written in Rust. It is 10-100x faster than existing tools like Flake8 or Black, implements over 800 rules, and can replace Flake8, isort, Black, pydocstyle, pyupgrade, autoflake, and more — all in a single tool.

---

## Table of Contents

1. [What is Ruff](#what-is-ruff)
2. [Installation](#installation)
3. [ruff check - Linting](#ruff-check---linting)
4. [ruff format - Formatting](#ruff-format---formatting)
5. [Basic pyproject.toml Integration](#basic-pyprojecttoml-integration)
6. [Exit Codes](#exit-codes)
7. [Pre-commit Integration](#pre-commit-integration)
8. [Editor Integration](#editor-integration)

---

## What is Ruff

Ruff is a modern Python tooling replacement written in Rust by Astral (the team behind `uv`).

Key characteristics:
- **Speed**: 10-100x faster than Flake8, Black, isort individually
- **Single tool**: Replaces Flake8 + dozens of plugins + isort + Black + pyupgrade + autoflake
- **800+ rules**: Native implementations of pycodestyle, pyflakes, isort, pep8-naming, pyupgrade, flake8-bugbear, flake8-bandit, and more
- **Auto-fix**: Many rules include automatic fixes
- **Caching**: Built-in caching prevents re-analyzing unchanged files
- **Drop-in compatible**: Respects `# noqa`, `# type: ignore`, `# flake8: noqa`, isort action comments
- **Editor support**: First-party VS Code extension, LSP support

What Ruff can replace:
- Flake8 (and most of its plugins)
- Black (via `ruff format`)
- isort
- pydocstyle
- pyupgrade
- autoflake
- bandit (subset)

---

## Installation

### via pip

```bash
pip install ruff
```

### via uv (recommended for projects)

```bash
uv add --dev ruff
# or globally:
uv tool install ruff
```

### via pipx

```bash
pipx install ruff
```

### via Homebrew (macOS/Linux)

```bash
brew install ruff
```

### Verify installation

```bash
ruff --version
# ruff 0.8.x
```

---

## ruff check - Linting

The primary linting command. Analyzes Python files for errors, style issues, and potential bugs.

### Basic usage

```bash
# Lint current directory (recursive)
ruff check

# Lint specific file or directory
ruff check src/
ruff check mymodule.py

# Lint multiple paths
ruff check src/ tests/
```

### Auto-fix violations

```bash
# Apply safe fixes automatically
ruff check --fix

# Apply safe fixes and show what was fixed
ruff check --fix --show-fixes

# Apply unsafe fixes too (use with caution)
ruff check --fix --unsafe-fixes

# Preview what would be fixed without applying
ruff check --diff
```

### Select specific rules

```bash
# Only check for unused imports (F401)
ruff check --select F401

# Enable a whole category
ruff check --select E,F

# Check everything
ruff check --select ALL

# Ignore specific rules
ruff check --ignore E501,W503
```

### Watch mode

```bash
# Re-lint on file changes
ruff check --watch
ruff check src/ --watch
```

### Show statistics

```bash
ruff check --statistics
```

Output example:
```
28  E501  Line too long (88 > 79 characters)
12  F401  'os' imported but unused
 5  F841  Local variable is assigned but never used
```

### Output formats

```bash
# Default: grouped by file
ruff check

# JSON output (for CI integrations)
ruff check --output-format json

# GitHub Actions annotations
ruff check --output-format github

# Concise (one error per line)
ruff check --output-format concise
```

---

## ruff format - Formatting

The code formatter. Drop-in replacement for Black.

### Basic usage

```bash
# Format all Python files in current directory
ruff format

# Format specific path
ruff format src/
ruff format mymodule.py

# Preview formatting without applying changes
ruff format --check
ruff format --diff

# Format stdin
echo "x=1" | ruff format -
```

### Check if files are already formatted (CI)

```bash
# Exit code 1 if any files would be reformatted
ruff format --check
```

---

## Basic pyproject.toml Integration

Add Ruff configuration to your existing `pyproject.toml`:

```toml
[tool.ruff]
# Python target version
target-version = "py311"

# Line length (default 88, matches Black)
line-length = 88

# Directories to exclude
exclude = [
    ".git",
    ".venv",
    "__pycache__",
    "migrations",
    "dist",
    "build",
]

[tool.ruff.lint]
# Enable rule categories
select = ["E", "F", "W", "I", "UP"]

# Ignore specific rules
ignore = ["E501"]  # line too long (handled by formatter)

# Allow automatic fixes for all enabled rules
fixable = ["ALL"]

[tool.ruff.format]
# Use double quotes (default)
quote-style = "double"

# Use spaces for indentation (default)
indent-style = "space"
```

### Standalone ruff.toml

```toml
# ruff.toml (no [tool.ruff] prefix needed)
target-version = "py311"
line-length = 88

[lint]
select = ["E", "F", "W", "I"]
ignore = ["E501"]

[format]
quote-style = "double"
```

### Minimal example for a new project

```toml
[tool.ruff]
line-length = 88

[tool.ruff.lint]
select = ["E", "F", "I"]
```

---

## Exit Codes

### ruff check

| Exit Code | Meaning |
|-----------|---------|
| `0` | No violations found (or all were fixed) |
| `1` | Violations found (unfixed) |
| `2` | Abnormal termination: invalid config, CLI error, or internal error |

Modifiers:
- `--exit-zero`: Always return 0 (useful in CI where you want to report but not fail)
- `--exit-non-zero-on-fix`: Return exit code 1 if any fixes were applied

```bash
# Always succeed (just report)
ruff check --exit-zero

# Fail if fixes were applied (enforce clean commits)
ruff check --fix --exit-non-zero-on-fix
```

### ruff format

| Exit Code | Meaning |
|-----------|---------|
| `0` | Formatting successful |
| `1` | Files would be reformatted (with `--check`); or abnormal termination with `--exit-non-zero-on-format` |
| `2` | Configuration error or CLI error |

---

## Pre-commit Integration

Add Ruff to `.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.8.0  # Use the latest version
    hooks:
      # Run the linter
      - id: ruff
        args: [--fix]
      # Run the formatter
      - id: ruff-format
```

Install and run:

```bash
pre-commit install
pre-commit run --all-files
```

---

## Editor Integration

### VS Code

Install the official [Ruff extension](https://marketplace.visualstudio.com/items?itemName=charliermarsh.ruff).

```json
// .vscode/settings.json
{
    "[python]": {
        "editor.formatOnSave": true,
        "editor.defaultFormatter": "charliermarsh.ruff",
        "editor.codeActionsOnSave": {
            "source.fixAll.ruff": "explicit",
            "source.organizeImports.ruff": "explicit"
        }
    }
}
```

### PyCharm / IntelliJ

Use the [Ruff plugin](https://plugins.jetbrains.com/plugin/20574-ruff) from the marketplace.

### Neovim

Via `nvim-lspconfig` with `ruff-lsp` or via `null-ls`/`conform.nvim`:

```lua
require('lspconfig').ruff.setup({})
```

---

## Quick Command Reference

```bash
# Check for issues
ruff check .

# Check and fix safe issues
ruff check . --fix

# Format code
ruff format .

# Check formatting without modifying
ruff format . --check

# Show all available rules
ruff rule --all

# Show details for a specific rule
ruff rule E501

# Show what rules are enabled
ruff check --show-settings

# Suppress all violations in a file
echo "# ruff: noqa" >> myfile.py

# Suppress a specific violation on a line
x = 1  # noqa: F841

# Add noqa comments automatically to all violations
ruff check --add-noqa
```
