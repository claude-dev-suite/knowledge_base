# GoogleTest - Basics & Test Discovery

> **Source**: https://google.github.io/googletest/primer.html
> **Skill**: dev-suite skill `testing/googletest` — see SKILL.md for the always-loaded quick reference.

## What this covers

How GoogleTest names, registers, and discovers tests at runtime; the lifecycle of a single test case;
the precise difference between `EXPECT_*` and `ASSERT_*` and when each one belongs in real code.

## Deep dive

### Registration and discovery

`TEST(Suite, Name)` expands to a class derived from `::testing::Test` whose constructor registers
the test with the global `UnitTest` singleton via static initialization. Linking a translation unit
that defines `TEST(...)` is enough — no central manifest. This is also why placing a `TEST` inside
a static library that nothing references will cause the linker to drop it; always link tests into
the executable directly, or use `--whole-archive` / `/WHOLEARCHIVE` for static-lib test bundles.

```cmake
# Wrong: tests in a static library may be stripped
add_library(my_tests STATIC test_a.cpp test_b.cpp)
target_link_libraries(runner PRIVATE my_tests)

# Right
add_executable(runner test_a.cpp test_b.cpp main.cpp)
```

### EXPECT vs ASSERT — semantic, not stylistic

`ASSERT_*` calls `return` on failure. That means:

- They **leak** in the SUT side (RAII still cleans up locals, but loops/state they were driving stop).
- They **cannot be used inside non-void helpers** that return a value other than `void`.
- They **must be used** when continuing would dereference a null/invalid object:

```cpp
TEST(UserService, FindReturnsValidObject) {
    auto user = service.find(42);
    ASSERT_NE(user, nullptr);     // continuing would crash the test process
    EXPECT_EQ(user->name(), "alice");
    EXPECT_EQ(user->age(), 30);
}
```

### Helper functions and `SCOPED_TRACE`

If you factor assertions into a helper, use `void` return and add `SCOPED_TRACE` so failures point
back to the call site:

```cpp
void ExpectValidUser(const User& u, const char* where) {
    SCOPED_TRACE(where);
    EXPECT_FALSE(u.name().empty());
    EXPECT_GE(u.age(), 0);
}

TEST(UserTest, AllUsersValid) {
    for (const auto& u : load_users())
        ExpectValidUser(u, u.id().c_str());
}
```

### Test ordering — don't depend on it

GoogleTest runs tests in registration order *within a suite* but suite order is unspecified.
Use `--gtest_shuffle` in CI to flush out hidden ordering dependencies. Combine with
`--gtest_repeat=N` to amplify flakes.

### The `--gtest_filter` mini-language

```bash
./tests --gtest_filter='UserTest.*-UserTest.Slow*'   # all UserTest except Slow*
./tests --gtest_filter='*Db*:*Net*'                  # union: matches Db OR Net
```

Filter is `positive[:-negative]`, with `:` for OR and `*`/`?` glob.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Using `ASSERT_*` in a helper that returns `int` | Compiler error: cannot return `void` | Make the helper return `void` and use `SCOPED_TRACE` |
| Tests in a STATIC lib silently disappear | Linker drops unreferenced symbols | Build tests into the executable directly |
| `EXPECT_EQ` then dereferencing the value | Continues after a null check fails → segfault | Use `ASSERT_NE(p, nullptr)` first |
| Suite name with `_` (e.g. `My_Suite`) | Reserved for type-parameterized internals | Stick to `MySuite`/`MyFooTest` |
| Same `(Suite, Name)` in two files | ODR violation, behavior undefined | Each TEST must be globally unique |

## See also

- https://google.github.io/googletest/faq.html
- https://google.github.io/googletest/advanced.html
