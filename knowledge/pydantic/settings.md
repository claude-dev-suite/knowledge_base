# Pydantic - Settings

> Official Documentation: https://docs.pydantic.dev/latest/concepts/pydantic_settings/

## Overview

`pydantic-settings` extends Pydantic with a `BaseSettings` class for managing application configuration. It automatically reads values from environment variables, `.env` files, secret files, CLI arguments, and more. It provides type validation and the same Pydantic model API, making configuration clean, documented, and easy to test.

---

## Table of Contents

1. [Installation & Basic Usage](#installation--basic-usage)
2. [Environment Variables](#environment-variables)
3. [.env File Support](#env-file-support)
4. [Secret Files](#secret-files)
5. [Nested Settings](#nested-settings)
6. [Source Priority & Custom Sources](#source-priority--custom-sources)
7. [Configuration File Sources](#configuration-file-sources)
8. [CLI Support](#cli-support)
9. [Testing & Overrides](#testing--overrides)
10. [Common Patterns](#common-patterns)

---

## Installation & Basic Usage

```bash
pip install pydantic-settings
```

### Minimal Example

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    app_name: str = "My Application"
    debug: bool = False
    port: int = 8000
    secret_key: str   # Required — must be set via env var or other source

settings = Settings()
print(settings.app_name)   # From env var APP_NAME or default
print(settings.port)       # From env var PORT or default 8000
```

Environment variable names match field names **case-insensitively** by default:
- Field `port` → env var `PORT` or `port`
- Field `secret_key` → env var `SECRET_KEY` or `secret_key`

---

## Environment Variables

### Basic Configuration

```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import Field

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_prefix="MYAPP_",          # Prefix all env vars
        case_sensitive=False,         # Case-insensitive matching (default)
        populate_by_name=True,        # Allow both field name and alias
    )

    # With prefix: reads from MYAPP_HOST
    host: str = "localhost"
    port: int = 8080
    debug: bool = False

    # Custom env var name via alias
    api_key: str = Field(alias="MY_API_KEY")
    # validation_alias applies only for env reading
    db_url: str = Field(validation_alias="DATABASE_URL")
```

### Prefix Behavior

```python
class AppSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="APP_")

    host: str = "localhost"       # reads APP_HOST
    port: int = 8000              # reads APP_PORT

    # Note: alias ignores prefix
    api_key: str = Field(alias="EXTERNAL_API_KEY")  # reads EXTERNAL_API_KEY (no prefix)
```

### Case Sensitivity

```python
# Default: case-insensitive — PORT, port, Port all work
class CaseInsensitiveSettings(BaseSettings):
    port: int = 8080

# Case-sensitive (note: Windows always case-insensitive)
class CaseSensitiveSettings(BaseSettings):
    model_config = SettingsConfigDict(case_sensitive=True)
    PORT: int = 8080  # Only reads PORT exactly
```

### Alias Choices (Multiple Env Var Names)

```python
from pydantic import AliasChoices, Field
from pydantic_settings import BaseSettings

class DatabaseSettings(BaseSettings):
    # Accept DATABASE_URL or DB_URL
    url: str = Field(
        validation_alias=AliasChoices("DATABASE_URL", "DB_URL", "db_url")
    )
    # Accept nested path: {"db": {"host": "..."}}
    from pydantic import AliasPath
    host: str = Field(
        validation_alias=AliasPath("db", "host"),
        default="localhost"
    )
```

---

## .env File Support

### Basic .env Usage

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8"
    )

    database_url: str
    secret_key: str
    debug: bool = False
    port: int = 8000
```

`.env` file:
```dotenv
DATABASE_URL=postgresql://user:pass@localhost:5432/mydb
SECRET_KEY=super-secret-production-key
DEBUG=false
PORT=8000
```

### Multiple .env Files (Priority)

Later files override earlier files:

```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=(".env", ".env.local", ".env.production"),
        # .env.production values override .env.local which overrides .env
    )
    database_url: str
```

### Override at Instantiation

```python
# Override env_file per-instance (useful for testing)
settings = Settings(_env_file="tests/.env.test", _env_file_encoding="utf-8")
```

### Environment Variables Override .env

Environment variables set in the shell always take precedence over `.env` file values:

```bash
export DATABASE_URL=postgresql://prod-host/prod-db  # Overrides .env
python app.py
```

---

## Secret Files

Store sensitive values in files where the **filename is the field name**:

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        secrets_dir="/run/secrets"
    )

    database_password: str   # reads /run/secrets/database_password
    api_token: str           # reads /run/secrets/api_token
```

File contents are stripped of leading/trailing whitespace.

### Multiple Secret Directories

```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        secrets_dir=("/run/secrets", "/vault/secrets")
        # Later paths override earlier paths
    )
    my_secret: str
```

### Docker Secrets Integration

```bash
# Create Docker secret
printf "mysecretvalue" | docker secret create database_password -

# In your service, secrets are mounted at /run/secrets/
```

```python
class ProductionSettings(BaseSettings):
    model_config = SettingsConfigDict(secrets_dir="/run/secrets")
    database_password: str
    jwt_secret: str
```

### Priority Order (highest to lowest)

1. Init kwargs: `Settings(field=value)`
2. Environment variables
3. `.env` file values
4. Secret files
5. Field defaults

---

## Nested Settings

### Nested Pydantic Models

```python
from pydantic import BaseModel
from pydantic_settings import BaseSettings, SettingsConfigDict

class DatabaseConfig(BaseModel):
    host: str = "localhost"
    port: int = 5432
    name: str = "mydb"
    user: str = "postgres"
    password: str = ""

class RedisConfig(BaseModel):
    host: str = "localhost"
    port: int = 6379
    db: int = 0

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_nested_delimiter="__"  # Use __ as separator for nested fields
    )

    database: DatabaseConfig = DatabaseConfig()
    redis: RedisConfig = RedisConfig()
    app_name: str = "myapp"
```

Environment variables for nested models:

```bash
DATABASE__HOST=prod-db.example.com
DATABASE__PORT=5432
DATABASE__NAME=production
DATABASE__PASSWORD=secretpass

REDIS__HOST=redis.example.com
REDIS__PORT=6380
```

JSON format also works:

```bash
DATABASE='{"host": "prod-db.example.com", "port": 5432, "name": "production"}'
```

### Nested Partial Updates

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_nested_delimiter="__",
        nested_model_default_partial_update=True,  # Only update specified fields
    )
    database: DatabaseConfig = DatabaseConfig(host="localhost", port=5432)

# With partial update: setting DATABASE__HOST only updates host,
# preserving port=5432 from default
# os.environ["DATABASE__HOST"] = "prod-db.example.com"
```

### Limiting Nesting Depth

```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_nested_delimiter="_",
        env_nested_max_split=1,    # Split at most once on "_"
        # Prevents "my_field_name" from being misinterpreted as nested
    )
    my_field_name: str  # reads MY_FIELD_NAME, not MY.FIELD.NAME
```

---

## Source Priority & Custom Sources

### Customizing Source Order

```python
from typing import Tuple, Type
from pydantic_settings import (
    BaseSettings,
    PydanticBaseSettingsSource,
    EnvSettingsSource,
    DotEnvSettingsSource,
    SecretsSettingsSource,
    InitSettingsSource,
)

class Settings(BaseSettings):
    database_url: str
    secret_key: str
    debug: bool = False

    @classmethod
    def settings_customise_sources(
        cls,
        settings_cls: Type[BaseSettings],
        init_settings: PydanticBaseSettingsSource,
        env_settings: PydanticBaseSettingsSource,
        dotenv_settings: PydanticBaseSettingsSource,
        file_secret_settings: PydanticBaseSettingsSource,
    ) -> Tuple[PydanticBaseSettingsSource, ...]:
        # Custom priority: init > env > dotenv > secrets
        return (init_settings, env_settings, dotenv_settings, file_secret_settings)
```

### Custom Settings Source

```python
from pydantic.fields import FieldInfo
from pydantic_settings import BaseSettings, PydanticBaseSettingsSource, EnvSettingsSource
from typing import Any, Tuple, Type, Dict
import json

class CommaSeparatedEnvSource(EnvSettingsSource):
    """Custom source that handles comma-separated list env vars."""

    def prepare_field_value(
        self,
        field_name: str,
        field: FieldInfo,
        value: Any,
        value_is_complex: bool,
    ) -> Any:
        # Parse comma-separated values for list fields
        if isinstance(value, str) and value_is_complex:
            try:
                return json.loads(value)  # Try JSON first
            except json.JSONDecodeError:
                return [item.strip() for item in value.split(",")]
        return super().prepare_field_value(field_name, field, value, value_is_complex)


class AppSettings(BaseSettings):
    allowed_hosts: list[str] = ["localhost"]
    debug: bool = False

    @classmethod
    def settings_customise_sources(
        cls,
        settings_cls,
        init_settings,
        env_settings,
        dotenv_settings,
        file_secret_settings,
    ):
        return (
            init_settings,
            CommaSeparatedEnvSource(settings_cls),
            dotenv_settings,
            file_secret_settings,
        )

# ALLOWED_HOSTS=localhost,example.com,api.example.com
```

---

## Configuration File Sources

### TOML Files

```bash
pip install pydantic-settings[toml]
```

```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic_settings import TomlConfigSettingsSource
from typing import Tuple, Type

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        toml_file="config.toml"
    )

    database_url: str
    debug: bool = False
    port: int = 8000

    @classmethod
    def settings_customise_sources(cls, settings_cls, init_settings, env_settings, dotenv_settings, file_secret_settings):
        return (init_settings, env_settings, TomlConfigSettingsSource(settings_cls))
```

`config.toml`:
```toml
database_url = "postgresql://localhost:5432/mydb"
debug = false
port = 8080
```

### YAML Files

```bash
pip install pydantic-settings[yaml]
```

```python
from pydantic_settings import BaseSettings, YamlConfigSettingsSource, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(yaml_file="config.yaml")

    database_url: str
    debug: bool = False

    @classmethod
    def settings_customise_sources(cls, settings_cls, init_settings, env_settings, dotenv_settings, file_secret_settings):
        return (init_settings, env_settings, YamlConfigSettingsSource(settings_cls))
```

`config.yaml`:
```yaml
database_url: postgresql://localhost:5432/mydb
debug: false
```

### JSON Files

```python
from pydantic_settings import BaseSettings, JsonConfigSettingsSource

class Settings(BaseSettings):
    model_config = {"json_file": "config.json"}

    @classmethod
    def settings_customise_sources(cls, settings_cls, **kwargs):
        return (kwargs["init_settings"], kwargs["env_settings"], JsonConfigSettingsSource(settings_cls))
```

### pyproject.toml

```python
from pydantic_settings import BaseSettings, PyprojectTomlConfigSettingsSource, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        pyproject_toml_table_header=("tool", "myapp")
    )
    database_url: str
```

`pyproject.toml`:
```toml
[tool.myapp]
database_url = "postgresql://localhost:5432/dev"
```

---

## CLI Support

### Basic CLI Parsing

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(cli_parse_args=True)

    host: str = "localhost"
    port: int = 8000
    debug: bool = False

settings = Settings()
```

```bash
python app.py --host 0.0.0.0 --port 9000 --debug True
```

### Subcommands

```python
from typing import Optional
from pydantic import BaseModel
from pydantic_settings import BaseSettings, CliSubCommand, CliPositionalArg, SettingsConfigDict, get_subcommand

class ServeCommand(BaseModel):
    host: str = "0.0.0.0"
    port: int = 8000
    workers: int = 4

class MigrateCommand(BaseModel):
    direction: CliPositionalArg[str]  # "up" or "down"
    steps: int = 1

class CLI(BaseSettings, cli_parse_args=True):
    serve: CliSubCommand[ServeCommand]
    migrate: CliSubCommand[MigrateCommand]

cli = CLI()
cmd = get_subcommand(cli)

if isinstance(cmd, ServeCommand):
    run_server(cmd.host, cmd.port, cmd.workers)
elif isinstance(cmd, MigrateCommand):
    run_migration(cmd.direction, cmd.steps)
```

```bash
python app.py serve --port 9000
python app.py migrate up --steps 3
```

### CliApp Pattern

```python
from pydantic_settings import BaseSettings, CliApp, SettingsConfigDict

class AppSettings(BaseSettings):
    model_config = SettingsConfigDict(cli_parse_args=True)

    host: str = "localhost"
    port: int = 8000
    verbose: bool = False

    def cli_cmd(self) -> None:
        print(f"Starting server on {self.host}:{self.port}")
        if self.verbose:
            print("Verbose mode enabled")

# Parses CLI args, creates Settings, calls cli_cmd()
CliApp.run(AppSettings)
```

---

## Testing & Overrides

### Override with kwargs

```python
# In tests, pass values directly to avoid env vars
settings = Settings(
    database_url="sqlite:///test.db",
    secret_key="test-secret",
    debug=True
)
```

### Mock Environment

```python
import pytest
from unittest.mock import patch

@pytest.fixture
def test_settings():
    with patch.dict("os.environ", {
        "DATABASE_URL": "sqlite:///test.db",
        "SECRET_KEY": "test-secret",
        "DEBUG": "true"
    }):
        yield Settings()

def test_something(test_settings):
    assert test_settings.debug is True
```

### Override .env File

```python
@pytest.fixture
def settings(tmp_path):
    env_file = tmp_path / ".env.test"
    env_file.write_text("DATABASE_URL=sqlite:///test.db\nDEBUG=true\n")
    return Settings(_env_file=str(env_file))
```

---

## Common Patterns

### Singleton Settings

```python
from functools import lru_cache
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
    )

    app_name: str = "MyApp"
    debug: bool = False
    database_url: str
    secret_key: str
    allowed_origins: list[str] = ["http://localhost:3000"]
    log_level: str = "INFO"

@lru_cache
def get_settings() -> Settings:
    return Settings()

# Usage across the app
settings = get_settings()
```

### Full Production Settings

```python
from pydantic import Field, AnyHttpUrl, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import BaseModel
from functools import lru_cache
from typing import Literal

class DatabaseSettings(BaseModel):
    host: str = "localhost"
    port: int = 5432
    name: str = "app"
    user: str = "postgres"
    password: SecretStr = SecretStr("")

    @property
    def url(self) -> str:
        return (
            f"postgresql://{self.user}:{self.password.get_secret_value()}"
            f"@{self.host}:{self.port}/{self.name}"
        )

class RedisSettings(BaseModel):
    host: str = "localhost"
    port: int = 6379
    db: int = 0
    password: SecretStr | None = None

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        env_nested_delimiter="__",
        secrets_dir="/run/secrets",
        extra="ignore",
    )

    # Application
    app_name: str = "MyApp"
    environment: Literal["development", "staging", "production"] = "development"
    debug: bool = False
    secret_key: SecretStr
    allowed_origins: list[str] = ["http://localhost:3000"]

    # Services
    database: DatabaseSettings = DatabaseSettings()
    redis: RedisSettings = RedisSettings()

    # External APIs
    stripe_api_key: SecretStr | None = None
    sendgrid_api_key: SecretStr | None = None

    # Feature flags
    enable_analytics: bool = False
    maintenance_mode: bool = False

    @property
    def is_production(self) -> bool:
        return self.environment == "production"

@lru_cache
def get_settings() -> Settings:
    return Settings()

# .env file
"""
APP_NAME=My Production App
ENVIRONMENT=production
DEBUG=false
SECRET_KEY=your-production-secret-key-here

DATABASE__HOST=prod-db.example.com
DATABASE__PORT=5432
DATABASE__NAME=production_db
DATABASE__USER=app_user
DATABASE__PASSWORD=prod-db-password

REDIS__HOST=redis.example.com
REDIS__PORT=6379

STRIPE_API_KEY=sk_live_...
ENABLE_ANALYTICS=true
"""
```

### Cloud Secret Managers

```python
# AWS Secrets Manager
from pydantic_settings import BaseSettings, AWSSecretsManagerSettingsSource
import os

class Settings(BaseSettings):
    database_password: str
    api_key: str

    @classmethod
    def settings_customise_sources(cls, settings_cls, init_settings, env_settings, dotenv_settings, file_secret_settings):
        return (
            init_settings,
            env_settings,
            AWSSecretsManagerSettingsSource(
                settings_cls,
                os.environ["AWS_SECRET_ID"]
            )
        )
```

```python
# Azure Key Vault
from azure.identity import DefaultAzureCredential
from pydantic_settings import BaseSettings, AzureKeyVaultSettingsSource

class Settings(BaseSettings):
    database_password: str
    api_key: str

    @classmethod
    def settings_customise_sources(cls, settings_cls, init_settings, env_settings, dotenv_settings, file_secret_settings):
        return (
            init_settings,
            env_settings,
            AzureKeyVaultSettingsSource(
                settings_cls,
                vault_url="https://my-vault.vault.azure.net/",
                credential=DefaultAzureCredential(),
                snake_case_conversion=True,
            )
        )
```

### Complex Type Parsing

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # Env var: ALLOWED_HOSTS='["localhost","example.com"]'  (JSON)
    allowed_hosts: list[str] = ["localhost"]

    # Env var: DATABASE_CONFIG='{"host":"localhost","port":5432}'  (JSON)
    database_config: dict[str, str | int] = {}

    # Nested with __
    # DATABASE__HOST=localhost DATABASE__PORT=5432
    # model_config = SettingsConfigDict(env_nested_delimiter="__")
```

### Disable JSON Parsing for Custom Parsing

```python
from typing import Annotated
from pydantic import field_validator
from pydantic_settings import BaseSettings, NoDecode

class Settings(BaseSettings):
    # Use NoDecode to prevent automatic JSON parsing
    raw_list: Annotated[list[str], NoDecode] = []

    @field_validator("raw_list", mode="before")
    @classmethod
    def parse_raw_list(cls, v: str) -> list[str]:
        if isinstance(v, str):
            return v.split(":")   # Colon-separated instead of JSON
        return v

# ALLOWED_PATHS=/home/user:/usr/local:/opt
```
