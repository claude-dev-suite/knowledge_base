# GoogleTest - CMake Integration & CTest

> **Source**: https://google.github.io/googletest/quickstart-cmake.html
> **Skill**: dev-suite skill `testing/googletest` — see SKILL.md for the always-loaded quick reference.

## What this covers

The two ways to bring GoogleTest into a CMake project (`FetchContent` vs `find_package`),
the difference between `gtest_add_tests` (parse-time) and `gtest_discover_tests` (post-build,
recommended), and CTest configuration that makes failures actionable in CI.

## Deep dive

### `FetchContent` — pinned, hermetic

```cmake
include(FetchContent)
FetchContent_Declare(googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG        v1.15.2
    GIT_SHALLOW    TRUE
    OVERRIDE_FIND_PACKAGE   # CMake 3.24+: lets find_package(GTest) succeed
)
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)   # MSVC: link with /MD not /MT
FetchContent_MakeAvailable(googletest)
```

Pin to a *tag*, not `main`. `OVERRIDE_FIND_PACKAGE` lets transitive dependencies that call
`find_package(GTest)` resolve to the fetched version, avoiding ABI mismatches.

### `find_package` — system-installed (or vcpkg/Conan)

```cmake
find_package(GTest CONFIG REQUIRED)
target_link_libraries(unit_tests PRIVATE GTest::gtest GTest::gtest_main GTest::gmock)
```

Use `CONFIG` mode (not the legacy `MODULE`) — exposes the modern imported targets and propagates
flags correctly. Targets:

- `GTest::gtest` — the framework, no `main()`
- `GTest::gtest_main` — provides `int main(int, char**)`
- `GTest::gmock` / `GTest::gmock_main` — same, for gmock

Don't link both `gtest_main` and `gmock_main` — pick one (`gmock_main` calls `InitGoogleMock`
which also initializes gtest).

### `gtest_discover_tests` (recommended) vs `gtest_add_tests`

```cmake
include(GoogleTest)
add_executable(unit_tests test_a.cpp test_b.cpp)
target_link_libraries(unit_tests PRIVATE mylib GTest::gmock_main)

gtest_discover_tests(unit_tests
    DISCOVERY_MODE PRE_TEST          # CMake 3.18+: defer discovery until ctest runs
    PROPERTIES
        TIMEOUT 60
        LABELS "unit"
    EXTRA_ARGS --gtest_color=yes
)
```

`gtest_discover_tests` runs the test binary at build/test time with `--gtest_list_tests` to
enumerate tests — handles `INSTANTIATE_TEST_SUITE_P` and runtime-determined parameter lists
correctly. `gtest_add_tests` parses C++ source at configure time and misses anything dynamic.

`DISCOVERY_MODE PRE_TEST` (the modern default) defers the enumeration to `ctest` time so
cross-compiled binaries that can't run on the build host still configure cleanly.

### CTest invocation patterns

```bash
# Parallel, fail-fast off (run all to surface every failure)
ctest --test-dir build -j$(nproc) --output-on-failure

# Filter by name, regex
ctest -R '^UserTest\.' -V

# Filter by label
ctest -L unit

# Re-run only failures
ctest --rerun-failed --output-on-failure

# JUnit XML for CI consumption
ctest --output-junit results.xml
```

For per-test JUnit output (one file per test binary, GTest's own format), pass
`--gtest_output=xml:results-` to the binary; CMake can wire this:

```cmake
gtest_discover_tests(unit_tests
    EXTRA_ARGS --gtest_output=xml:${CMAKE_BINARY_DIR}/test-results/unit_tests.xml
)
```

### Sanitizer-instrumented test runs

```cmake
option(ENABLE_ASAN "" OFF)
if(ENABLE_ASAN AND NOT MSVC)
    target_compile_options(unit_tests PRIVATE -fsanitize=address -fno-omit-frame-pointer -g)
    target_link_options(unit_tests PRIVATE -fsanitize=address)
endif()

# At test discovery, propagate ASAN_OPTIONS so leaks are reported:
gtest_discover_tests(unit_tests
    PROPERTIES ENVIRONMENT "ASAN_OPTIONS=detect_leaks=1:abort_on_error=1"
)
```

### Multi-config generators (Visual Studio, Xcode)

`gtest_discover_tests` injects per-config logic automatically. Run with:

```bash
ctest -C Debug --output-on-failure
```

Without `-C`, multi-config builds report no tests found.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| MSVC ABI mismatch (mixed `/MT` / `/MD`) | LNK2038 errors | `set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)` before fetch |
| `gtest_add_tests` doesn't see parameterized cases | Parses source statically | Switch to `gtest_discover_tests` |
| Discovery runs at configure on a cross-compile | Binary can't execute on build host | `DISCOVERY_MODE PRE_TEST` (CMake 3.18+) |
| Missing tests under multi-config IDE | `ctest -C <config>` not passed | Pass `-C Debug` (or set `CMAKE_CONFIGURATION_TYPES`) |
| Linking both `gtest_main` and `gmock_main` | Duplicate `main` symbol | Pick one — `gmock_main` for gmock projects |
| `FetchContent` pulling `main` branch | Builds break on upstream change | Pin a `GIT_TAG` to a release |

## See also

- https://google.github.io/googletest/quickstart-cmake.html
- https://cmake.org/cmake/help/latest/module/GoogleTest.html
