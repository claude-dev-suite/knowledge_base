# GoogleTest - Death Tests

> **Source**: https://google.github.io/googletest/advanced.html#death-tests
> **Skill**: dev-suite skill `testing/googletest` — see SKILL.md for the always-loaded quick reference.

## What this covers

What "death" actually means in GoogleTest, the two forking styles (`fast` vs `threadsafe`),
why most CI failures around death tests are caused by mismatched style and threading, and how
to write death-test regex that works on both POSIX and Windows.

## Deep dive

### What counts as a "death"

A death test passes if the statement causes the process to exit with non-zero status (or be killed
by a signal) AND the standard error output matches the regex. `std::exit(0)` does **not** count
as a death — the test fails. To assert on a particular exit code:

```cpp
EXPECT_EXIT(do_thing(),
            ::testing::ExitedWithCode(2),
            "configuration invalid");
```

`ExitedWithCode(N)` is portable; `KilledBySignal(SIGSEGV)` is POSIX-only.

### Style: `fast` vs `threadsafe`

```cpp
// Top of main, before InitGoogleTest:
::testing::FLAGS_gtest_death_test_style = "threadsafe";
```

| Style | How | When to use |
|-------|-----|-------------|
| `fast` (default) | `fork()` and run only the death statement; no re-exec | Single-threaded test process, fast |
| `threadsafe` | `fork()` then `execve()` re-runs the test binary with `--gtest_internal_run_death_test=...` | Any test that has spawned threads before the death test |

If you have *any* background thread (logger, prometheus exporter, gRPC server) running in the test
process, use `threadsafe`. The `fast` style copies thread state at fork() and re-acquired locks
will deadlock or behave nondeterministically.

### Regex matching — `RE2` syntax, but limited on Windows

Death-test regex uses RE2 on POSIX and a limited subset on Windows (no character classes, no
alternation in some versions). Use literal substrings whenever possible:

```cpp
// Portable
EXPECT_DEATH(deref(nullptr), "null pointer");

// Won't match on Windows — alternation
EXPECT_DEATH(panic(), "panic|abort");

// Use HasSubstr matcher via EXPECT_DEATH_IF_SUPPORTED + capture
```

For complex matching, capture stderr separately and use `EXPECT_THAT` matchers.

### `EXPECT_DEATH_IF_SUPPORTED`

Embedded targets and some sanitizer combinations don't support fork(). Use the
`_IF_SUPPORTED` variant to keep the build green on those platforms:

```cpp
EXPECT_DEATH_IF_SUPPORTED(crash_on_bad_input(""), "invalid input");
```

### What to test as a death

Good targets for death tests:

```cpp
TEST(ConfigDeathTest, AbortsOnMissingRequired) {
    EXPECT_DEATH(load_config("/nonexistent.toml"),
                 "missing required key");
}

TEST(InvariantDeathTest, AssertOnNegativeIndex) {
    EXPECT_DEATH({ int v[10]; (void)checked_at(v, -1); },
                 "index out of range");
}
```

Bad targets: testing exception throws (use `EXPECT_THROW`), testing regular error returns
(use `EXPECT_EQ`), or testing that a function logs a message (capture stderr instead).

### Naming convention

GoogleTest expects death-test suites to end in `DeathTest` so they can be ordered first
(reduces the chance of a thread being spawned before the death test by another suite):

```cpp
TEST(MyComponentDeathTest, AbortsOnNull) { ... }
```

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Death test deadlocks under `fast` style | Forked thread held a lock | Switch to `threadsafe`, or keep death tests single-threaded |
| Regex with `()` or `|` fails on Windows | Limited regex engine | Use literal substring; capture stderr for complex match |
| `ASSERT_DEATH` followed by code that needs the SUT alive | The current test process is fine — but you wrote the code wrong | Death test runs in a child; parent state is unaffected |
| Death tests on macOS with sanitizers report leaks | LSan inherits across fork | `LSAN_OPTIONS=detect_leaks=0` for death tests, or use `threadsafe` |
| Suite name doesn't end in `DeathTest` | Other tests run first, may spawn threads | Rename suite to `*DeathTest` |
| Asserting on `std::exit(0)` | Exit code 0 is not a "death" by GTest's definition | Use `EXPECT_EXIT(stmt, ExitedWithCode(0), regex)` explicitly |

## See also

- https://google.github.io/googletest/reference/assertions.html#death
- https://google.github.io/googletest/advanced.html#regular-expression-syntax
