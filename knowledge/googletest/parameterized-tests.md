# GoogleTest - Value-Parameterized Tests

> **Source**: https://google.github.io/googletest/advanced.html#value-parameterized-tests
> **Skill**: dev-suite skill `testing/googletest` — see SKILL.md for the always-loaded quick reference.

## What this covers

Building parameterized tests beyond the `Values(1, 2, 3)` cheat-sheet form: custom value
generators, `Combine` for cartesian products, struct parameters with custom name printers, and
keeping diagnostics readable when a parameterized test fails.

## Deep dive

### Custom name printers — turn `IsPrime/0` into `IsPrime/2`

```cpp
class IsPrimeTest : public ::testing::TestWithParam<int> {};

TEST_P(IsPrimeTest, RecognizesPrimes) {
    EXPECT_TRUE(is_prime(GetParam())) << "for " << GetParam();
}

INSTANTIATE_TEST_SUITE_P(
    SmallPrimes,
    IsPrimeTest,
    ::testing::Values(2, 3, 5, 7, 11, 13),
    [](const ::testing::TestParamInfo<int>& info) {
        return "n_" + std::to_string(info.param);
    }
);
// Test names: SmallPrimes/IsPrimeTest.RecognizesPrimes/n_2, .../n_3, ...
```

Name characters must match `[A-Za-z0-9_]`; signs and dots will fail registration.

### Struct parameters — readable, type-safe

```cpp
struct ParseCase {
    std::string input;
    int         expected;
    bool        should_throw;
};

class ParseTest : public ::testing::TestWithParam<ParseCase> {};

TEST_P(ParseTest, MatchesExpectation) {
    const auto& c = GetParam();
    if (c.should_throw) {
        EXPECT_THROW(parse(c.input), std::invalid_argument);
    } else {
        EXPECT_EQ(parse(c.input), c.expected);
    }
}

INSTANTIATE_TEST_SUITE_P(
    Cases, ParseTest,
    ::testing::Values(
        ParseCase{"42",   42,  false},
        ParseCase{"-7",   -7,  false},
        ParseCase{"abc",  0,   true},
        ParseCase{"",     0,   true}
    ),
    [](const auto& info) {
        return info.param.input.empty() ? "empty" : info.param.input;
    }
);
```

### Generators

| Generator | Produces |
|-----------|----------|
| `Values(v1, v2, ...)` | Each value once |
| `ValuesIn(container)` / `ValuesIn(begin, end)` | Each element |
| `Range(begin, end[, step])` | Half-open arithmetic sequence |
| `Bool()` | `true, false` |
| `Combine(g1, g2, ...)` | Cartesian product as `std::tuple` |
| Custom: derive from `::testing::internal::ParamGeneratorInterface<T>` | Anything |

### `Combine` for matrix tests

```cpp
class CodecTest : public ::testing::TestWithParam<std::tuple<Codec, int>> {};

TEST_P(CodecTest, RoundTrips) {
    auto [codec, size] = GetParam();
    auto data = random_bytes(size);
    EXPECT_EQ(decode(codec, encode(codec, data)), data);
}

INSTANTIATE_TEST_SUITE_P(
    Matrix, CodecTest,
    ::testing::Combine(
        ::testing::Values(Codec::Zstd, Codec::Lz4, Codec::Deflate),
        ::testing::Values(0, 1, 1024, 1'048'576)
    )
);
// 3 codecs × 4 sizes = 12 tests
```

### Generated parameters at runtime

`GetParam()` is evaluated once per test instance, but the *generator* runs at
`RUN_ALL_TESTS()` time. If you need a parameter list determined by reading a file or env var,
populate a `std::vector` first and use `ValuesIn(vec)`:

```cpp
const auto kFixturePaths = []{
    std::vector<std::string> v;
    for (auto& p : std::filesystem::directory_iterator("fixtures/"))
        v.push_back(p.path().string());
    return v;
}();
INSTANTIATE_TEST_SUITE_P(All, FixtureTest, ::testing::ValuesIn(kFixturePaths));
```

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Special characters in parameter names | Registration fails silently | Sanitize: replace `.`, `-`, `/` with `_` in printer |
| `INSTANTIATE_TEST_SUITE_P` in a header | ODR violation across TUs | Always in a single `.cpp` |
| Empty value list | Test treated as "not instantiated" → CI passes vacuously | Add `GTEST_ALLOW_UNINSTANTIATED_PARAMETERIZED_TEST(SuiteName)` only intentionally |
| Forgetting `GTEST_ALLOW_...` for runtime-empty generators | Build error after upgrade | Mark the suite or pre-check the list |
| `Combine` exploding to thousands of cases | Slow CI | Tier with `Combine(small_set, full_set)` and gate full set on nightly |

## See also

- https://google.github.io/googletest/advanced.html#how-to-write-value-parameterized-tests
- https://google.github.io/googletest/reference/testing.html#param-generators
