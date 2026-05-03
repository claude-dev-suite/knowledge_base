# CMake - install() and Package Config Files

> **Source**: https://cmake.org/cmake/help/latest/command/install.html
> **Skill**: dev-suite skill `cmake` — see SKILL.md for the always-loaded quick reference.

## What this covers

The full surface of `install()` (TARGETS, EXPORT, FILES, DIRECTORY, RUNTIME_DEPENDENCIES, SCRIPT/CODE), the role of `GNUInstallDirs`, COMPONENT-based installs for partial packaging, and how to author a complete CMake package other projects can `find_package`.

## Deep dive

### Always include GNUInstallDirs

```cmake
include(GNUInstallDirs)
```

This sets cross-platform variables that respect the user's prefix and conventions:

| Variable | Linux default | Windows default |
|----------|---------------|-----------------|
| `CMAKE_INSTALL_BINDIR` | `bin` | `bin` |
| `CMAKE_INSTALL_LIBDIR` | `lib` or `lib64` | `lib` |
| `CMAKE_INSTALL_INCLUDEDIR` | `include` | `include` |
| `CMAKE_INSTALL_DATADIR` | `share` | `share` |
| `CMAKE_INSTALL_DOCDIR` | `share/doc/<project>` | `doc` |
| `CMAKE_INSTALL_SYSCONFDIR` | `etc` | `etc` |

Hardcoding `lib` breaks on multilib distributions (RHEL, openSUSE) that need `lib64`. Always use the variables.

### install(TARGETS ...) — the full form

```cmake
install(TARGETS mylib myapp myhdr
    EXPORT  myprojectTargets
    RUNTIME       DESTINATION ${CMAKE_INSTALL_BINDIR}        # .exe, .dll
                  COMPONENT runtime
    LIBRARY       DESTINATION ${CMAKE_INSTALL_LIBDIR}        # .so, dylib
                  COMPONENT runtime
    ARCHIVE       DESTINATION ${CMAKE_INSTALL_LIBDIR}        # .a, .lib
                  COMPONENT development
    PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}    # legacy; prefer FILE_SET
                  COMPONENT development
    FILE_SET HEADERS                                         # CMake 3.23+
                  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
                  COMPONENT development
    INCLUDES      DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}    # adds to INSTALL_INTERFACE
)
```

Each kind targets a different artifact type. `EXPORT` registers the targets in an export set for later `install(EXPORT ...)`.

### install(EXPORT ...) — generate consumer config

```cmake
install(EXPORT myprojectTargets
    FILE        myprojectTargets.cmake
    NAMESPACE   myproject::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/myproject
    COMPONENT   development
)

include(CMakePackageConfigHelpers)

write_basic_package_version_file(
    ${CMAKE_CURRENT_BINARY_DIR}/myprojectConfigVersion.cmake
    VERSION       ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion
    ARCH_INDEPENDENT                                  # if your lib is header-only
)

configure_package_config_file(
    cmake/myprojectConfig.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/myprojectConfig.cmake
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/myproject
)

install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/myprojectConfig.cmake
    ${CMAKE_CURRENT_BINARY_DIR}/myprojectConfigVersion.cmake
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/myproject
    COMPONENT   development
)
```

`myprojectConfig.cmake.in`:

```cmake
@PACKAGE_INIT@

include(CMakeFindDependencyMacro)
find_dependency(fmt 10 CONFIG)
find_dependency(spdlog 1.13 CONFIG)

include("${CMAKE_CURRENT_LIST_DIR}/myprojectTargets.cmake")

check_required_components(myproject)
```

Use `find_dependency` (not `find_package`) inside the config so version/QUIET propagate correctly.

### install(FILES) and install(DIRECTORY)

```cmake
install(FILES README.md LICENSE
    DESTINATION ${CMAKE_INSTALL_DOCDIR}
    COMPONENT   documentation
)

install(DIRECTORY include/
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    COMPONENT   development
    FILES_MATCHING PATTERN "*.hpp"
    PATTERN "internal" EXCLUDE
)

# Trailing slash matters: `include/` copies CONTENTS, `include` copies the dir itself
```

Prefer `FILE_SET HEADERS` over `install(DIRECTORY include/)` for new code — it tracks header dependencies and works with `install(TARGETS)` in one call.

### Runtime dependencies (CMake 3.21+)

```cmake
install(TARGETS myapp
    RUNTIME_DEPENDENCY_SET myapp_deps
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

install(RUNTIME_DEPENDENCY_SET myapp_deps
    PRE_EXCLUDE_REGEXES "api-ms-" "ext-ms-" "system32"
    POST_EXCLUDE_REGEXES ".*system32/.*\\.dll"
    DIRECTORIES $<TARGET_FILE_DIR:third_party::lib>
)
```

Bundles all transitive DLL/.so dependencies of `myapp` into the install prefix. Replaces `BundleUtilities` and `fixup_bundle`.

### COMPONENTS for split installs

```bash
cmake --install build --component runtime --prefix /usr/local
cmake --install build --component development --prefix /usr/local
cmake --install build --component documentation --prefix /usr/share/doc/myproject
```

Distros use this to split into `myproject`, `myproject-dev`, `myproject-doc` packages.

### install(SCRIPT) and install(CODE)

```cmake
install(CODE "message(STATUS \"Installing to ${CMAKE_INSTALL_PREFIX}\")")
install(SCRIPT cmake/post_install.cmake)
```

Run arbitrary CMake at install time. Use sparingly — usually a sign you're working around the system.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Hardcoding `DESTINATION lib` | Breaks on lib64 distros | Use `${CMAKE_INSTALL_LIBDIR}` from GNUInstallDirs |
| Forgetting `INCLUDES DESTINATION` | Installed targets have no include dir | Add `INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}` |
| Calling `find_package` (not `find_dependency`) in config | Loses QUIET / REQUIRED context | Use `find_dependency` in `<Pkg>Config.cmake` |
| `install(DIRECTORY include)` (no trailing slash) | Installs `include/include/...` | Add trailing slash to install contents |
| Public headers exposed via PUBLIC include but not installed | Consumers fail at build time | Install the headers via `FILE_SET HEADERS` or `install(DIRECTORY)` |
| Missing `NAMESPACE myproject::` | Consumers can't tell `myproject::lib` apart | Always namespace exported targets |

## See also

- https://cmake.org/cmake/help/latest/module/CMakePackageConfigHelpers.html — config-file helpers
- https://cmake.org/cmake/help/latest/module/GNUInstallDirs.html — standard directories
- https://cmake.org/cmake/help/latest/manual/cmake-packages.7.html — full package authoring guide
