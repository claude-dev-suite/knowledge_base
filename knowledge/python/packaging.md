# Python Package Management

Complete guide to Python package management with uv, poetry, and pyproject.toml.

## Tool Comparison

| Tool | Speed | Lock Files | PEP 621 | Best For |
|------|-------|------------|---------|----------|
| **uv** | Fastest (10-100x pip) | Yes | Full | New projects |
| Poetry | Good | Yes (strong) | Partial | Complex deps |
| PDM | Fast | Yes | Full | Standards compliance |
| pip | Baseline | No | N/A | Basic installation |

## uv (Recommended)

uv is an extremely fast Python package manager written in Rust.

### Installation

```bash
# Unix/macOS
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"

# pip
pip install uv
```

### Project Management

```bash
# Create new project
uv init my-project
cd my-project

# Initialize in existing directory
uv init

# Add dependencies
uv add requests fastapi pydantic

# Add dev dependencies
uv add --dev pytest ruff mypy

# Add optional dependency group
uv add --optional docs sphinx

# Remove dependency
uv remove requests

# Update dependencies
uv lock --upgrade
uv lock --upgrade-package requests

# Sync environment from lock
uv sync

# Sync with dev dependencies
uv sync --dev

# Run command in environment
uv run python main.py
uv run pytest
uv run ruff check .
```

### Python Version Management

```bash
# Install Python version
uv python install 3.12
uv python install 3.11 3.12 3.13

# List installed versions
uv python list

# Pin project Python version
uv python pin 3.12

# Run with specific version
uv run --python 3.12 python main.py
```

### Tool Management

```bash
# Install global tool
uv tool install ruff
uv tool install mypy

# Run tool without install
uvx ruff check .
uvx black .

# List installed tools
uv tool list

# Update tool
uv tool upgrade ruff
```

### uv as pip Replacement

```bash
# pip-compatible commands
uv pip install requests
uv pip install -r requirements.txt
uv pip uninstall requests
uv pip freeze
uv pip list
uv pip show requests

# Compile requirements
uv pip compile requirements.in -o requirements.txt
```

## pyproject.toml (PEP 621)

### Complete Template

```toml
[project]
name = "my-project"
version = "1.0.0"
description = "A Python project"
readme = "README.md"
license = {text = "MIT"}
requires-python = ">=3.10"
authors = [
    {name = "Your Name", email = "you@example.com"}
]
keywords = ["python", "example"]
classifiers = [
    "Development Status :: 4 - Beta",
    "Intended Audience :: Developers",
    "License :: OSI Approved :: MIT License",
    "Programming Language :: Python :: 3.10",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
]

dependencies = [
    "fastapi>=0.115.0",
    "pydantic>=2.0",
    "sqlalchemy>=2.0",
    "httpx>=0.27",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "pytest-cov>=4.0",
    "ruff>=0.14",
    "mypy>=1.0",
]
docs = [
    "sphinx>=7.0",
    "sphinx-rtd-theme>=2.0",
]

[project.scripts]
my-cli = "my_project.cli:main"

[project.urls]
Homepage = "https://github.com/user/my-project"
Documentation = "https://my-project.readthedocs.io"
Repository = "https://github.com/user/my-project.git"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/my_project"]

# uv specific
[tool.uv]
dev-dependencies = [
    "pytest>=8.0",
    "ruff>=0.14",
]

# Tool configurations
[tool.ruff]
line-length = 88
target-version = "py312"

[tool.ruff.lint]
select = ["E", "W", "F", "I", "B", "C4", "UP", "S"]

[tool.mypy]
python_version = "3.12"
strict = true

[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"
```

## Project Structure

### src Layout (Recommended)

```
my-project/
├── src/
│   └── my_package/
│       ├── __init__.py
│       ├── __main__.py     # python -m my_package
│       ├── core.py
│       ├── cli.py
│       └── utils/
│           └── __init__.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── test_core.py
│   └── test_cli.py
├── docs/
│   └── index.md
├── pyproject.toml
├── uv.lock
├── README.md
└── LICENSE
```

**Benefits of src layout:**
- Tests run against installed package
- Prevents accidental imports from working directory
- Cleaner package builds

