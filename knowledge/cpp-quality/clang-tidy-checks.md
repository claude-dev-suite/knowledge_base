# C++ Quality - clang-tidy Check Categories Deep Dive

> **Source**: https://clang.llvm.org/extra/clang-tidy/checks/list.html
> **Skill**: dev-suite skill `quality/cpp-quality` — see SKILL.md for the always-loaded quick reference.

## What this covers

What each major clang-tidy check category actually flags, with concrete code examples of the
patterns each catches. Use this when you want to choose a curated check set instead of enabling
"everything" and triaging hundreds of false positives.

## Deep dive

### `bugprone-*` — real defects

| Check | Catches |
|-------|---------|
| `bugprone-use-after-move` | Reading from a moved-from object |
| `bugprone-string-integer-assignment` | `std::string s = 'a';` (assigns ASCII 97) |
| `bugprone-sizeof-expression` | `sizeof(ptr)` instead of `sizeof(*ptr)` |
| `bugprone-infinite-loop` | Loop with no exit and no side-effect |
| `bugprone-unhandled-self-assignment` | Op= without self-assignment guard |
| `bugprone-narrowing-conversions` | Implicit narrowing in initialization |
| `bugprone-suspicious-string-compare` | `strcmp(a, b)` then `if (cmp)` instead of `if (cmp == 0)` |

```cpp
std::string s = std::move(other);
log("moved: {}", other);   // bugprone-use-after-move
```

### `cert-*` — SEI CERT C++ rules

`cert-*` is mostly aliases over `bugprone-*` and `cppcoreguidelines-*`, plus a few unique:

- `cert-err58-cpp` — non-trivial static initialization that can throw
- `cert-oop54-cpp` — gracefully handle self-copy
- `cert-dcl58-cpp` — modifying `std` namespace

If you turn on `cppcoreguidelines-*` and `bugprone-*`, you get most of `cert-*` for free.
Enable `cert-*` explicitly only if you need SEI CERT compliance reporting.

### `modernize-*` — suggest modern equivalents

| Check | Suggests |
|-------|----------|
| `modernize-use-nullptr` | `nullptr` instead of `0`/`NULL` |
| `modernize-use-auto` | `auto` for verbose types |
| `modernize-loop-convert` | Range-for instead of index loop |
| `modernize-make-unique`/`-make-shared` | `std::make_unique<T>(...)` instead of `new` |
| `modernize-use-override` | `override` keyword |
| `modernize-use-default-member-init` | `int x{0};` in declaration |
| `modernize-use-emplace` | `emplace_back` over `push_back(T(...))` |

Often disabled: `modernize-use-trailing-return-type` (cosmetic, divisive),
`modernize-avoid-c-arrays` (false positives on extern "C" interop).

### `performance-*` — measurable wins

| Check | Catches |
|-------|---------|
| `performance-unnecessary-copy-initialization` | `auto v = container[i];` when `const auto&` would do |
| `performance-for-range-copy` | `for (auto x : v)` for non-trivial `x` |
| `performance-inefficient-string-concatenation` | `s = s + "x"` in a loop |
| `performance-move-const-arg` | `std::move` on a `const` (no-op) |
| `performance-unnecessary-value-param` | Pass-by-value of non-trivial type that isn't moved |
| `performance-no-int-to-ptr` | Casting int→ptr (sanitizer-hostile) |

```cpp
for (auto entry : large_map) { ... }  // performance-for-range-copy
//   ^^^^ should be const auto&
```

### `cppcoreguidelines-*` — Stroustrup/Sutter rules

The biggest category, with significant overlap with the others. Highlights:

- `cppcoreguidelines-pro-bounds-pointer-arithmetic` — no `p + 1` outside `std::span`
- `cppcoreguidelines-pro-type-reinterpret-cast` — `reinterpret_cast` is forbidden
- `cppcoreguidelines-narrowing-conversions` — alias of `bugprone-narrowing-conversions`
- `cppcoreguidelines-special-member-functions` — Rule of 5
- `cppcoreguidelines-avoid-non-const-global-variables`
- `cppcoreguidelines-init-variables` — every variable initialized at declaration

Often disabled: `cppcoreguidelines-avoid-magic-numbers` (too noisy for embedded),
`cppcoreguidelines-pro-bounds-array-to-pointer-decay` (interferes with C interop).

### `clang-analyzer-*` — path-sensitive

Runs the full Clang Static Analyzer engine, which is path-sensitive (tracks values across
branches). Catches:

- Null dereference along a specific path
- Memory leaks (forgot `delete` along an exception path)
- Use-after-free
- Division by zero

Most expensive category; consider running it less often than the rest (e.g. nightly).

### Recommended starter set

```yaml
Checks: >
  -*,
  bugprone-*,
  cert-*,
  clang-analyzer-*,
  concurrency-*,
  cppcoreguidelines-*,
  modernize-*,
  performance-*,
  portability-*,
  readability-*,
  -modernize-use-trailing-return-type,
  -modernize-avoid-c-arrays,
  -readability-magic-numbers,
  -readability-identifier-length,
  -cppcoreguidelines-avoid-magic-numbers,
  -cppcoreguidelines-pro-bounds-array-to-pointer-decay
WarningsAsErrors: 'bugprone-*,cert-*,clang-analyzer-*,concurrency-*'
```

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Enabling `cppcoreguidelines-avoid-magic-numbers` in scientific code | Every constant in a formula is "magic" | Disable; use `-Wno` for the category |
| `modernize-loop-convert` rewriting performance-critical loops | New form may pessimize for some compilers | Verify benchmarks; keep manual loops where measured |
| `cppcoreguidelines-special-member-functions` spam | Triggers on every class with a destructor | Enable `AllowSoleDefaultDtor: true` option |
| `readability-identifier-naming` enforcing one style globally | Library and your code clash | Set per-directory `.clang-tidy` with different naming options |
| Treating `clang-analyzer-*` warnings as errors | False positives kill CI | Errors only for `bugprone-*` initially, ratchet others |

## See also

- https://clang.llvm.org/extra/clang-tidy/checks/bugprone/
- https://wiki.sei.cmu.edu/confluence/display/cplusplus/SEI+CERT+C%2B%2B+Coding+Standard
