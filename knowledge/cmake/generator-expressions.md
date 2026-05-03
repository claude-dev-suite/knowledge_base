# CMake - Generator Expressions

> **Source**: https://cmake.org/cmake/help/latest/manual/cmake-generator-expressions.7.html
> **Skill**: dev-suite skill `cmake` — see SKILL.md for the always-loaded quick reference.

## What this covers

When generator expressions are evaluated (during build-system *generation*, not configure), the four major categories (logical, informational, output, debug), the BUILD_INTERFACE / INSTALL_INTERFACE pattern, and how to debug them with `file(GENERATE)`.

## Deep dive

### When are they evaluated

Most CMake commands evaluate immediately at configure time. Generator expressions delay evaluation until *generation* — the brief phase between configure and build, when CMake writes the actual buildsystem. This matters because:

- Multi-config generators (Visual Studio, Xcode, Ninja Multi-Config) need *one* CMakeLists to produce *multiple* configurations. `$<CONFIG:Debug>` lets the same line emit different flags per config.
- Per-target queries like `$<TARGET_PROPERTY:tgt,COMPILE_OPTIONS>` aren't fully resolvable until all targets are known.

```cmake
# This is wrong — message() runs at configure time, sees the literal string
message(STATUS "Build type: $<CONFIG>")   # prints "Build type: $<CONFIG>"

# This works — file(GENERATE) runs at generate time, expands properly
file(GENERATE OUTPUT ${CMAKE_BINARY_DIR}/info.txt
    CONTENT "Build type: $<CONFIG>\n")
```

### Syntax

`$<...>` is the wrapper. Three forms:

- `$<KEYWORD>` — query, returns value (`$<CONFIG>`, `$<PLATFORM_ID>`)
- `$<KEYWORD:argument>` — operates on argument (`$<UPPER_CASE:hello>` -> `HELLO`)
- `$<condition:value>` — emits `value` when condition is true, empty otherwise

### Logical operators

```cmake
$<NOT:cond>
$<AND:cond1,cond2,...>
$<OR:cond1,cond2,...>
$<IF:cond,if_true,if_false>
$<STREQUAL:str1,str2>
$<EQUAL:n1,n2>
$<IN_LIST:item,list>
$<VERSION_LESS:v1,v2>      $<VERSION_GREATER:v1,v2>
$<TARGET_EXISTS:tgt>
$<COMPILE_LANGUAGE:CXX>    $<COMPILE_LANGUAGE:CUDA>
$<COMPILE_LANG_AND_ID:CXX,Clang,GNU>
$<CONFIG:Debug,RelWithDebInfo>
$<CXX_COMPILER_ID:MSVC>
$<PLATFORM_ID:Linux,Darwin>
```

### Realistic examples

```cmake
target_compile_options(mylib PRIVATE
    # Strict warnings only when not building tests
    $<$<NOT:$<BOOL:${BUILD_TESTING}>>:-Werror>

    # Different optimization per config
    $<$<CONFIG:Debug>:-O0 -g3 -fno-omit-frame-pointer>
    $<$<CONFIG:Release>:-O3 -DNDEBUG>
    $<$<CONFIG:RelWithDebInfo>:-O2 -g>

    # Cross-compiler warning sets
    $<$<CXX_COMPILER_ID:MSVC>:/W4 /permissive- /Zc:__cplusplus>
    $<$<CXX_COMPILER_ID:GNU,Clang>:-Wall -Wextra -Wpedantic -Wshadow>

    # Suppress a warning only on Clang 16+
    $<$<AND:$<CXX_COMPILER_ID:Clang>,$<VERSION_GREATER_EQUAL:$<CXX_COMPILER_VERSION>,16>>:-Wno-deprecated-declarations>

    # Per-language: only for CUDA sources
    $<$<COMPILE_LANGUAGE:CUDA>:--expt-relaxed-constexpr>
)
```

### BUILD_INTERFACE vs INSTALL_INTERFACE

The single most useful pattern in modern CMake:

```cmake
target_include_directories(mylib PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)
```

When the target is consumed *from the build tree* (via `add_subdirectory` or `FetchContent`), the include path is the source directory. When consumed *from an installed package* (via `find_package`), the include path is `${CMAKE_INSTALL_PREFIX}/include`. One declaration handles both.

### Useful informational expressions

| Expression | Returns |
|------------|---------|
| `$<TARGET_FILE:tgt>` | Full path to the built artifact (executable / library) |
| `$<TARGET_FILE_DIR:tgt>` | Directory of the artifact |
| `$<TARGET_NAME_IF_EXISTS:tgt>` | Target name if it exists, empty otherwise |
| `$<TARGET_PROPERTY:tgt,prop>` | Read a property |
| `$<TARGET_GENEX_EVAL:tgt,expr>` | Evaluate `expr` in the context of `tgt` |
| `$<INSTALL_PREFIX>` | Install prefix (only valid in install rules / generated files) |

### Use case: copy a DLL next to an exe at build time

```cmake
add_custom_command(TARGET myapp POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
        $<TARGET_FILE:third_party::dll>
        $<TARGET_FILE_DIR:myapp>
    COMMAND_EXPAND_LISTS
)
```

`COMMAND_EXPAND_LISTS` ensures list-valued expressions become space-separated arguments.

### Debugging generator expressions

`message()` won't print the expanded form. Use `file(GENERATE)`:

```cmake
file(GENERATE OUTPUT ${CMAKE_BINARY_DIR}/debug-genex-$<CONFIG>.txt
    CONTENT "options for mylib: $<TARGET_PROPERTY:mylib,COMPILE_OPTIONS>\n"
)
```

Inspect the generated file per config.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Using genex inside `message()` or `set()` | They run at configure, before generation | Use `file(GENERATE)` or `add_custom_command` |
| Quoting genex with embedded commas | Comma is the separator | Wrap as a list element or use `$<COMMA>` |
| `$<CONFIG>` empty for single-config generators if `CMAKE_BUILD_TYPE` unset | No config selected | Set a default config in the preset |
| Genex in `install(FILES ...)` referencing per-config artifacts | Multi-config installs need `--config` flag | Use `install(TARGETS ... CONFIGURATIONS ...)` |
| Comparing strings with `$<EQUAL>` | EQUAL is numeric only | Use `$<STREQUAL:a,b>` |
| Forgetting `BUILD_INTERFACE` causes installed paths to leak the source tree | Includes refer to your dev directory | Always wrap target_include_directories paths |

## See also

- https://cmake.org/cmake/help/latest/manual/cmake-generator-expressions.7.html — full reference
- https://cmake.org/cmake/help/latest/command/file.html#generate — file(GENERATE) for debugging
- https://cmake.org/cmake/help/latest/prop_tgt/INTERFACE_INCLUDE_DIRECTORIES.html — usage in install
