# C++ Quality - cppcheck

> **Source**: https://cppcheck.sourceforge.io/manual.pdf
> **Skill**: dev-suite skill `quality/cpp-quality` — see SKILL.md for the always-loaded quick reference.

## What this covers

What cppcheck catches that clang-tidy doesn't (and vice versa), enable/severity levels,
inline and project-level suppressions, addons (MISRA, CERT), and how to integrate cppcheck
without duplicating clang-tidy's noise.

## Deep dive

### Why run both clang-tidy AND cppcheck?

They overlap, but each finds bugs the other misses:

- **cppcheck** is a custom static analyzer that does *symbolic execution* with a focus on raw
  pointer aliasing, uninitialized member tracking across constructors, and dangling references.
  It works without `compile_commands.json` (best-effort include resolution).
- **clang-tidy** uses Clang's AST and the Clang Static Analyzer for path-sensitive analysis.
  Stronger for modern C++ idioms; needs `compile_commands.json`.

Concrete examples cppcheck catches that clang-tidy commonly misses:

```cpp
class Widget {
    int x;
    int y;
    Widget(int v) : x(v) {}     // cppcheck: uninitMemberVar — y not initialized
};

void f(int* p, int* q) {
    *p = 0;
    *q = 1;
    return *p;                  // possibly *p == 1 if p == q (aliasing)
}
```

### Severity levels

```bash
cppcheck --enable=warning,style,performance,portability,information \
         --inline-suppr \
         --error-exitcode=2 \
         --project=build/compile_commands.json \
         --suppress=missingIncludeSystem \
         -i tests/ -i build/ \
         src/
```

| Level | Meaning |
|-------|---------|
| `error` | Always on; certain bugs |
| `warning` | Likely bugs |
| `style` | Style/readability |
| `performance` | Suboptimal patterns |
| `portability` | Non-portable constructs |
| `information` | Hints |

`--enable=all` is generally too noisy; pick categories explicitly.

### Inline suppressions

```cpp
// cppcheck-suppress unusedVariable
int debug_only;

// cppcheck-suppress-begin uninitMemberVar
class LegacyApi { /* ... */ };
// cppcheck-suppress-end uninitMemberVar
```

Always include the rule name. `--inline-suppr` (CLI flag) is required for them to take effect.

### Project-level suppressions

`suppressions.txt`:
```
missingIncludeSystem
unmatchedSuppression
*:third_party/*
uninitMemberVar:src/legacy/*
```

```bash
cppcheck --suppressions-list=suppressions.txt ...
```

The `unmatchedSuppression` line is critical: without it, suppressions for files no longer in the
project quietly become errors.

### MISRA addon

```bash
cppcheck --addon=misra --project=build/compile_commands.json src/
```

Checks against MISRA C++ 2008 / 2023 rules. Useful for embedded/automotive; requires the rule
texts (which are licensed) to get full descriptions, but rule numbers are reported either way.

### CERT addon

```bash
cppcheck --addon=cert --project=build/compile_commands.json src/
```

Maps cppcheck checks to SEI CERT C++ rule IDs in the output. Useful for compliance reporting.

### Integration into a CI that already runs clang-tidy

Run cppcheck *only* with rules clang-tidy doesn't already cover. Suggested config:

```bash
cppcheck --enable=warning,performance,portability \
         --suppress=missingIncludeSystem \
         --suppress=missingInclude \
         --suppress=unmatchedSuppression \
         --suppress=uninitMemberVar:tests/* \
         --suppress=normalCheckLevelMaxBranches \
         --inline-suppr \
         --error-exitcode=2 \
         -j$(nproc) \
         --project=build/compile_commands.json
```

Skip `--enable=style` if clang-tidy `readability-*` is on — duplication.

### XML output for CI dashboards

```bash
cppcheck --xml --xml-version=2 ... 2> cppcheck-result.xml
cppcheck-htmlreport --file=cppcheck-result.xml --report-dir=cppcheck-html
```

For SonarQube/Jenkins consumption use the XML format and the included plugin.

### `--check-level=exhaustive` (cppcheck 2.11+)

```bash
cppcheck --check-level=exhaustive ...
```

Increases analysis depth (more value-flow steps). 2-5x slower but catches deeper aliasing /
loop-related bugs. Worth running on nightly builds.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| `--enable=all` then drowning in style noise | Includes `style` and `information` at full volume | Pick categories explicitly |
| Running without `--project` | cppcheck guesses includes → false positives on macros | Always pass `--project=compile_commands.json` |
| Forgetting `--inline-suppr` | Inline `// cppcheck-suppress` lines silently ignored | Always include the flag in CI |
| `unmatchedSuppression` not suppressed | Stale entries in suppressions.txt break CI | Add `unmatchedSuppression` to suppressions |
| Running cppcheck on the build/ dir | Lints CMake-generated artifacts | `-i build/` |
| Not parallelizing | Slow on large codebases | `-j$(nproc)` (cppcheck 2.0+) |

## See also

- https://cppcheck.sourceforge.io/
- https://github.com/danmar/cppcheck/blob/main/man/manual.md
