# C++ - Core Guidelines (Stroustrup & Sutter)

> **Source**: https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines
> **Skill**: dev-suite skill `cpp` — see SKILL.md for the always-loaded quick reference.

## What this covers

How to actually use the Core Guidelines: the rule numbering scheme, the GSL helpers (`gsl::not_null`, `gsl::span`, `gsl::owner`), the most impactful sections (Interfaces, Resource management, Concurrency), and how to enforce a subset via clang-tidy's `cppcoreguidelines-*` checks.

## Deep dive

### Rule numbering and how to cite

Rules are categorized by prefix:

| Prefix | Topic | Example |
|--------|-------|---------|
| `P` | Philosophy | P.1 Express ideas directly in code |
| `I` | Interfaces | I.11 Never transfer ownership by a raw pointer |
| `F` | Functions | F.7 Pass by `T*` only when nullable |
| `C` | Classes | C.20 If you can avoid defining default operations, do |
| `Enum` | Enumerations | Enum.3 Prefer `enum class` |
| `R` | Resource mgmt | R.20 Use `unique_ptr` or `shared_ptr` to own |
| `ES` | Expressions/stmts | ES.1 Prefer the standard library |
| `Per` | Performance | Per.1 Don't optimize without reason |
| `CP` | Concurrency | CP.20 Use RAII, never plain `lock()`/`unlock()` |
| `E` | Error handling | E.6 Use RAII to prevent leaks |
| `T` | Templates | T.1 Use templates to raise the abstraction level |

In code reviews, cite as "C.21" or "ES.49" — rules are stable URLs.

### The GSL (Guidelines Support Library)

A small header-only library providing types the guidelines reference. Microsoft's `gsl-lite` and the original `Microsoft/GSL` are common.

```cpp
#include <gsl/gsl>

// not_null<T*> — pointer that cannot be null (asserts in debug, documents intent)
void render(gsl::not_null<Renderer*> r, gsl::span<const Vertex> verts);

// owner<T*> — annotates ownership for legacy APIs returning raw pointers
gsl::owner<Resource*> acquire();   // caller deletes
void release(gsl::owner<Resource*>);

// finally / scope_exit — ad-hoc RAII
auto cleanup = gsl::finally([&]{ close(fd); });

// narrow / narrow_cast — explicit narrowing conversion
int n = gsl::narrow<int>(some_size_t);   // throws if value doesn't fit
```

C++20/23 standardize many of these (`std::span`, `std::source_location`); GSL fills the remaining gaps.

### Highest-leverage rules to internalize

These appear in nearly every code review:

- **F.15** — Prefer simple and conventional ways of passing information. The infamous "pass by value or by const reference" decision tree.
- **F.21** — To return multiple values, prefer returning a struct (or `tuple`).
- **C.20** — Rule of zero: if you can, write *no* special members.
- **C.21** — Rule of five: if you write one, audit them all.
- **R.30** — Take smart pointers as parameters only to express lifetime semantics. (`void f(unique_ptr<T>)` = "I take ownership"; `void f(T*)` = "I observe".)
- **ES.5** — Keep scopes small.
- **ES.20** — Always initialize objects.
- **NR.1** — "Don't use a singleton" — debunks the no-rule.
- **CP.22** — Never call unknown code while holding a lock.
- **Per.7** — Design to enable optimization.

### Enforcement via clang-tidy

```yaml
# .clang-tidy
Checks: >
  cppcoreguidelines-*,
  -cppcoreguidelines-avoid-magic-numbers,
  -cppcoreguidelines-pro-type-vararg,
  modernize-*,
  bugprone-*,
  performance-*
WarningsAsErrors: '*'
```

Useful checks worth keeping enabled:

| Check | Catches |
|-------|---------|
| `cppcoreguidelines-init-variables` | Uninitialized locals |
| `cppcoreguidelines-pro-bounds-*` | Pointer arithmetic, array decay |
| `cppcoreguidelines-rvalue-reference-param-not-moved` | Forgotten `std::move` |
| `cppcoreguidelines-special-member-functions` | Rule of 5 violations |
| `cppcoreguidelines-virtual-class-destructor` | Missing `virtual ~` on base |
| `modernize-use-nullptr` | Legacy `0`/`NULL` |

Wire it into CMake via `set(CMAKE_CXX_CLANG_TIDY clang-tidy)`.

### When to deviate

The guidelines are *guidelines*, not laws. Common, justified deviations:

- Game/embedded code disabling exceptions (E.* rules adapt — use `std::expected`).
- Performance-critical code using raw pointers internally (still wrap at the API boundary).
- ABI-stable libraries pinning to C++17 features.

Document deviations near the code (`// CG: F.7 — nullable observer pointer`).

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Treating the guidelines as dogma | Some are dated; not all apply to every domain | Pick a subset, enforce via tooling |
| Enabling all `cppcoreguidelines-*` checks at once | Hundreds of warnings flood the build | Roll out incrementally, fix per-check |
| Ignoring `pro-bounds-*` warnings as noise | They catch real OOB bugs | Fix with `gsl::span`/`std::span` instead of disabling |
| Citing rules by name from memory | Names change | Always link the URL with the rule ID |
| Believing "don't use exceptions" is a guideline | It's not — guidelines *use* exceptions | If banned in your project, document the alternative |

## See also

- https://github.com/isocpp/CppCoreGuidelines — source repository (PRs accepted)
- https://github.com/microsoft/GSL — Microsoft's GSL implementation
- https://clang.llvm.org/extra/clang-tidy/checks/list.html — full clang-tidy check catalog
