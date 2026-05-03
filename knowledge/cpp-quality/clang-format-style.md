# C++ Quality - clang-format Style Options

> **Source**: https://clang.llvm.org/docs/ClangFormatStyleOptions.html
> **Skill**: dev-suite skill `quality/cpp-quality` — see SKILL.md for the always-loaded quick reference.

## What this covers

How to choose a base style (`BasedOnStyle`), which options matter most for diff-stability, the
trade-offs of each well-known preset (LLVM, Google, Mozilla, Chromium, Microsoft, GNU), and
options that are easy to get wrong (`AlignAfterOpenBracket`, `BinPackArguments`, `BreakBeforeBraces`).

## Deep dive

### Picking a base

| Style | Indent | Column | Brace | Notable |
|-------|--------|--------|-------|---------|
| `LLVM` | 2 | 80 | Attach | Conservative, the default |
| `Google` | 2 | 80 | Attach | `AllowShortFunctionsOnASingleLine: All`, no `else` after `return` reformatting |
| `Chromium` | 2 | 80 | Attach | Like Google but more conservative on auto |
| `Mozilla` | 2 | 80 | Mozilla (custom) | Newline before `class` braces |
| `Microsoft` | 4 | 120 | Allman | Closer to Visual Studio defaults |
| `GNU` | 2 | 79 | GNU (custom) | Awful bracketing IMO; avoid for new projects |
| `WebKit` | 4 | 0 | WebKit | Used by JSC and friends |

The recommendation: pick the closest base, override only what bothers your team, and **commit a
copy** rather than relying on `BasedOnStyle: Google` continuing to mean the same thing across LLVM
versions. The shipped styles drift between releases.

### Options with high diff impact

These cause big visible reformatting; pick once and don't re-litigate:

```yaml
ColumnLimit: 100               # 80 / 100 / 120 — pick one, hold the line
IndentWidth: 4                  # most modern C++ codebases use 4
ContinuationIndentWidth: 4
TabWidth: 4
UseTab: Never
PointerAlignment: Left          # int* p; not int *p;
ReferenceAlignment: Pointer     # follow PointerAlignment
NamespaceIndentation: None      # don't indent inside namespace { }
BreakBeforeBraces: Attach       # K&R; or Allman if you must
AccessModifierOffset: -4        # public:/private: outdented to align with class
Standard: c++20                 # pin so trailing commas, designated init format correctly
```

### `BinPack*` and bracket alignment — the long-line behavior

```yaml
AlignAfterOpenBracket: Align   # arguments line up under the (
BinPackArguments: false        # if doesn't fit, ONE arg per line (not packed)
BinPackParameters: false       # same for declarations
AllowAllArgumentsOnNextLine: true
AllowAllParametersOfDeclarationOnNextLine: true
```

With `BinPackArguments: false`, a long call wraps:

```cpp
auto result = process_request(
    request,
    options,
    timeout,
    callback);
```

With `true` (default in LLVM), it packs:

```cpp
auto result = process_request(request, options,
                              timeout, callback);
```

The unpacked form produces cleaner diffs when an argument is added or removed.

### Includes — sort and group

```yaml
SortIncludes: CaseInsensitive
IncludeBlocks: Regroup
IncludeCategories:
  - Regex:    '^"[^/]+\.h"$'         # local "x.h" — same dir
    Priority: 1
  - Regex:    '^".+/.+\.h"$'         # local "lib/x.h"
    Priority: 2
  - Regex:    '^<[a-z_]+>$'          # C++ stdlib (single token)
    Priority: 4
  - Regex:    '^<.+>$'                # third-party (anything else in <>)
    Priority: 3
```

`IncludeBlocks: Regroup` enforces blank-line separation between groups. Be careful: incorrect
ordering can change behavior (e.g. needing windows.h before winsock2.h). Mark such cases with:

```cpp
#include <winsock2.h>   // clang-format off
#include <windows.h>    // clang-format on
```

### Function and lambda formatting

```yaml
AllowShortFunctionsOnASingleLine: Empty   # only `void f() {}` stays on one line
AllowShortLambdasOnASingleLine: Inline
AllowShortIfStatementsOnASingleLine: Never
AllowShortCaseLabelsOnASingleLine: false
AlwaysBreakTemplateDeclarations: Yes      # template<...>\nT foo(...)
SpaceAfterTemplateKeyword: false
```

### Operators

```yaml
BreakBeforeBinaryOperators: NonAssignment   # newline before && in long expressions
AlignOperands: AlignAfterOperator
SpacesInParentheses: false
SpaceBeforeParens: ControlStatements
SpaceBeforeCpp11BracedList: false           # vector<int>{1, 2}, not vector<int> {1, 2}
SpacesInContainerLiterals: false
```

### Validating your config

```bash
# Dump the effective style with all defaults filled in
clang-format -dump-config -style=file

# See what a hypothetical change would do
clang-format -style=file src/foo.cpp | diff -u src/foo.cpp -
```

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Relying on `BasedOnStyle: Google` long-term | Google preset shifts between LLVM versions | Run `-dump-config` once, commit the materialized file |
| `ColumnLimit: 0` | Disables wrapping entirely → unreadable lines | Pick a real limit even if generous (120) |
| `BreakBeforeBraces: Allman` mid-project | Reformats every function | Settle on Attach in greenfield, accept it everywhere |
| `IncludeCategories` reordering windows.h | Breaks the build (winsock conflict) | Wrap with `// clang-format off` |
| `Standard: Auto` | Inconsistent diffs between machines with different LLVM | Pin `Standard: c++20` (or your standard) |
| `AlwaysBreakTemplateDeclarations: No` | Template + return type on one line is unreadable | `Yes` always |

## See also

- https://clang.llvm.org/docs/ClangFormatStyleOptions.html
- https://zed0.co.uk/clang-format-configurator/
