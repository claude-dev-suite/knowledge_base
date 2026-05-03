# C++ Quality - Include-What-You-Use (IWYU)

> **Source**: https://include-what-you-use.org/
> **Skill**: dev-suite skill `quality/cpp-quality` — see SKILL.md for the always-loaded quick reference.

## What this covers

Why include hygiene matters for build times and binary size, how IWYU's mapping files work,
the difference between `// IWYU pragma: keep` and `// IWYU pragma: export`, and the
`fix_includes.py` workflow for applying suggestions safely.

## Deep dive

### What "include what you use" actually means

Every type, function, or macro your TU uses must be declared in a header that TU `#include`s
*directly*. Relying on transitive includes (header A includes B, you include A and use B) is
fragile — when A's maintainer drops the include, your code stops compiling.

IWYU computes the minimal direct-include set per source file. Result: faster builds (smaller
preprocessed input), fewer ripple-effect breakages when third-party headers change, and clearer
dependency graphs.

### Setup

IWYU is built against a specific Clang version. Mismatch causes "unknown intrinsic" errors:

```bash
# Build IWYU against the system Clang
git clone https://github.com/include-what-you-use/include-what-you-use.git
cd include-what-you-use && mkdir build && cd build
cmake -G "Unix Makefiles" -DCMAKE_PREFIX_PATH=/usr/lib/llvm-18 ..
make -j$(nproc)
```

Or from package managers: `apt install iwyu`, `brew install include-what-you-use`,
`vcpkg install include-what-you-use`.

### Running

```bash
# Single file
include-what-you-use -p build src/foo.cpp

# Whole project (Python wrapper)
iwyu_tool.py -p build > iwyu.out

# Apply suggestions in-place
fix_includes.py < iwyu.out

# Re-format after — IWYU's output is unsorted
clang-format -i src/**/*.cpp include/**/*.hpp
```

### Pragmas

| Pragma | Effect |
|--------|--------|
| `#include <foo.h>  // IWYU pragma: keep` | Don't suggest removing this include |
| `#include "foo.h"  // IWYU pragma: export` | Re-export foo.h's symbols (this header is a "facade") |
| `#include "foo_inl.h"  // IWYU pragma: associated` | Mark foo.h as the associated header for foo_inl.h |
| `// IWYU pragma: private, include "public.h"` | This header is private; users should include public.h |
| `class Foo;  // IWYU pragma: forward_declare` | Forward declaration is the right choice — don't suggest a full include |
| `// IWYU pragma: begin_exports` ... `// IWYU pragma: end_exports` | Multiple re-exports |

### Mapping files (IMP)

For library facades (Boost, Qt) that have many internal headers, IWYU uses `.imp` files to map
internal → public:

```
# qt5_x11extras.imp
[
    { include: ['"qt5/QtX11Extras/qx11info_x11.h"', private,
                '<QX11Info>',                       public ] },
    { include: ['<bits/string_fwd.h>',              private,
                '<string>',                         public ] },
]
```

```bash
iwyu_tool.py -p build -- -Xiwyu --mapping_file=qt5_x11extras.imp
```

IWYU ships with mappings for libstdc++, libc++, GTest, and others; you'll write your own only
for in-house facade headers.

### Forward declarations vs full includes

IWYU's default is "forward declare wherever you can." This often produces:

```cpp
// Suggested
class Logger;     // IWYU pragma: forward_declare
void log_to(Logger& l, std::string_view msg);
```

This is correct and faster to compile, but it can break consumers that previously got `Logger`'s
full declaration transitively. Run IWYU on the *consumers* too, or add `// IWYU pragma: keep`
on the full include.

### `--no_fwd_decls` for projects that prefer full includes

```bash
iwyu_tool.py -p build -- -Xiwyu --no_fwd_decls
```

Disables the forward-declare-where-possible behavior. Cleaner header files at the cost of
compile time.

### Common workflow

1. Run IWYU once, save output.
2. Apply with `fix_includes.py`.
3. Build — fix any breakages by adding `// IWYU pragma: keep` or actual missing includes.
4. Run IWYU again to confirm clean.
5. Add an `iwyu_check` make/CMake target that runs IWYU but only **warns** in CI. Treat as errors
   only after the codebase is clean.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| IWYU built against wrong Clang version | Crashes / "unknown attribute" errors | Match Clang versions exactly |
| Applying suggestions without re-running clang-format | Unsorted, awkward layout | Always `clang-format -i` after `fix_includes.py` |
| Forward-declared types used by value | Compile error: incomplete type | Add `// IWYU pragma: keep` on the full include |
| IWYU on generated code | Spurious "missing include" suggestions | Exclude generated dirs from `iwyu_tool.py` |
| Trusting IWYU on heavy template libraries | Suggestions for internal Boost / range-v3 headers | Use mapping files or `// IWYU pragma: keep` |
| Running IWYU in CI as fail-on-warning before cleanup | Constant red CI | Warn-only mode until baseline is clean |

## See also

- https://github.com/include-what-you-use/include-what-you-use/blob/master/docs/IWYUPragmas.md
- https://github.com/include-what-you-use/include-what-you-use/blob/master/docs/IWYUMappings.md
