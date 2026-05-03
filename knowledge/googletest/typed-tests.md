# GoogleTest - Typed and Type-Parameterized Tests

> **Source**: https://google.github.io/googletest/advanced.html#typed-tests
> **Skill**: dev-suite skill `testing/googletest` — see SKILL.md for the always-loaded quick reference.

## What this covers

The two related but distinct mechanisms for sweeping the same test logic over a list of types:
`TYPED_TEST` (closed list, single TU) and `TYPED_TEST_P` (open list, registered per-translation-unit
and instantiated separately) — useful for shipping a test suite as a header that downstream library
authors can instantiate over their own types.

## Deep dive

### TYPED_TEST — closed type list

```cpp
template <typename Container>
class ContainerTest : public ::testing::Test {
protected:
    Container c_;
};

using ContainerTypes = ::testing::Types<
    std::vector<int>,
    std::deque<int>,
    std::list<int>>;

TYPED_TEST_SUITE(ContainerTest, ContainerTypes);

TYPED_TEST(ContainerTest, StartsEmpty) {
    EXPECT_TRUE(this->c_.empty());
    EXPECT_EQ(this->c_.size(), 0u);
}

TYPED_TEST(ContainerTest, GrowsAfterInsert) {
    this->c_.insert(this->c_.end(), 42);
    EXPECT_EQ(this->c_.size(), 1u);
}
```

Important syntax notes:
- Inside `TYPED_TEST`, members of the fixture must be accessed via `this->` because the base
  class is dependent on the template parameter.
- `TypeParam` is the alias for the current type; `TestFixture` for the fixture instantiation.

### TYPED_TEST_P — open registration, separate instantiation

Use this when the type list lives in a different translation unit than the test logic — typical
when you ship a "concept conformance" test suite as a library:

```cpp
// File: serialization_conformance.hpp  (shipped header)
template <typename Codec>
class CodecConformance : public ::testing::Test {};

TYPED_TEST_SUITE_P(CodecConformance);

TYPED_TEST_P(CodecConformance, RoundTripsEmptyMessage) {
    using Codec = TypeParam;
    Message empty;
    auto encoded = Codec::encode(empty);
    auto decoded = Codec::decode(encoded);
    EXPECT_EQ(decoded, empty);
}

TYPED_TEST_P(CodecConformance, ProducesNonEmptyOutput) {
    using Codec = TypeParam;
    EXPECT_FALSE(Codec::encode(Message{42}).empty());
}

REGISTER_TYPED_TEST_SUITE_P(CodecConformance,
    RoundTripsEmptyMessage,
    ProducesNonEmptyOutput);
```

```cpp
// File: my_codec_test.cpp  (downstream user)
#include "serialization_conformance.hpp"
#include "json_codec.hpp"
#include "msgpack_codec.hpp"

using MyCodecs = ::testing::Types<JsonCodec, MsgPackCodec>;
INSTANTIATE_TYPED_TEST_SUITE_P(MyImpls, CodecConformance, MyCodecs);
```

If you forget any name in `REGISTER_TYPED_TEST_SUITE_P`, that test silently never runs.
Conversely, registering a name without a matching `TYPED_TEST_P` is a compile-time error.

### Combining typed tests with `if constexpr`

Sometimes one of the types in the list lacks a feature the others have:

```cpp
TYPED_TEST(ContainerTest, RandomAccessOperations) {
    if constexpr (requires(TypeParam c) { c[0]; }) {
        this->c_.insert(this->c_.end(), {1, 2, 3});
        EXPECT_EQ(this->c_[1], 2);
    } else {
        GTEST_SKIP() << "no random access";
    }
}
```

This is preferable to splitting the type list into "indexable" and "not" — the diagnostic is
clearer and the suite stays single-source.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Forgetting `this->` inside `TYPED_TEST` | Compiler can't find dependent base members | Use `this->member_` for every fixture access |
| Name in `REGISTER_TYPED_TEST_SUITE_P` doesn't match a `TYPED_TEST_P` | Compile error or silently missing test | Keep the registration list mechanically synchronized |
| Same `INSTANTIATE_TYPED_TEST_SUITE_P` prefix in two files | Duplicate registration → linker / runtime error | Use a unique prefix per call site |
| `using TypeParam = ...` shadowing | Confusing diagnostics | Use explicit `using T = TypeParam;` if you need a short alias |
| Heavy type lists (e.g. 50 types × 30 tests) | Compile times explode (template instantiations) | Split into multiple suites, or move to runtime-parameterized tests |

## See also

- https://google.github.io/googletest/advanced.html#type-parameterized-tests
- https://google.github.io/googletest/reference/testing.html#TYPED_TEST_SUITE
