# GoogleTest - Full Assertion Catalog & Predicates

> **Source**: https://google.github.io/googletest/reference/assertions.html
> **Skill**: dev-suite skill `testing/googletest` — see SKILL.md for the always-loaded quick reference.

## What this covers

The full assertion surface beyond the cheat-sheet: predicate assertions, `AssertionResult`,
explicit success/failure, custom diagnostics, and the often-overlooked `EXPECT_PRED_FORMAT*` family
that produces dramatically better failure messages than naive `EXPECT_TRUE(predicate(...))`.

## Deep dive

### `EXPECT_TRUE(pred(a, b))` vs `EXPECT_PRED2(pred, a, b)`

```cpp
bool is_subset(const std::set<int>& a, const std::set<int>& b);

// Diagnostic on failure:
//   Value of: is_subset(a, b)
//   Actual:   false
EXPECT_TRUE(is_subset(a, b));

// Diagnostic on failure:
//   is_subset(a, b) evaluates to false, where
//   a evaluates to { 1, 2, 9 }
//   b evaluates to { 1, 2, 3 }
EXPECT_PRED2(is_subset, a, b);
```

For the highest-quality diagnostics write a *predicate-formatter* that returns
`::testing::AssertionResult`:

```cpp
::testing::AssertionResult IsSubsetOf(const char* a_expr,
                                      const char* b_expr,
                                      const std::set<int>& a,
                                      const std::set<int>& b) {
    std::set<int> diff;
    std::set_difference(a.begin(), a.end(), b.begin(), b.end(),
                        std::inserter(diff, diff.end()));
    if (diff.empty()) return ::testing::AssertionSuccess();
    return ::testing::AssertionFailure()
        << a_expr << " is not a subset of " << b_expr
        << "; missing elements: " << ::testing::PrintToString(diff);
}

EXPECT_PRED_FORMAT2(IsSubsetOf, a, b);
```

### Floating-point assertions — what `EXPECT_DOUBLE_EQ` actually does

It compares within **4 ULPs** (units in the last place), not a fixed epsilon. This is the right
default for *computational equality* but is wrong if you need a relative tolerance:

```cpp
EXPECT_DOUBLE_EQ(result, expected);          // 4 ULP comparison
EXPECT_NEAR(result, expected, 1e-9);         // absolute tolerance
EXPECT_THAT(result, ::testing::DoubleNear(expected, 1e-9));
```

Comparing a NaN with anything (including itself) is always false — `EXPECT_DOUBLE_EQ(nan, nan)` *fails*.
Use `EXPECT_TRUE(std::isnan(x))` instead.

### Exception assertions

```cpp
EXPECT_THROW(parse(""), std::invalid_argument);

// Inspect the exception
try {
    parse("");
    FAIL() << "Expected std::invalid_argument";
} catch (const std::invalid_argument& e) {
    EXPECT_THAT(e.what(), ::testing::HasSubstr("empty input"));
} catch (...) {
    FAIL() << "Wrong exception type";
}
```

`EXPECT_THROW` cannot inspect `what()` — fall back to manual try/catch when the message matters.

### Generating failures explicitly

```cpp
TEST(StateMachine, ValidTransitions) {
    switch (current) {
    case State::Idle:    /* OK */ break;
    case State::Running: /* OK */ break;
    default: FAIL() << "Unexpected state: " << static_cast<int>(current);
    }
}

ADD_FAILURE() << "Non-fatal — keep running this test";
SUCCEED();   // marks an explicit pass for clarity (rarely needed)
```

`FAIL()` is `ASSERT_*`-class (returns); `ADD_FAILURE()` is `EXPECT_*`-class (continues).

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| `EXPECT_EQ(0u, container.size())` | Mixed signedness warning under `-Wsign-compare` | `EXPECT_EQ(container.size(), 0u)` (actual second) — or use `IsEmpty()` matcher |
| `EXPECT_DOUBLE_EQ(a/b, expected)` for tiny values | 4-ULP test fails near zero | `EXPECT_NEAR` with absolute epsilon |
| `EXPECT_THROW(stmt, std::exception)` | Catches anything derived → too loose | Catch the most-derived expected type |
| Using `<<` with `ASSERT_*` then `return` | Stream message lost if `<<` throws | Keep stream payload simple, no SUT calls |
| `EXPECT_TRUE(v.empty())` | "Actual: false" — useless | `EXPECT_THAT(v, IsEmpty())` for "{1, 2, 3}" diagnostic |

## See also

- https://google.github.io/googletest/reference/matchers.html
- https://google.github.io/googletest/advanced.html#more-assertions
