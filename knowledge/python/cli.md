# Python CLI Development

Complete guide to building command-line applications with Typer, Click, and Rich.

## Tool Comparison

| Tool | Type Hints | Auto Completion | Best For |
|------|------------|-----------------|----------|
| **Typer** | Native | Automatic | Modern CLIs |
| Click | Decorators | Manual | Complex CLIs |
| argparse | Manual | Manual | No dependencies |

## Typer

Typer is a library for building CLI applications based on Python type hints.

### Installation

```bash
uv add typer
# With all features
uv add "typer[all]"
```

### Basic Application

```python
import typer

app = typer.Typer()

@app.command()
def hello(name: str, count: int = 1) -> None:
    """Greet NAME, COUNT times."""
    for _ in range(count):
        print(f"Hello {name}!")

if __name__ == "__main__":
    app()
```

```bash
python main.py hello World --count 3
python main.py hello --help
```

### Arguments and Options

```python
from typing import Annotated
import typer

app = typer.Typer()

@app.command()
def process(
    # Required argument
    path: str,

    # Optional argument with default
    output: str = "output.txt",

    # Option with short name
    verbose: Annotated[bool, typer.Option("--verbose", "-v")] = False,

    # Option with prompt
    name: Annotated[str, typer.Option(prompt=True)],

    # Option with environment variable
    token: Annotated[str, typer.Option(envvar="API_TOKEN")] = "",

    # Option with choices
    format: Annotated[str, typer.Option()] = "json",

    # Flag (--flag / --no-flag)
    dry_run: Annotated[bool, typer.Option("--dry-run/--no-dry-run")] = False,
) -> None:
    """Process a file at PATH."""
    if verbose:
        print(f"Processing {path}")
    print(f"Output: {output}")
```

### Multiple Commands

```python
import typer

app = typer.Typer(help="My awesome CLI")

@app.command()
def create(name: str) -> None:
    """Create a new item."""
    print(f"Creating {name}")

@app.command()
def delete(
    name: str,
    force: Annotated[bool, typer.Option("--force", "-f")] = False,
) -> None:
    """Delete an item."""
    if force or typer.confirm(f"Delete {name}?"):
        print(f"Deleted {name}")

@app.command("list")  # Rename command
def list_items() -> None:
    """List all items."""
    print("Items: a, b, c")

if __name__ == "__main__":
    app()
```

### Subcommands (Groups)

```python
import typer

app = typer.Typer()
users_app = typer.Typer(help="User management")
app.add_typer(users_app, name="users")

@users_app.command("list")
def list_users() -> None:
    """List all users."""
    print("Users: alice, bob")

@users_app.command("create")
def create_user(name: str, email: str) -> None:
    """Create a new user."""
    print(f"Created {name} ({email})")

@users_app.command("delete")
def delete_user(user_id: int) -> None:
    """Delete a user by ID."""
    print(f"Deleted user {user_id}")
```

```bash
python main.py users list
python main.py users create alice alice@example.com
```

### Callbacks and Context

```python
import typer

app = typer.Typer()

@app.callback()
def main(
    ctx: typer.Context,
    verbose: Annotated[bool, typer.Option("--verbose", "-v")] = False,
) -> None:
    """My CLI application."""
    ctx.ensure_object(dict)
    ctx.obj["verbose"] = verbose

@app.command()
def process(ctx: typer.Context, path: str) -> None:
    """Process a file."""
    if ctx.obj["verbose"]:
        print(f"Verbose mode: processing {path}")
    print(f"Done: {path}")
```

### Exit Codes and Errors

```python
import typer
from typing import NoReturn

def error_exit(message: str, code: int = 1) -> NoReturn:
    """Print error and exit."""
    typer.echo(typer.style(f"Error: {message}", fg=typer.colors.RED), err=True)
    raise typer.Exit(code=code)

@app.command()
def validate(path: str) -> None:
    """Validate a file."""
    from pathlib import Path

    if not Path(path).exists():
        error_exit(f"File not found: {path}")

    if not Path(path).suffix == ".json":
        error_exit(f"Expected JSON file: {path}", code=2)

    typer.echo(typer.style("Valid!", fg=typer.colors.GREEN))
```

## Rich Integration

Rich provides beautiful terminal output.

### Installation

```bash
uv add rich
```

### Basic Output

```python
from rich import print as rprint
from rich.console import Console

console = Console()

# Styled text
rprint("[bold green]Success![/bold green]")
rprint("[red]Error:[/red] Something went wrong")
rprint("[blue underline]https://example.com[/blue underline]")

# Console methods
console.print("Info", style="blue")
console.print("Warning", style="yellow bold")
console.log("Debug message")  # With timestamp
```

### Tables

```python
from rich.console import Console
from rich.table import Table

console = Console()

table = Table(title="Users")
table.add_column("ID", style="cyan", justify="right")
table.add_column("Name", style="green")
table.add_column("Email")
table.add_column("Active", justify="center")

table.add_row("1", "Alice", "alice@example.com", "✓")
table.add_row("2", "Bob", "bob@example.com", "✓")
table.add_row("3", "Charlie", "charlie@example.com", "✗")

console.print(table)
```

