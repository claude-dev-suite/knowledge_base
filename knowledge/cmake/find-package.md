# CMake - find_package (CONFIG vs MODULE Mode)

> **Source**: https://cmake.org/cmake/help/latest/command/find_package.html
> **Skill**: dev-suite skill `cmake` — see SKILL.md for the always-loaded quick reference.

## What this covers

The two completely different mechanisms hidden behind one command (`Find<Pkg>.cmake` modules vs `<Pkg>Config.cmake` config files), version comparison rules, COMPONENTS, the `<Pkg>_ROOT` search hint, and how `find_package` interacts with vcpkg/Conan toolchains.

## Deep dive

### MODULE vs CONFIG mode

```cmake
find_package(Foo 1.2 REQUIRED)              # Tries MODULE first, falls back to CONFIG
find_package(Foo 1.2 REQUIRED MODULE)       # Only MODULE: looks for FindFoo.cmake
find_package(Foo 1.2 REQUIRED CONFIG)       # Only CONFIG: looks for FooConfig.cmake
```

**MODULE mode**: CMake searches `CMAKE_MODULE_PATH` and its bundled modules for `FindFoo.cmake`. The file contains *imperative* code that probes the system (headers, libraries, pkg-config) and creates IMPORTED targets. Used for legacy libraries that don't ship CMake support (zlib, OpenGL, BLAS).

**CONFIG mode**: CMake searches well-known paths for `FooConfig.cmake` (or `foo-config.cmake`) installed by the library itself. The file is *generated* by CMake's `install(EXPORT ...)` and contains exact target definitions for the installed binaries.

**Prefer CONFIG** when available — it ships with the library and stays accurate; `Find*.cmake` modules drift.

### Search paths (CONFIG mode)

CMake looks in this order (selected highlights):

1. `<Pkg>_ROOT` variable / environment variable
2. `CMAKE_PREFIX_PATH`
3. `<Pkg>_DIR` cache variable
4. Per-platform standard prefixes (`/usr/local`, `/opt`, registry on Windows)

Inside each prefix, CMake looks for:

```
<prefix>/                      <Pkg>Config.cmake
<prefix>/(cmake|CMake)/        <Pkg>Config.cmake
<prefix>/lib/cmake/<pkg>/      <Pkg>Config.cmake
<prefix>/lib/<arch>/cmake/<pkg>/ <Pkg>Config.cmake
```

Set `<Pkg>_ROOT` for one-off overrides without polluting the global path.

### Version specification

```cmake
find_package(fmt 10.0          REQUIRED)   # >= 10.0, same major series
find_package(fmt 10.2.1        REQUIRED)   # >= 10.2.1, same major
find_package(fmt 10.0...11.0   REQUIRED)   # >= 10.0 AND < 11.0
find_package(fmt 10.0 EXACT    REQUIRED)   # exactly 10.0.x
```

The version comparison is performed by `<Pkg>ConfigVersion.cmake`. Authors specify the policy when generating it via `write_basic_package_version_file(... COMPATIBILITY ...)`:

| COMPATIBILITY | Behavior |
|---------------|----------|
| `AnyNewerVersion` | Found if installed >= requested |
| `SameMajorVersion` | Same major number |
| `SameMinorVersion` | Same major.minor |
| `ExactVersion` | Exact match |

### COMPONENTS

```cmake
find_package(Boost 1.84 REQUIRED COMPONENTS system filesystem)
target_link_libraries(myapp PRIVATE Boost::system Boost::filesystem)

find_package(Qt6 REQUIRED COMPONENTS Core Widgets Network OPTIONAL_COMPONENTS Qml)
```

Each component typically becomes its own IMPORTED target. `OPTIONAL_COMPONENTS` won't fail the configure if missing; check `<Pkg>_<Comp>_FOUND`.

### REQUIRED vs QUIET vs NO_*

```cmake
find_package(spdlog 1.13 QUIET CONFIG)
if(spdlog_FOUND)
    target_link_libraries(mylib PRIVATE spdlog::spdlog)
else()
    # Fall back to FetchContent
    include(FetchContent)
    FetchContent_Declare(spdlog GIT_REPOSITORY ... GIT_TAG v1.13.0)
    FetchContent_MakeAvailable(spdlog)
endif()
```

`QUIET` suppresses the "could not find" message during the optional probe. `REQUIRED` makes a missing package fatal.

### Interplay with vcpkg/Conan

Both vcpkg and Conan install packages with proper `<Pkg>Config.cmake` files. The toolchain file they provide tweaks `CMAKE_PREFIX_PATH` so that `find_package` finds them transparently:

```cmake
# Your CMakeLists.txt — same regardless of vcpkg/Conan
find_package(fmt CONFIG REQUIRED)
target_link_libraries(myapp PRIVATE fmt::fmt)
```

The user invokes:

```bash
# vcpkg
cmake -S . -B build -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake

# Conan 2 (after `conan install`)
cmake -S . -B build -DCMAKE_TOOLCHAIN_FILE=build/conan_toolchain.cmake
```

Your CMake code stays portable across package managers.

### Authoring a config file (consumer-facing)

```cmake
# In your library's CMakeLists.txt
include(CMakePackageConfigHelpers)

install(EXPORT mylibTargets
    FILE        mylibTargets.cmake
    NAMESPACE   mylib::
    DESTINATION lib/cmake/mylib
)

configure_package_config_file(
    cmake/mylibConfig.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/mylibConfig.cmake
    INSTALL_DESTINATION lib/cmake/mylib
)

write_basic_package_version_file(
    ${CMAKE_CURRENT_BINARY_DIR}/mylibConfigVersion.cmake
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion
)

install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/mylibConfig.cmake
    ${CMAKE_CURRENT_BINARY_DIR}/mylibConfigVersion.cmake
    DESTINATION lib/cmake/mylib
)
```

Now consumers do `find_package(mylib CONFIG REQUIRED)`.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Forgetting `CONFIG` and getting a stale bundled `Find<Pkg>.cmake` | Module mode picks up the wrong copy | Be explicit: `find_package(... CONFIG)` |
| Missing `REQUIRED` then later assuming the targets exist | Silent skip, errors at link time | Use `REQUIRED` or check `<Pkg>_FOUND` |
| `find_package(Boost)` linking the wrong Boost on a system with multiple installs | Search path resolution | Set `Boost_ROOT` or `BOOST_ROOT` |
| Hardcoding `find_package(... 1.2.3 EXACT)` | Brittle to patch updates | Use a range `1.2...2.0` |
| Calling `find_package` before `project()` | Compiler not yet probed | Always after `project()` |
| Linking against unnamespaced target (`spdlog`) | Bare name may not exist; use the namespaced form | Use `spdlog::spdlog` |

## See also

- https://cmake.org/cmake/help/latest/manual/cmake-packages.7.html — package authoring guide
- https://cmake.org/cmake/help/latest/module/CMakePackageConfigHelpers.html — helper module reference
- https://cmake.org/cmake/help/latest/manual/cmake-developer.7.html#find-modules — writing Find modules