### __main__.py

```python
# src/my_package/__main__.py
from my_package.cli import app

if __name__ == "__main__":
    app()
```

Run with: `python -m my_package` or `uv run python -m my_package`

## Dependency Versioning

```toml
# Exact version (pinned)
"package==1.2.3"

# Minimum version (apps)
"package>=1.2.0"

# Compatible version (libraries)
"package>=1.2,<2.0"
"package~=1.2"  # Same as >=1.2,<2.0

# Exclude specific versions
"package>=1.0,!=1.5.0"

# Pre-release versions
"package>=1.0a1"

# Local path
"package @ file:///path/to/package"

# Git
"package @ git+https://github.com/user/repo.git@v1.0"
```

## Virtual Environments

```bash
# uv (recommended)
uv venv
source .venv/bin/activate      # Unix
.venv\Scripts\activate         # Windows

# Standard library
python -m venv .venv
source .venv/bin/activate

# Check active environment
which python                    # Unix
where python                    # Windows
```

## Lock Files

### Purpose

Lock files ensure reproducible builds by pinning exact versions of all dependencies (including transitive).

### uv.lock

```bash
# Generate/update lock file
uv lock

# Update all dependencies
uv lock --upgrade

# Update specific package
uv lock --upgrade-package requests

# Install from lock
uv sync
```

### Committing Lock Files

| Project Type | Commit Lock? |
|--------------|--------------|
| Application | Yes |
| Library | Optional (usually no) |
| CI/CD scripts | Yes |

## Poetry (Alternative)

### Installation

```bash
curl -sSL https://install.python-poetry.org | python3 -
```

### Usage

```bash
# Create new project
poetry new my-project

# Initialize in existing directory
poetry init

# Add dependencies
poetry add requests
poetry add --group dev pytest

# Install dependencies
poetry install

# Run command
poetry run python main.py
poetry run pytest

# Shell (activate venv)
poetry shell

# Build package
poetry build

# Publish
poetry publish
```

### poetry pyproject.toml

```toml
[tool.poetry]
name = "my-project"
version = "1.0.0"
description = ""
authors = ["Your Name <you@example.com>"]
readme = "README.md"

[tool.poetry.dependencies]
python = "^3.10"
fastapi = "^0.115.0"
pydantic = "^2.0"

[tool.poetry.group.dev.dependencies]
pytest = "^8.0"
ruff = "^0.14"
mypy = "^1.0"

[tool.poetry.scripts]
my-cli = "my_package.cli:main"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

## Migration Guides

### From requirements.txt to uv

```bash
# Simple migration
uv init
uv add $(cat requirements.txt | grep -v "^#" | grep -v "^$" | cut -d'=' -f1 | tr '\n' ' ')
rm requirements.txt
```

### From Poetry to uv

```bash
# Export dependencies
poetry export -f requirements.txt --without-hashes > requirements.txt

# Create uv project
uv init
uv add $(cat requirements.txt | cut -d';' -f1 | cut -d'=' -f1 | tr '\n' ' ')
rm requirements.txt poetry.lock pyproject.toml
# Re-add tool configs to pyproject.toml
```

### From pip to uv

```bash
# Direct replacement
alias pip='uv pip'

# Or use uv project management
uv init
uv add <packages>
```

## CI/CD Integration

### GitHub Actions

```yaml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v4

      - name: Set up Python
        run: uv python install 3.12

      - name: Install dependencies
        run: uv sync --dev

      - name: Lint
        run: uv run ruff check .

      - name: Type check
        run: uv run mypy src/

      - name: Test
        run: uv run pytest --cov
```

## Best Practices

1. **Use uv** for new projects (fastest, modern)
2. **Use src layout** for packages
3. **Always use virtual environments**
4. **Commit lock files** for applications
5. **Use dependency groups** (dev, docs, test)
6. **Pin Python version** in pyproject.toml
7. **Use PEP 621** format in pyproject.toml

## References

- [uv documentation](https://docs.astral.sh/uv/)
- [Poetry documentation](https://python-poetry.org/docs/)
- [PEP 621 - Project metadata](https://peps.python.org/pep-0621/)
- [Python Packaging Guide](https://packaging.python.org/)
