# CMake - Modern Targets and Visibility

> **Source**: https://cmake.org/cmake/help/latest/manual/cmake-buildsystem.7.html
> **Skill**: dev-suite skill `cmake` — see SKILL.md for the always-loaded quick reference.

## What this covers

How target properties propagate via PUBLIC/PRIVATE/INTERFACE, the difference between *build requirements* and *usage requirements*, ALIAS targets for namespacing, IMPORTED targets for external libs, and INTERFACE libraries as configuration carriers.

## Deep dive

### Two parallel property sets per target

Each target carries two sets of properties:

| Property set | Used by | Set via |
|--------------|---------|---------|
| `<PROP>` (build requirements) | The target itself when compiling/linking | PRIVATE / PUBLIC keyword |
| `INTERFACE_<PROP>` (usage requirements) | Targets that link to this one | INTERFACE / PUBLIC keyword |

The keywords map as:

- `PRIVATE` → set `<PROP>` only
- `INTERFACE` → set `INTERFACE_<PROP>` only
- `PUBLIC` → set both

Example for include directories:

```cmake
target_include_directories(mylib
    PRIVATE   src                  # mylib.cpp can include from src/, consumers cannot
    PUBLIC    include              # both mylib.cpp and consumers can include from include/
    INTERFACE extra_consumer_only  # consumers can include, mylib itself cannot
)
```

### When to use which keyword

| You're declaring... | Keyword | Example |
|---------------------|---------|---------|
| A header included only inside `.cpp` | PRIVATE | `target_link_libraries(mylib PRIVATE spdlog::spdlog)` |
| A header `#include`d from your public headers | PUBLIC | `target_link_libraries(mylib PUBLIC fmt::fmt)` if your `.hpp` does `#include <fmt/format.h>` |
| A header-only utility consumers need | INTERFACE | header-only target only |

The mental check: "Will my consumers need this transitively?" If yes, PUBLIC. If your headers don't expose the type, PRIVATE.

### Transitive propagation

```cmake
add_library(core STATIC ...)
target_link_libraries(core PUBLIC fmt::fmt)        # core's headers use fmt
target_link_libraries(core PRIVATE spdlog::spdlog) # core's .cpp uses spdlog

add_library(net STATIC ...)
target_link_libraries(net PUBLIC core)             # net headers use core types

add_executable(server main.cpp)
target_link_libraries(server PRIVATE net)
# server transitively gets: net + core + fmt
# server does NOT get spdlog (it was PRIVATE to core)
```

This is the "transitive interface" — the entire reason modern CMake works.

### ALIAS targets for namespacing

```cmake
add_library(mylib STATIC src/foo.cpp)
add_library(MyProject::mylib ALIAS mylib)
```

Now consumers always write `target_link_libraries(app PRIVATE MyProject::mylib)`. The `Project::name` form fails fast at configure time if the target is missing, while a typo in a bare name silently becomes a "-lname" linker flag.

### IMPORTED targets

External libraries you don't build appear as IMPORTED targets:

```cmake
add_library(zstd::libzstd_static STATIC IMPORTED)
set_target_properties(zstd::libzstd_static PROPERTIES
    IMPORTED_LOCATION /opt/zstd/lib/libzstd.a
    INTERFACE_INCLUDE_DIRECTORIES /opt/zstd/include
)
```

In practice, `find_package(zstd CONFIG)` creates these for you. You only write `add_library(... IMPORTED)` when wrapping a system library by hand.

### INTERFACE libraries as configuration bundles

```cmake
add_library(project_warnings INTERFACE)
target_compile_options(project_warnings INTERFACE
    $<$<CXX_COMPILER_ID:GNU,Clang>:-Wall -Wextra -Wpedantic -Wshadow -Wconversion>
    $<$<CXX_COMPILER_ID:MSVC>:/W4 /permissive->
)

add_library(project_options INTERFACE)
target_compile_features(project_options INTERFACE cxx_std_20)
target_compile_definitions(project_options INTERFACE
    $<$<CONFIG:Debug>:MYAPP_DEBUG=1>
)

# Apply to every target in the project
target_link_libraries(mylib  PRIVATE project_warnings project_options)
target_link_libraries(myexe  PRIVATE project_warnings project_options)
```

This pattern centralizes warning levels and feature flags in one place.

### target_sources with FILE_SET (CMake 3.23+)

```cmake
add_library(mylib)
target_sources(mylib
    PRIVATE src/widget.cpp src/internal.cpp
    PUBLIC FILE_SET HEADERS
        BASE_DIRS include
        FILES include/mylib/widget.hpp include/mylib/api.hpp
)
```

`FILE_SET HEADERS` declares headers that get installed automatically by `install(TARGETS mylib FILE_SET HEADERS)` — no separate `install(DIRECTORY include/)` needed.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| `target_link_libraries(mylib spdlog)` (no keyword) | Legacy form, conflicts in mixed projects | Always specify PRIVATE/PUBLIC/INTERFACE |
| Marking a dep PUBLIC "to be safe" | Forces all consumers to find that dep | Use PRIVATE unless your headers expose the type |
| Adding `target_include_directories(... PRIVATE ${PROJECT_SOURCE_DIR}/include)` | Bypasses the public/private distinction | Use the `include/` dir as PUBLIC; private includes go in `src/` |
| `target_link_libraries(mylib PRIVATE -lpthread)` | Raw flag bypasses transitivity | Use `find_package(Threads); target_link_libraries(mylib PRIVATE Threads::Threads)` |
| ALIAS target without `::` namespace | Defeats fail-fast checking | Always namespace: `MyProject::mylib` |
| Setting `CMAKE_CXX_FLAGS` instead of `target_compile_options` | Leaks to every target globally | Use per-target options, gathered via INTERFACE library if needed |

## See also

- https://cmake.org/cmake/help/latest/manual/cmake-buildsystem.7.html#build-specification-and-usage-requirements — formal definitions
- https://cmake.org/cmake/help/latest/command/target_link_libraries.html — full keyword semantics
- https://cmake.org/cmake/help/latest/prop_tgt/INTERFACE_LINK_LIBRARIES.html — interface link properties
