# C++ Security - UndefinedBehaviorSanitizer (UBSan)

> **Source**: https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html
> **Skill**: dev-suite skill `security/cpp-security` — see SKILL.md for the always-loaded quick reference.

## What this covers

UBSan's individual sub-checks (it's not a single sanitizer but a suite of ~25), trade-offs of
trap-mode vs runtime-mode, integer overflow handling, and which checks are safe enough to ship
in release builds for defense in depth.

## Deep dive

### UBSan is a collection — pick what you need

`-fsanitize=undefined` is shorthand for many checks. You can enable subsets:

| Check | What it catches |
|-------|-----------------|
| `signed-integer-overflow` | `INT_MAX + 1` |
| `unsigned-integer-overflow` | Optional, since unsigned overflow is **defined** in C/C++ — only enable if you want to find unintentional overflow |
| `shift` (= `shift-base` + `shift-exponent`) | `1 << 32`, `x << -1` |
| `integer-divide-by-zero`, `float-divide-by-zero` | Self-explanatory |
| `null` | Null pointer load/store, member access |
| `bool` | Loading non-{0,1} into `bool` |
| `enum` | Loading out-of-range value into a strict enum |
| `vptr` | Wrong vtable for the dynamic type — catches use-after-free of polymorphic object |
| `bounds` | Array OOB *for static-size arrays* (similar to ASan but compile-time) |
| `alignment` | Misaligned load/store (sub-architecture penalty / SIGBUS) |
| `object-size` | OOB detected via `__builtin_object_size` |
| `pointer-overflow` | Pointer arithmetic that wraps |
| `function` | Indirect call through wrong function-pointer type |
| `return` | Falling off a non-void function |
| `unreachable` | Reaching `__builtin_unreachable()` |
| `nullability-arg`, `nullability-return` | Null where `_Nonnull` was annotated |

Enable the production-safe subset always; use the full set in CI:

```bash
# CI / dev — comprehensive
clang++ -fsanitize=undefined,nullability \
        -fno-sanitize-recover=all \
        -fno-omit-frame-pointer -g -O1 ...

# Production — cheap, defensive
clang++ -fsanitize=signed-integer-overflow,shift,null,bounds,vptr \
        -fsanitize-trap=signed-integer-overflow,shift \
        -O2 ...
```

### Trap mode vs runtime mode

Default UBSan calls a runtime function on violation that prints a diagnostic. Trap mode emits a
`ud2`/`brk` instruction:

```bash
-fsanitize-trap=signed-integer-overflow,bounds
# or all of UBSan:
-fsanitize-trap=undefined
```

Trap mode:
- Zero binary-size cost for the runtime
- No diagnostics, just SIGILL (so `coredump` and post-mortem)
- Fast — comparable to native code
- Good for production "fail-closed" defense

Runtime mode:
- Helpful "expression evaluated to ... at file:line" messages
- Slightly larger binary
- Default for `-fsanitize=undefined`

### `-fno-sanitize-recover` — fail fast

By default, UBSan reports and continues. Make it abort:

```bash
-fno-sanitize-recover=all
# or:
UBSAN_OPTIONS=halt_on_error=1:print_stacktrace=1 ./myapp
```

Continue-mode is useful to catch many UB sites in one run; halt-mode is what you want in CI to
avoid a single test "passing" with 50 UB warnings buried in the log.

### Combining with ASan

UBSan composes with ASan with no conflict:

```bash
clang++ -fsanitize=address,undefined -fno-omit-frame-pointer -g -O1 ...
```

Total runtime cost: ~2.5x. Both error reports interleave on stderr.

### Integer overflow specifically

Signed overflow is UB in C and C++. The pragmatic options:

1. **Compile with UBSan in CI** — find the spots.
2. **Use checked arithmetic** for boundary code (parsers, indices):

```cpp
#include <limits>
#include <stdexcept>

template <std::integral T>
T checked_mul(T a, T b) {
    T result;
    if (__builtin_mul_overflow(a, b, &result))
        throw std::overflow_error("multiplication overflow");
    return result;
}
```

3. **Mark hot intentional wrapping** with `[[gnu::no_sanitize("signed-integer-overflow")]]` or use
   `__builtin_add_overflow` family.

### Suppression file (rare, but)

```
# ubsan.supp
signed-integer-overflow:vendor/lib.cpp
function:third_party/legacy.cpp
```

```bash
UBSAN_OPTIONS=suppressions=ubsan.supp:print_stacktrace=1 ./myapp
```

### MSVC equivalents

MSVC doesn't ship UBSan. The closest are `/RTC1` (runtime checks for variables, stack frames),
`/sdl` (lots of runtime safety), and `/GS` (stack canary). Use ASan + UBSan via clang-cl in CI
even on Windows projects.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Enabling `unsigned-integer-overflow` blindly | Wrap is defined; many libs rely on it | Enable only if you have a specific bug to hunt |
| `vptr` check without RTTI | Can't determine dynamic type | Build with `-frtti` (default on Clang) |
| Trap mode in CI | No diagnostic to debug from | Runtime mode in CI; trap mode in production binaries |
| Mixing UBSan and `-fwrapv` | `-fwrapv` makes signed wrap defined → UBSan sees no UB | Pick one philosophy |
| Forgetting `-fno-sanitize-recover=all` | Tests pass while UB happens | Always halt-on-error in CI |
| Not linking with `-fsanitize=undefined` | Missing ubsan_runtime → undefined references | Pass to compile AND link |

## See also

- https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html#available-checks
- https://gcc.gnu.org/onlinedocs/gcc/Instrumentation-Options.html
