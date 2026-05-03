# CMake - FetchContent

> **Source**: https://cmake.org/cmake/help/latest/module/FetchContent.html
> **Skill**: dev-suite skill `cmake` — see SKILL.md for the always-loaded quick reference.

## What this covers

How FetchContent fetches and integrates dependencies at configure time, the difference between `FetchContent_MakeAvailable` and the older two-step pattern, GIT_TAG pinning practices, the `FetchContent` <-> `find_package` integration via `FETCHCONTENT_TRY_FIND_PACKAGE_MODE`, and `OVERRIDE_FIND_PACKAGE` for declaring local sources as package providers.

## Deep dive

### Basic usage

```cmake
include(FetchContent)

FetchContent_Declare(
    fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG        10.2.1                 # exact tag, NOT a branch
    GIT_SHALLOW    TRUE                   # faster clone
    GIT_PROGRESS   TRUE
    EXCLUDE_FROM_ALL                      # don't include fmt's targets in ALL
    SYSTEM                                # treat fmt's includes as system headers (no warnings)
)

FetchContent_MakeAvailable(fmt)
target_link_libraries(myapp PRIVATE fmt::fmt)
```

`FetchContent_MakeAvailable` (CMake 3.14+) downloads, calls `add_subdirectory`, and registers the targets in one step. Use it.

### GIT_TAG: pin to commits, not branches

```cmake
GIT_TAG main          # BAD — clones main HEAD; reproducible builds break
GIT_TAG v1.2.3        # OK — but a tag can be moved
GIT_TAG abc1234def... # BEST — full commit SHA, immutable
```

Combined with `GIT_SHALLOW TRUE`, full SHAs are required (shallow clones can't fetch arbitrary refs from history). For tags, use `GIT_SHALLOW FALSE` if the tag isn't on the default branch tip.

### EXCLUDE_FROM_ALL and SYSTEM

- `EXCLUDE_FROM_ALL` (3.28+) keeps the fetched project's targets out of `cmake --build .` unless explicitly built. They still get built when something depends on them.
- `SYSTEM` (3.25+) marks the include directories as system headers, suppressing warnings from third-party code (essential when you use `-Werror`).

### Multiple dependencies

```cmake
FetchContent_Declare(spdlog
    GIT_REPOSITORY https://github.com/gabime/spdlog.git
    GIT_TAG        v1.14.1
    GIT_SHALLOW    TRUE
)
FetchContent_Declare(googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG        v1.15.2
    GIT_SHALLOW    TRUE
)
FetchContent_MakeAvailable(spdlog googletest)
```

Both fetched in parallel (3.24+) when running with `-DFETCHCONTENT_PARALLEL=ON` or via Ninja's job system.

### Per-dependency CMake options

Set options *before* `FetchContent_MakeAvailable`:

```cmake
set(FMT_INSTALL OFF)              # don't install fmt as part of `cmake --install`
set(BUILD_TESTING OFF)            # most projects respect this
set(BENCHMARK_ENABLE_TESTING OFF) # google/benchmark
FetchContent_MakeAvailable(fmt benchmark)
```

Many libraries respect a `<PROJ>_*` option set this way. Check the library's CMakeLists for the exact option names.

### find_package integration (CMake 3.24+)

```cmake
# Globally: try find_package first, fall back to FetchContent
set(FETCHCONTENT_TRY_FIND_PACKAGE_MODE OPT_IN)

FetchContent_Declare(fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG        10.2.1
    FIND_PACKAGE_ARGS 10 CONFIG    # try find_package(fmt 10 CONFIG) first
)
FetchContent_MakeAvailable(fmt)
```

This lets system-installed packages win when present (faster builds, distro updates) and silently falls back to the source for users without it. `FETCHCONTENT_TRY_FIND_PACKAGE_MODE` accepts `OPT_IN` (per-declaration), `ALWAYS`, or `NEVER`.

### OVERRIDE_FIND_PACKAGE: provide a package from source

```cmake
FetchContent_Declare(fmt
    GIT_REPOSITORY ...
    GIT_TAG        10.2.1
    OVERRIDE_FIND_PACKAGE
)
```

After this declare, any `find_package(fmt)` (anywhere in the build, including transitive dependencies) is satisfied by your fetched copy. Useful when a downstream library does `find_package(fmt)` but you want to control the version.

### Build-time vs configure-time fetching

FetchContent runs at *configure* time. The first configure clones; subsequent configures are no-ops if the source dir exists. Cache the source dir:

- Set `FETCHCONTENT_BASE_DIR=/persistent/cache/dir` to share clones across build trees.
- In CI, cache `${CMAKE_BINARY_DIR}/_deps` between runs to avoid re-cloning.

### When NOT to use FetchContent

- For widely-installed system libs (use `find_package`).
- For huge dependencies (Qt, Boost) — use vcpkg/Conan/system packages.
- When you need binary caching across machines (use Conan or a real package manager).

FetchContent shines for small/medium libraries and CI determinism.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| `GIT_TAG main` | Non-reproducible build | Pin to a SHA or signed tag |
| Forgot `GIT_SHALLOW TRUE` | Slow CI clones | Add it; use full SHA if tag isn't on tip |
| `BUILD_TESTING ON` propagates to deps | Builds gtest's own tests, etc. | `set(BUILD_TESTING OFF)` before MakeAvailable, then re-enable for your project |
| Fetched lib's warnings break `-Werror` | Their headers fail your strict compile | Add `SYSTEM` keyword |
| Two deps each fetch the same lib at different versions | Diamond dependency conflict | First `FetchContent_Declare` wins; use `OVERRIDE_FIND_PACKAGE` to control |
| Calling `cmake --install` includes the fetched lib | They share the same install destination | Set per-lib `<PROJ>_INSTALL OFF` if available, or use `EXCLUDE_FROM_ALL` |

## See also

- https://cmake.org/cmake/help/latest/module/FetchContent.html — full module reference
- https://cmake.org/cmake/help/latest/module/ExternalProject.html — older sibling that fetches at build time
- https://github.com/cpm-cmake/CPM.cmake — wrapper that adds dependency caching
