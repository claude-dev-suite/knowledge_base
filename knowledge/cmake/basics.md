# CMake - Project Basics and Structure

> **Source**: https://cmake.org/cmake/help/latest/guide/tutorial/index.html
> **Skill**: dev-suite skill `cmake` — see SKILL.md for the always-loaded quick reference.

## What this covers

How CMake's two-phase model (configure -> generate -> build) actually works, what `cmake_minimum_required` controls (policies, not just version), the structure of `project()` and the variables it sets, the differences between executable/static/shared/object/interface libraries, and how to organize a multi-directory project.

## Deep dive

### The two-phase model

1. **Configure** (`cmake -S . -B build`) — runs your CMake scripts, populates the cache, generates the buildsystem files (Ninja, Make, .vcxproj, ...).
2. **Build** (`cmake --build build`) — invokes the generated buildsystem.

Reconfiguration is automatic when CMakeLists.txt changes, but adding/removing files via `file(GLOB ...)` does *not* re-glob unless you reconfigure. List sources explicitly.

### cmake_minimum_required does more than gatekeep

```cmake
cmake_minimum_required(VERSION 3.25)
```

This sets *policy defaults* up to 3.25. CMake policies (CMP0xxx) control behavior changes between versions — without this, you get OLD behavior for unknown policies (warnings galore). Bump to the highest version you can require; don't pin to ancient values.

For libraries, you can specify a range to opt into newer behaviors while keeping the floor low:

```cmake
cmake_minimum_required(VERSION 3.20...3.30)
```

### project() side effects

```cmake
project(myapp
    VERSION 1.4.2
    DESCRIPTION "A widget renderer"
    HOMEPAGE_URL "https://example.com/myapp"
    LANGUAGES CXX C
)
```

Sets `PROJECT_NAME`, `PROJECT_VERSION`, `PROJECT_VERSION_MAJOR/MINOR/PATCH`, `PROJECT_SOURCE_DIR`, `PROJECT_BINARY_DIR`. Also probes the toolchain — *don't* call `find_package`, `set(CMAKE_*_FLAGS ...)`, or detect the compiler before this.

Always specify `LANGUAGES`. Default is `C CXX`, which fails the configure if a C compiler isn't available even when you only need C++.

### Library types

```cmake
add_library(mylib STATIC src/a.cpp src/b.cpp)   # libmylib.a / mylib.lib
add_library(mylib SHARED src/a.cpp src/b.cpp)   # libmylib.so / mylib.dll
add_library(mylib OBJECT src/a.cpp src/b.cpp)   # .o files only, no archive

# Header-only library — INTERFACE has no sources
add_library(myhdr INTERFACE)
target_include_directories(myhdr INTERFACE include)

# Type chosen by BUILD_SHARED_LIBS
add_library(mylib src/a.cpp src/b.cpp)
```

`OBJECT` libraries are useful when the same sources need to feed into multiple final libraries with different flags. `INTERFACE` libraries carry usage requirements (include dirs, definitions, link libs) without producing binary output — ideal for header-only libs and "configuration bundles".

### Multi-directory layout

```
myapp/
├── CMakeLists.txt              # top-level
├── CMakePresets.json
├── cmake/                      # custom CMake modules
│   └── MyAppOptions.cmake
├── src/
│   ├── CMakeLists.txt
│   ├── core/
│   │   ├── CMakeLists.txt
│   │   └── widget.cpp
│   └── ui/
│       ├── CMakeLists.txt
│       └── window.cpp
├── include/myapp/              # public headers
│   └── widget.hpp
├── tests/
│   └── CMakeLists.txt
└── apps/
    └── CMakeLists.txt
```

Top-level:

```cmake
cmake_minimum_required(VERSION 3.25)
project(myapp VERSION 1.0 LANGUAGES CXX)

list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

option(MYAPP_BUILD_TESTS "Build tests" ${PROJECT_IS_TOP_LEVEL})

add_subdirectory(src)
add_subdirectory(apps)
if(MYAPP_BUILD_TESTS)
    enable_testing()
    add_subdirectory(tests)
endif()
```

`PROJECT_IS_TOP_LEVEL` (3.21+) lets the same CMakeLists work standalone *and* as an `add_subdirectory` dependency without forcing tests on consumers.

### Variables you'll actually use

| Variable | Meaning |
|----------|---------|
| `CMAKE_SOURCE_DIR` | Top-level source (the one passed to `-S`) |
| `CMAKE_BINARY_DIR` | Top-level build (the one passed to `-B`) |
| `CMAKE_CURRENT_SOURCE_DIR` | Source dir of the current `CMakeLists.txt` |
| `CMAKE_CURRENT_BINARY_DIR` | Build dir matching above |
| `PROJECT_*` | Per-`project()` equivalents |
| `CMAKE_BUILD_TYPE` | Single-config generators only |
| `CMAKE_CONFIGURATION_TYPES` | Multi-config generators (Visual Studio, Xcode, Ninja Multi-Config) |

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| `cmake_minimum_required(VERSION 3.0)` | Disables every policy improvement since 2014 | Set to the actual minimum your code needs |
| Calling `project()` after `find_package` | Toolchain not yet probed | Always call `project()` first |
| Hardcoding `set(CMAKE_BUILD_TYPE Release)` in CMakeLists | Overrides user/IDE choice | Set in preset or detect: `if(NOT CMAKE_BUILD_TYPE) ...` |
| Using `CMAKE_SOURCE_DIR` in a subproject | Refers to outer project when `add_subdirectory`'d | Use `PROJECT_SOURCE_DIR` |
| `file(GLOB)` for sources | Stale until reconfigure | List files explicitly; CMake 3.12+ `CONFIGURE_DEPENDS` reduces but doesn't fix |
| Mixing `add_executable(myapp WIN32 ...)` cross-platform | `WIN32` is silently ignored elsewhere; on Win it hides console | Set `WIN32_EXECUTABLE` per-target conditionally |

## See also

- https://cmake.org/cmake/help/latest/manual/cmake-buildsystem.7.html — buildsystem manual
- https://cmake.org/cmake/help/latest/command/project.html — `project()` reference
- https://cliutils.gitlab.io/modern-cmake/ — community "Modern CMake" guide