### Progress Bars

```python
from rich.progress import track, Progress
import time

# Simple progress
for item in track(range(100), description="Processing..."):
    time.sleep(0.01)

# Advanced progress
with Progress() as progress:
    task1 = progress.add_task("[red]Downloading...", total=100)
    task2 = progress.add_task("[green]Processing...", total=100)

    while not progress.finished:
        progress.update(task1, advance=0.5)
        progress.update(task2, advance=0.3)
        time.sleep(0.02)
```

### Panels and Trees

```python
from rich.console import Console
from rich.panel import Panel
from rich.tree import Tree

console = Console()

# Panel
console.print(Panel("Hello, World!", title="Greeting", border_style="green"))

# Tree
tree = Tree("Project")
src = tree.add("src")
src.add("main.py")
src.add("utils.py")
tree.add("tests").add("test_main.py")
tree.add("pyproject.toml")

console.print(tree)
```

### Syntax Highlighting

```python
from rich.console import Console
from rich.syntax import Syntax

console = Console()

code = '''
def hello(name: str) -> str:
    return f"Hello, {name}!"

print(hello("World"))
'''

syntax = Syntax(code, "python", theme="monokai", line_numbers=True)
console.print(syntax)
```

### Status Spinner

```python
from rich.console import Console
import time

console = Console()

with console.status("[bold green]Working on something...") as status:
    time.sleep(2)
    status.update("[bold blue]Almost done...")
    time.sleep(1)

console.print("[bold green]Done!")
```

## Typer + Rich

```python
import typer
from rich.console import Console
from rich.table import Table
from rich.progress import track

app = typer.Typer()
console = Console()

@app.command()
def users() -> None:
    """List all users."""
    table = Table(title="Users")
    table.add_column("ID", style="cyan")
    table.add_column("Name", style="green")

    for i in range(3):
        table.add_row(str(i), f"User {i}")

    console.print(table)

@app.command()
def process(count: int = 100) -> None:
    """Process items with progress bar."""
    for _ in track(range(count), description="Processing..."):
        pass  # Do work

    console.print("[bold green]Done!")
```

## Click (Alternative)

Click is a mature library for creating CLIs.

### Basic Usage

```python
import click

@click.group()
def cli() -> None:
    """My CLI application."""
    pass

@cli.command()
@click.argument("name")
@click.option("--count", "-c", default=1, help="Number of greetings")
def hello(name: str, count: int) -> None:
    """Greet NAME."""
    for _ in range(count):
        click.echo(f"Hello {name}!")

@cli.command()
@click.argument("path", type=click.Path(exists=True))
@click.option("--output", "-o", default="out.txt")
def process(path: str, output: str) -> None:
    """Process file at PATH."""
    click.echo(f"Processing {path} -> {output}")

if __name__ == "__main__":
    cli()
```

### Click with Rich (rich-click)

```python
import rich_click as click

click.rich_click.USE_RICH_MARKUP = True
click.rich_click.USE_MARKDOWN = True

@click.command()
@click.option("--name", "-n", help="Your [bold]name[/bold]")
def hello(name: str) -> None:
    """Say **hello** to someone."""
    click.echo(f"Hello {name}!")
```

## Testing CLI Apps

### With Typer

```python
from typer.testing import CliRunner
from myapp.cli import app

runner = CliRunner()

def test_hello() -> None:
    result = runner.invoke(app, ["hello", "World"])
    assert result.exit_code == 0
    assert "Hello World" in result.stdout

def test_hello_with_count() -> None:
    result = runner.invoke(app, ["hello", "World", "--count", "3"])
    assert result.exit_code == 0
    assert result.stdout.count("Hello World") == 3

def test_process_missing_file() -> None:
    result = runner.invoke(app, ["process", "nonexistent.txt"])
    assert result.exit_code == 1
    assert "not found" in result.stdout.lower()
```

### With Click

```python
from click.testing import CliRunner
from myapp.cli import cli

runner = CliRunner()

def test_hello() -> None:
    result = runner.invoke(cli, ["hello", "World"])
    assert result.exit_code == 0
    assert "Hello World" in result.output
```

## Packaging CLI

### pyproject.toml

```toml
[project.scripts]
my-cli = "my_package.cli:app"

# Multiple entry points
[project.scripts]
my-cli = "my_package.cli:app"
my-tool = "my_package.tools:main"
```

### Usage After Install

```bash
# Install package
uv pip install .

# Use CLI
my-cli hello World
my-cli --help
```

## Best Practices

1. **Use Typer** for new projects (type hints, auto-completion)
2. **Use Rich** for beautiful output
3. **Add `--help`** to all commands (automatic with Typer)
4. **Use exit codes** properly (0=success, non-zero=error)
5. **Test with CliRunner** (Typer or Click)
6. **Use `Annotated`** for complex options
7. **Group related commands** with subcommands

## References

- [Typer documentation](https://typer.tiangolo.com/)
- [Rich documentation](https://rich.readthedocs.io/)
- [Click documentation](https://click.palletsprojects.com/)
- [rich-click](https://github.com/ewels/rich-click)
