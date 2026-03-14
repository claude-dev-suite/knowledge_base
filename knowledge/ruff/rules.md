# Ruff - Rules Reference

> Official Documentation: https://docs.astral.sh/ruff/rules/

## Overview

Ruff implements 800+ rules across 50+ categories, each identified by a prefix code. Rules can be selected by full code (e.g., `F401`), prefix (e.g., `F4`), or category (e.g., `F`).

---

## Table of Contents

1. [Rule Code Structure](#rule-code-structure)
2. [E/W - pycodestyle](#ew---pycodestyle)
3. [F - Pyflakes](#f---pyflakes)
4. [I - isort](#i---isort)
5. [N - pep8-naming](#n---pep8-naming)
6. [UP - pyupgrade](#up---pyupgrade)
7. [B - flake8-bugbear](#b---flake8-bugbear)
8. [S - flake8-bandit (Security)](#s---flake8-bandit-security)
9. [ANN - flake8-annotations](#ann---flake8-annotations)
10. [Other Notable Categories](#other-notable-categories)
11. [Rule Suppression](#rule-suppression)

---

## Rule Code Structure

Format: `<PREFIX><3-digit-number>`

- `F401` — `F` = Pyflakes, `401` = specific rule
- `E501` — `E` = pycodestyle error, `501` = line too long

Fixable rules are marked with a wrench icon in the docs. Auto-fix is applied with `ruff check --fix`.

---

## E/W - pycodestyle

Style rules ported from `pycodestyle` (formerly `pep8`).

**Category prefix**: `E` (errors), `W` (warnings)

### Most Common E Rules

| Rule | Description | Fixable |
|------|-------------|---------|
| `E101` | Indentation contains mixed spaces and tabs | Yes |
| `E111` | Indentation is not a multiple of `indent-width` | Yes |
| `E114` | Indentation is not a multiple of `indent-width` (comment) | Yes |
| `E117` | Over-indented | Yes |
| `E201` | Whitespace after `(`, `[`, or `{` | Yes |
| `E202` | Whitespace before `)`, `]`, or `}` | Yes |
| `E203` | Whitespace before `:`, `;`, or `,` | Yes |
| `E211` | Whitespace before `(` or `[` | Yes |
| `E225` | Missing whitespace around operator | Yes |
| `E231` | Missing whitespace after `,`, `;`, or `:` | Yes |
| `E251` | Unexpected spaces around keyword / parameter equals | Yes |
| `E261` | At least two spaces before inline comment | Yes |
| `E262` | Inline comment should start with `# ` | Yes |
| `E265` | Block comment should start with `# ` | Yes |
| `E271` | Multiple spaces after keyword | Yes |
| `E302` | Expected 2 blank lines, got {found} | Yes |
| `E303` | Too many blank lines ({found}) | Yes |
| `E304` | `def` that follows decorator should have no blank lines | Yes |
| `E305` | Expected 2 blank lines after class or function definition | Yes |
| `E401` | Multiple imports on one line | No |
| `E402` | Module-level import not at top of file | No |
| `E501` | Line too long ({width} > {limit} characters) | No |
| `E711` | Comparison to `None` using `==` (use `is` or `is not`) | Yes |
| `E712` | Comparison to `True`/`False` using `==` (use `is`) | Yes |
| `E721` | Type comparison using `==` (use `isinstance()`) | No |
| `E731` | Do not assign a `lambda` expression, use `def` | Yes |
| `E741` | Ambiguous variable name (l, O, I) | No |
| `E902` | `SyntaxError` in file | No |

### Most Common W Rules

| Rule | Description | Fixable |
|------|-------------|---------|
| `W191` | Indentation contains tabs | Yes |
| `W291` | Trailing whitespace | Yes |
| `W292` | No newline at end of file | Yes |
| `W293` | Whitespace before comment | Yes |
| `W505` | Doc line too long | No |
| `W605` | Invalid escape sequence `\x` | Yes |

### Examples

```python
# E711 — comparison to None
if x == None:  # bad
    pass
if x is None:  # good
    pass

# E712 — comparison to boolean
if flag == True:  # bad
    pass
if flag:  # good
    pass

# E741 — ambiguous name
l = []  # bad (looks like 1)
items = []  # good

# E731 — lambda assignment
double = lambda x: x * 2  # bad
def double(x):  # good
    return x * 2
```

---

## F - Pyflakes

Logical errors: undefined names, unused imports, unreachable code, etc.

**Category prefix**: `F`

### Key Rules

| Rule | Description | Fixable |
|------|-------------|---------|
| `F401` | `{module}` imported but unused | Yes (with caution) |
| `F402` | Import `{module}` from line {line} shadowed by loop variable | No |
| `F403` | `from {module} import *` used; unable to detect undefined names | No |
| `F404` | Late future import | No |
| `F405` | `{name}` may be undefined or from star import | No |
| `F406` | `module` imported for side-effect only; unable to detect undefined names | No |
| `F407` | Future feature is not defined | No |
| `F501`-`F509` | `%`-format string errors (wrong arg count, invalid format) | No |
| `F521`-`F524` | `.format()` call errors | No |
| `F601`-`F632` | Various logical issues | Partial |
| `F811` | Redefinition of unused `{name}` from import | Yes |
| `F821` | Undefined name `{name}` | No |
| `F841` | Local variable `{name}` is assigned but never used | Yes |
| `F842` | Local variable `{name}` is annotated but never used | No |
| `F901` | `raise NotImplemented` should be `raise NotImplementedError` | Yes |

### Examples

```python
# F401 — unused import
import os  # bad if os is never used

# F841 — assigned but never used
def process():
    result = compute()  # F841 if result is never used
    return "done"

# F821 — undefined name
print(undefined_variable)  # F821

# F811 — redefinition
import os
import os  # F811 — duplicate import

# F901 — wrong exception
def not_implemented():
    raise NotImplemented  # bad
    raise NotImplementedError  # good
```

---

## I - isort

Import sorting rules.

**Category prefix**: `I`

| Rule | Description | Fixable |
|------|-------------|---------|
| `I001` | Import block is unsorted or unformatted | Yes |
| `I002` | Missing required import | Yes |

### What I001 enforces

- Standard library imports before third-party
- Third-party imports before local imports
- Alphabetical order within sections
- Blank lines between sections

```python
# Bad (I001)
import mymodule
import os
import requests

# Good (after fix)
import os

import requests

import mymodule
```

### Configuration

```toml
[tool.ruff.lint.isort]
known-first-party = ["mypackage"]
force-single-line = false
lines-between-sections = 1
```

---

## N - pep8-naming

Naming convention rules from `pep8-naming`.

**Category prefix**: `N`

| Rule | Description | Fixable |
|------|-------------|---------|
| `N801` | Class name `{name}` should use CapWords convention | No |
| `N802` | Function name `{name}` should be lowercase | No |
| `N803` | Argument name `{name}` should be lowercase | No |
| `N804` | First argument of a class method should be named `cls` | No |
| `N805` | First argument of a method should be named `self` | No |
| `N806` | Variable `{name}` in function should be lowercase | No |
| `N811` | Constant `{name}` imported as non-constant | No |
| `N812` | Lowercase `{name}` imported as non-lowercase | No |
| `N813` | Camelcase `{name}` imported as lowercase | No |
| `N814` | Camelcase `{name}` imported as constant | No |
| `N815` | MixedCase `{name}` in global scope | No |
| `N816` | MixedCase `{name}` in global scope (variable) | No |
| `N817` | CamelCase `{name}` imported as acronym | No |
| `N818` | Exception name `{name}` should be named with an `Error` suffix | No |
| `N999` | Invalid module name `{name}` | No |

### Examples

```python
# N801 — class name not CapWords
class my_class:  # bad
    pass
class MyClass:  # good
    pass

# N802 — function not lowercase
def MyFunction():  # bad
    pass
def my_function():  # good
    pass

# N818 — exception without Error suffix
class NotFoundException(Exception):  # bad
    pass
class NotFoundError(Exception):  # good
    pass

# N805 — method first arg not self
class MyClass:
    def method(this):  # bad
        pass
    def method(self):  # good
        pass
```

---

## UP - pyupgrade

Modernize Python syntax for newer versions. 300+ rules.

**Category prefix**: `UP`

| Rule | Description | Fixable |
|------|-------------|---------|
| `UP001` | `%s` formatting → f-string | Yes |
| `UP003` | `type()` → `isinstance()` for type checks | Yes |
| `UP004` | Class inherits from `object` unnecessarily (Python 3) | Yes |
| `UP006` | Use `list` instead of `typing.List` (Python 3.9+) | Yes |
| `UP007` | Use `X \| Y` instead of `Union[X, Y]` (Python 3.10+) | Yes |
| `UP008` | Use `super()` instead of `super(__class__, self)` | Yes |
| `UP009` | UTF-8 encoding declaration is unnecessary | Yes |
| `UP010` | Unnecessary `__future__` import | Yes |
| `UP011` | `NameError: name 'unicode' is not defined` | Yes |
| `UP012` | Unnecessary call to `encode` with `"utf-8"` as the default | Yes |
| `UP015` | Unnecessary open mode parameter | Yes |
| `UP017` | Use `datetime.UTC` alias | Yes |
| `UP018` | Unnecessary `{literal_type}` call (e.g., `str("abc")`) | Yes |
| `UP019` | `typing.Text` is deprecated, use `str` | Yes |
| `UP020` | Use `builtin` `open` | Yes |
| `UP021` | `universal_newlines` is deprecated, use `text` | Yes |
| `UP024` | Replace aliased errors with `OSError` | Yes |
| `UP025` | Remove unicode literals from strings | Yes |
| `UP026` | `mock` is deprecated, use `unittest.mock` | Yes |
| `UP028` | Replace `yield` in generator with `yield from` | Yes |
| `UP029` | Unnecessary builtin import | Yes |
| `UP030` | Use implicit references for positional format fields | Yes |
| `UP031` | Use format specifiers instead of `%` format | Yes |
| `UP032` | Use f-string instead of `format` call | Yes |
| `UP034` | Extraneous parentheses | Yes |
| `UP035` | `typing` attribute removed (use `collections.abc` instead) | Yes |
| `UP036` | Version block is outdated for `target-version` | Yes |
| `UP038` | Use `X \| Y` in `isinstance` call instead of `(X, Y)` | Yes |
| `UP040` | Type alias uses `TypeAlias` annotation (use `type` statement) | Yes |

### Examples

```python
# UP006/UP007 — typing modernization
from typing import List, Optional, Union

def process(items: List[str]) -> Optional[int]:  # bad on Python 3.9+
    pass

def process(items: list[str]) -> int | None:  # good
    pass

# UP004 — remove object inheritance
class MyClass(object):  # bad
    pass
class MyClass:  # good
    pass

# UP032 — f-string
name = "world"
msg = "Hello, {}".format(name)  # bad
msg = f"Hello, {name}"          # good

# UP008 — modern super()
class Child(Parent):
    def __init__(self):
        super(Child, self).__init__()  # bad
        super().__init__()             # good
```

---

## B - flake8-bugbear

Detects likely bugs and design problems.

**Category prefix**: `B`

| Rule | Description | Fixable |
|------|-------------|---------|
| `B002` | Python does not support the unary prefix increment (`++n`) | No |
| `B003` | Avoid assigning a lambda expression; use a `def` | No |
| `B004` | Using `hasattr(x, "__call__")` to test if callable is unreliable | No |
| `B005` | Do not use `.strip()` with multi-character strings | No |
| `B006` | Do not use mutable data structures for argument defaults | No |
| `B007` | Loop control variable `{name}` not used in loop body | Yes |
| `B008` | Do not perform function call in default argument | No |
| `B009` | Do not call `getattr` with a constant attribute value | Yes |
| `B010` | Do not call `setattr` with a constant attribute value | Yes |
| `B011` | Do not assert `False` (`raise AssertionError()` instead) | Yes |
| `B012` | No `return` or `continue` in `finally` block | No |
| `B013` | Redundant tuple in exception handler | Yes |
| `B014` | Redundant exception types in exception handler | Yes |
| `B015` | Pointless comparison (`a == b` without assignment or return) | No |
| `B016` | Cannot raise a literal | No |
| `B017` | `pytest.raises({exception})` should be followed by `.match` | No |
| `B018` | Found useless expression | No |
| `B019` | Use of `functools.lru_cache` or `functools.cache` on methods can lead to memory leaks | No |
| `B020` | Loop control variable `{name}` overrides iterator | No |
| `B021` | `f-string` used as docstring | No |
| `B022` | No arguments passed to `contextlib.suppress` | No |
| `B023` | Function definition does not bind loop variable `{name}` | No |
| `B024` | `{name}` is an abstract base class, but none of its methods are abstract | No |
| `B025` | `try-except-else` doesn't allow catching exceptions in the else block | No |
| `B026` | Star-arg unpacking after a keyword argument | No |
| `B027` | `{name}` is an empty method in an abstract base class, consider `@abstractmethod` | No |
| `B028` | No explicit `stacklevel` keyword argument found | No |
| `B029` | Using `warnings.warn` without `stacklevel` | No |
| `B034` | Re-reading `sys.stdin` or `sys.stdout` after seeking | No |
| `B035` | Dict comprehension uses variable from outer scope as key | No |
| `B904` | Within an `except` clause, raise exceptions with `raise ... from err` | No |
| `B905` | `zip()` without an explicit `strict=` parameter | No |

### Examples

```python
# B006 — mutable default argument
def add_item(item, lst=[]):  # bad — shared across calls!
    lst.append(item)
    return lst

def add_item(item, lst=None):  # good
    if lst is None:
        lst = []
    lst.append(item)
    return lst

# B007 — unused loop variable
for _ in range(10):  # bad (use _ if intentionally unused, but ruff checks non-_ names)
    print("hello")

# B020 — loop variable overrides iterator
items = [1, 2, 3]
for items in items:  # bad — overwrites `items`!
    print(items)

# B904 — chained exception
try:
    risky_operation()
except ValueError:
    raise RuntimeError("failed")  # bad
    raise RuntimeError("failed") from ValueError  # good

# B905 — zip without strict
for a, b in zip(list1, list2):  # bad
    pass
for a, b in zip(list1, list2, strict=True):  # good
    pass
```

---

## S - flake8-bandit (Security)

Security vulnerability detection ported from `bandit`.

**Category prefix**: `S`

| Rule | Description | Fixable |
|------|-------------|---------|
| `S101` | Use of `assert` detected | No |
| `S102` | Use of `exec` detected | No |
| `S103` | `os.chmod` setting a permissive mask on file or directory | No |
| `S104` | Possible binding to all interfaces | No |
| `S105` | Possible hardcoded password assigned to: `{name}` | No |
| `S106` | Possible hardcoded password as function argument | No |
| `S107` | Possible hardcoded password as default argument | No |
| `S108` | Probable insecure usage of temporary file or directory | No |
| `S110` | `try`-`except`-`pass` detected | No |
| `S112` | `try`-`except`-`continue` detected | No |
| `S113` | Probable use of `requests` call without timeout | No |
| `S201` | Flask app with debug=True | No |
| `S202` | Use of `tarfile.extractall()` | No |
| `S301` | `pickle` and related modules are insecure | No |
| `S303` | Use of MD2, MD4, MD5 detected (weak hash) | No |
| `S304` | Use of weak encryption cipher detected | No |
| `S305` | Use of deprecated DES or RC2 cipher | No |
| `S306` | Use of insecure cipher RC4 | No |
| `S307` | Use of possibly insecure function (`eval`) | No |
| `S308` | Use of `mark_safe()` may expose cross-site scripting | No |
| `S310` | Audit URL open for permitted schemes | No |
| `S311` | Standard pseudo-random generators are not suitable for security/cryptographic purposes | No |
| `S312` | `telnetlib` use is deprecated | No |
| `S313`-`S320` | XML parsing vulnerable to attacks | No |
| `S321` | FTP-related functions | No |
| `S323` | `ssl.create_default_context` used without `check_hostname` | No |
| `S324` | `hashlib` functions with `usedforsecurity=False` | No |
| `S401`-`S419` | Various insecure imports | No |
| `S501` | Requests with `verify=False` | No |
| `S502` | SSL wrap_socket without `ssl_version=TLSv1` | No |
| `S503` | SSL/TLS functions with `certfile` but no `keyfile` | No |
| `S504` | SSL with no version | No |
| `S505` | Weak cryptographic key | No |
| `S506` | Unsafe use of `yaml.load()` | Yes |
| `S507` | SSH no host key verification | No |
| `S508` | SNMP with insecure version | No |
| `S509` | SNMP with weak cipher | No |
| `S601` | Possible shell injection via `Popen` with `shell=True` | No |
| `S602` | `subprocess` call with `shell=True` | No |
| `S603` | `subprocess` call - check for execution of untrusted input | No |
| `S604` | Function call with `shell=True` | No |
| `S605` | Starting a process with a shell: audit for injection | No |
| `S606` | Starting a process without a shell | No |
| `S607` | Starting a process with a partial executable path | No |
| `S608` | Possible SQL injection via string-based query construction | No |
| `S609` | Possible wildcard injection in `subprocess` call | No |
| `S610`-`S612` | Use of Django `extra` / `RawSQL` / `Unparseable` | No |
| `S701` | Jinja2 templates with `autoescape=False` | No |
| `S702` | Mako templates allow HTML/Script | No |

### Examples

```python
# S101 — assert (disabled at runtime with -O flag)
assert user.is_admin  # bad in production code
if not user.is_admin:  # good
    raise PermissionError("Not admin")

# S105 — hardcoded password
password = "mysecretpassword"  # bad
password = os.environ["DB_PASSWORD"]  # good

# S311 — insecure random
import random
token = random.randint(1, 1000000)  # bad
import secrets
token = secrets.randbelow(1000000)  # good

# S506 — unsafe yaml.load
import yaml
data = yaml.load(content)  # bad
data = yaml.safe_load(content)  # good (fixable)

# S608 — SQL injection
query = "SELECT * FROM users WHERE name = '%s'" % name  # bad
query = "SELECT * FROM users WHERE name = %s"  # good (use parameterized)
cursor.execute(query, (name,))

# S602 — shell injection
subprocess.run(f"rm {user_input}", shell=True)  # bad
subprocess.run(["rm", user_input])  # good
```

---

## ANN - flake8-annotations

Enforce type annotation requirements.

**Category prefix**: `ANN`

| Rule | Description | Fixable |
|------|-------------|---------|
| `ANN001` | Missing type annotation for function argument `{name}` | No |
| `ANN002` | Missing type annotation for `*args` | No |
| `ANN003` | Missing type annotation for `**kwargs` | No |
| `ANN101` | Missing type annotation for `self` in method | No |
| `ANN102` | Missing type annotation for `cls` in classmethod | No |
| `ANN201` | Missing return type annotation for public function `{name}` | No |
| `ANN202` | Missing return type annotation for private function `{name}` | No |
| `ANN204` | Missing return type annotation for special method `{name}` | No |
| `ANN205` | Missing return type annotation for static method `{name}` | No |
| `ANN206` | Missing return type annotation for class method `{name}` | No |
| `ANN401` | Dynamically typed expressions (`typing.Any`) are disallowed | No |

### Examples

```python
# ANN001 / ANN201
def process(data):  # bad
    return data

def process(data: list[str]) -> list[str]:  # good
    return data

# ANN002 / ANN003
def func(*args, **kwargs):  # bad
    pass

def func(*args: str, **kwargs: int) -> None:  # good
    pass

# ANN401 — avoid Any
from typing import Any
def handle(data: Any) -> Any:  # bad
    return data
```

---

## Other Notable Categories

| Prefix | Source | Key Focus |
|--------|--------|-----------|
| `A` | flake8-builtins | Variable names that shadow builtins (`id`, `list`, `type`, etc.) |
| `ARG` | flake8-unused-arguments | Unused function arguments |
| `C4` | flake8-comprehensions | Simplify list/dict/set comprehensions |
| `C90` | mccabe | Cyclomatic complexity |
| `COM` | flake8-commas | Trailing comma enforcement |
| `D` | pydocstyle | Docstring style and presence |
| `DTZ` | flake8-datetimez | Naive datetime usage (missing timezone) |
| `EM` | flake8-errmsg | Error message formatting |
| `ERA` | eradicate | Commented-out code |
| `FBT` | flake8-boolean-trap | Boolean argument traps |
| `G` | flake8-logging-format | Logging format strings |
| `ICN` | flake8-import-conventions | Standard import aliases (`import numpy as np`) |
| `ISC` | flake8-implicit-str-concat | Implicit string concatenation |
| `PD` | pandas-vet | Pandas anti-patterns |
| `PERF` | perflint | Performance anti-patterns |
| `PIE` | flake8-pie | Misc cleanups |
| `PL` | Pylint | Pylint rules subset |
| `PT` | flake8-pytest-style | pytest best practices |
| `PTH` | flake8-use-pathlib | Use `pathlib` instead of `os.path` |
| `Q` | flake8-quotes | Quote style consistency |
| `RET` | flake8-return | Return statement issues |
| `RSE` | flake8-raise | Raise statement issues |
| `RUF` | Ruff-specific | Ruff's own rules |
| `SIM` | flake8-simplify | Code simplification opportunities |
| `T10` | flake8-debugger | `breakpoint()` and `pdb` usage |
| `T20` | flake8-print | `print` statement detection |
| `TCH` | flake8-type-checking | Move imports to `TYPE_CHECKING` block |
| `TID` | flake8-tidy-imports | Import tidiness |
| `TRY` | tryceratops | `try`-`except` best practices |
| `YTT` | flake8-2020 | Misuse of `sys.version` |

---

## Rule Suppression

### Per-line (noqa)

```python
import os  # noqa: F401
import os  # noqa: F401, E302

# Suppress ALL violations on this line
x = 1  # noqa
```

### Per-block

```python
# ruff: disable[E501]
VERY_LONG_STRING = "This is an extremely long string that would normally violate E501..."
# ruff: enable[E501]
```

### Per-file

```python
# At the top of the file:
# ruff: noqa
# ruff: noqa: F401,E302
# flake8: noqa  (also respected)
```

### Auto-add noqa comments

```bash
# Add noqa directives to all current violations
ruff check --add-noqa

# Find unnecessary noqa comments
ruff check --extend-select RUF100 --fix
```

### Via configuration (for bulk suppression)

```toml
[tool.ruff.lint.per-file-ignores]
"legacy_module.py" = ["ALL"]
"generated_file.py" = ["E", "W", "F"]
```
