# C++ Quality - clang-tidy Setup

> **Source**: https://clang.llvm.org/extra/clang-tidy/
> **Skill**: dev-suite skill `quality/cpp-quality` — see SKILL.md for the always-loaded quick reference.

## What this covers

How clang-tidy actually finds your code (`compile_commands.json`), how `.clang-tidy` files are
discovered and merged across the source tree, parallelization with `run-clang-tidy`, and
strategies for incremental adoption that don't require fixing the entire codebase up front.

## Deep dive

### `compile_commands.json` is non-negotiable

clang-tidy needs the *exact* compiler invocation per source file: include paths, defines, language
standard, target triple. Generate with CMake:

```cmake
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
```

Or with Bear (for non-CMake builds):

```bash
bear -- make -j$(nproc)
```

Without it, clang-tidy uses `-I` based on guesses → false positives like "unknown identifier"
on every type from a third-party header.

### `.clang-tidy` discovery — closest wins

clang-tidy walks **upward** from each source file looking for a `.clang-tidy` file. The closest
one wins, but with `InheritParentConfig: true` the closer file *extends* its parent rather than
replacing it:

```yaml
# project root .clang-tidy — strict
Checks: '-*,bugprone-*,cert-*,modernize-*,performance-*'
WarningsAsErrors: '*'

# vendor/third_party/.clang-tidy — disable for vendored code
Checks: '-*'
WarningsAsErrors: ''
InheritParentConfig: false
```

This is the cleanest way to opt out vendored code without globbing in CI.

### `run-clang-tidy` — parallel & filtered

```bash
run-clang-tidy -p build/                                     \
               -j$(nproc)                                    \
               -header-filter='^.*/(include|src)/.*\.(h|hpp)$' \
               -quiet                                         \
               'src/.*\.cpp$'                                # positional regex of files
```

`-header-filter` is critical: without it, every diagnostic in every system header gets reported
(thousands of lines). The regex matches **headers included by** the source files clang-tidy is
analyzing — it doesn't make clang-tidy analyze headers directly (it analyzes the TU and reports
diagnostics found in matching headers).

### Auto-fix workflow

```bash
# Apply suggested fixes in-place; -fix-errors also applies fixes for diagnostics
# that clang-tidy considered errors (default is to skip those for safety).
run-clang-tidy -p build/ -fix -fix-errors -j$(nproc) 'src/.*'
clang-format -i src/**/*.cpp        # re-format after — fixes can be ugly
```

Always run inside a clean git tree so you can `git diff` the changes and revert per-file.

### Incremental adoption — three useful patterns

**Pattern 1: lint only changed files** (PR-friendly, fast):

```bash
git diff --name-only --diff-filter=AM origin/main...HEAD -- '*.cpp' '*.hpp' \
  | xargs -r clang-tidy -p build/ --quiet
```

**Pattern 2: ratchet — fail only on increases**:

```bash
TIDY=$(run-clang-tidy -p build -quiet 2>&1 | grep -c 'warning:')
BASELINE=$(cat .tidy-baseline)
if [ "$TIDY" -gt "$BASELINE" ]; then echo "regressed"; exit 1; fi
echo "$TIDY" > .tidy-baseline
```

**Pattern 3: per-directory enablement** — start strict in `core/`, lax in `legacy/`:

```yaml
# core/.clang-tidy — strict ruleset
# legacy/.clang-tidy — Checks: '-*' (silent)
```

### Suppressing precisely

```cpp
// Disable one diagnostic on the next line
int* p = (int*)x;  // NOLINT(cppcoreguidelines-pro-type-cstyle-cast)

// Block range
// NOLINTBEGIN(modernize-use-nodiscard, readability-identifier-naming)
extern "C" int legacy_api_INT();
// NOLINTEND(modernize-use-nodiscard, readability-identifier-naming)

// Whole-file (place at top)
// NOLINTBEGIN(*)
// ... entire file ...
// NOLINTEND(*)
```

Always include the check name in parentheses. Bare `// NOLINT` disables *every* diagnostic on that
line, which is almost always wrong.

### Treating warnings as errors selectively

```yaml
WarningsAsErrors: 'bugprone-*,cert-*,clang-analyzer-*'
```

Use a tighter set than `Checks:` — you want modernize-* suggestions visible but not blocking.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Running clang-tidy in tree that builds with MSVC | Clang's frontend rejects MSVC-only constructs | Use `clang-cl` driver, or build a separate `compile_commands.json` with clang |
| `-header-filter` missing → 5,000-line CI log | System headers spam | Always set it to your include/src tree |
| Forgetting `-fix-errors` and wondering why fixes weren't applied | Defaults skip diagnostics tagged as errors | Add `-fix-errors` once you trust the rules |
| `WarningsAsErrors: '*'` without `Checks` filter | Some checks have many false positives | Pin both to the same set; expand cautiously |
| `.clang-tidy` only at repo root in monorepo | Hard to opt out vendored code | Per-subdirectory `.clang-tidy` with `InheritParentConfig` |
| Generated code linted | Noise; can't fix | Add `--exclude-header-filter` or path-glob exclusion |

## See also

- https://clang.llvm.org/extra/clang-tidy/Integrations.html
- https://clang.llvm.org/extra/clang-tidy/checks/list.html
