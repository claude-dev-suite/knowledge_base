# C++ - Modules (C++20)

> **Source**: https://en.cppreference.com/w/cpp/language/modules
> **Skill**: dev-suite skill `cpp` — see SKILL.md for the always-loaded quick reference.

## What this covers

What a module *actually* is from the compiler's perspective (a precompiled binary representation), the difference between primary interface units, partitions, and implementation units, how `import` differs from `#include`, and the practical state of toolchain support (CMake 3.28+, Clang, MSVC, GCC 14+).

## Deep dive

### Why modules exist

A `#include` re-tokenizes and re-parses the header for every translation unit. Modules compile the interface *once* into a binary module interface (BMI), then `import` reads the BMI. This eliminates header repetition, macro pollution, and ODR pitfalls — at the cost of a stricter dependency graph the build system must compute.

### Anatomy of a module

```cpp
// matrix.cppm  -- primary module interface unit
module;                              // global module fragment (legacy includes go here)
#include <cstddef>                   // headers that pre-date module support

export module geom.matrix;           // module declaration

import std;                          // C++23 std module (libc++/MSVC)
import :detail;                      // import a partition (declared below)

export namespace geom {              // export an entire namespace
    template <std::size_t R, std::size_t C> class Matrix {
        /* ... */
    public:
        Matrix transpose() const;
    };
}

// matrix-detail.cppm  -- module partition interface
export module geom.matrix:detail;
namespace geom::detail {
    template <typename T> T determinant_lu(T const&);  // exported only within geom.matrix
}

// matrix-impl.cpp  -- implementation unit (no `export`)
module geom.matrix;
namespace geom {
    template <std::size_t R, std::size_t C>
    Matrix<R, C> Matrix<R, C>::transpose() const { /* ... */ }
}

// main.cpp
import geom.matrix;
int main() { geom::Matrix<3, 3> m; m.transpose(); }
```

Key rules:

- **Primary interface unit**: exactly one per module, contains `export module name;`.
- **Partitions**: `export module name:part;` — visible only inside the same module unless the primary re-exports.
- **Implementation units**: `module name;` (no `export`) — like `.cpp` files for the module.
- Names not marked `export` are reachable but not visible to importers.

### Macros do not cross module boundaries

```cpp
// foo.cppm
export module foo;
#define HIDDEN 42
export int get() { return HIDDEN; }

// main.cpp
import foo;
int x = HIDDEN;   // ERROR — HIDDEN not visible
int y = get();    // OK — returns 42
```

This is the headline benefit. If you genuinely need a macro across boundaries, use a header (still allowed) or a `header unit` (`import <vector>;` if your toolchain enables it).

### CMake 3.28+ support

```cmake
cmake_minimum_required(VERSION 3.28)
project(myapp CXX)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_SCAN_FOR_MODULES ON)

add_library(geom)
target_sources(geom
    PUBLIC FILE_SET CXX_MODULES FILES
        src/matrix.cppm
        src/matrix-detail.cppm
    PRIVATE
        src/matrix-impl.cpp
)
target_compile_features(geom PUBLIC cxx_std_20)

add_executable(app src/main.cpp)
target_link_libraries(app PRIVATE geom)
```

Use the `Ninja` generator (1.11+); `Make` does not handle the dynamic dependency scan well. As of 2026, MSVC and Clang have the most complete support; GCC 14+ is workable but partition support lags.

### Migration strategy

Don't rewrite. Start by:

1. Enabling `import std;` in new translation units.
2. Adding new code as modules (`*.cppm`).
3. Leaving legacy headers as headers; mix freely via the global module fragment.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| `#include` after `export module` declaration | Includes are "attached" to the module → ODR risk | Put includes in the global module fragment (`module;` block before `export module`) |
| Mixing modules and headers for the same library | Two views of the same entity | Pick one boundary; library = module **or** library = header |
| BMI cache going stale | Tools generate per-flag BMIs | Use one preset / build dir per flag set |
| Templates in implementation units | Can't be instantiated by importers | Define template members in the interface unit |
| `export namespace ns { ... }` then adding non-exported names later | Compiler may export them anyway | Be explicit per-declaration when adding incrementally |
| Using `import std;` without compiler/std-lib support | Compile error | Fall back to per-header imports `import <vector>;` |

## See also

- https://en.cppreference.com/w/cpp/language/modules — full grammar and semantics
- https://www.kitware.com/import-cmake-the-experiment-is-over/ — CMake module support history
- https://clang.llvm.org/docs/StandardCPlusPlusModules.html — Clang implementation status
