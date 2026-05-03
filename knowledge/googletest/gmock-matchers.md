# GoogleTest - GMock Matchers Catalog

> **Source**: https://google.github.io/googletest/reference/matchers.html
> **Skill**: dev-suite skill `testing/googletest` — see SKILL.md for the always-loaded quick reference.

## What this covers

The full matcher catalog organized by what you're matching against (containers, members,
pointers, strings) plus composition (`AllOf`, `AnyOf`, `Not`) and writing custom matchers
with `MATCHER_P` for domain-specific assertions.

## Deep dive

### Matchers by target type

| Target | Matchers |
|--------|----------|
| Generic | `Eq`, `Ne`, `Lt`, `Le`, `Gt`, `Ge`, `IsNull`, `NotNull`, `Ref(var)`, `Truly(pred)`, `_` |
| Numeric | `FloatEq`, `DoubleEq`, `FloatNear(x, eps)`, `DoubleNear`, `IsNan`, `IsInf`, `IsFinite` |
| String | `StrEq`, `StrNe`, `StrCaseEq`, `HasSubstr`, `StartsWith`, `EndsWith`, `MatchesRegex`, `ContainsRegex` |
| Containers | `IsEmpty`, `SizeIs(n)`, `Contains(elem)`, `Each(matcher)`, `ElementsAre(...)`, `ElementsAreArray(arr)`, `UnorderedElementsAre(...)`, `WhenSorted(matcher)`, `Pointwise(m, b)` |
| Optional | `Optional(matcher)`, `Eq(std::nullopt)` |
| Variant | `VariantWith<T>(matcher)` |
| Pair | `Pair(k_matcher, v_matcher)` |
| Pointers/smart ptrs | `Pointee(matcher)`, `WhenDynamicCastTo<T*>(matcher)` |
| Members | `Field(&Struct::m, matcher)`, `Property(&Class::getter, matcher)` |
| Composition | `AllOf(...)`, `AnyOf(...)`, `Not(matcher)`, `AnyOfArray`, `AllOfArray` |

### Composing struct matchers

```cpp
struct Order {
    int id;
    std::string customer;
    std::vector<LineItem> items;
};

EXPECT_THAT(order, ::testing::AllOf(
    ::testing::Field(&Order::id, ::testing::Gt(0)),
    ::testing::Field(&Order::customer, ::testing::HasSubstr("@example.com")),
    ::testing::Field(&Order::items, ::testing::SizeIs(::testing::Ge(1)))
));
```

`Property` works the same but takes a member-function pointer — useful for classes with private
data:

```cpp
EXPECT_THAT(user, ::testing::AllOf(
    ::testing::Property(&User::age, ::testing::Ge(18)),
    ::testing::Property(&User::name, ::testing::Not(::testing::IsEmpty()))
));
```

### Container matchers — picking the right one

```cpp
std::vector<int> v = {3, 1, 2};

EXPECT_THAT(v, ::testing::ElementsAre(3, 1, 2));            // exact order, exact size
EXPECT_THAT(v, ::testing::UnorderedElementsAre(1, 2, 3));   // any order, exact size
EXPECT_THAT(v, ::testing::Contains(2));                     // at least one match
EXPECT_THAT(v, ::testing::Each(::testing::Lt(10)));         // every element matches
EXPECT_THAT(v, ::testing::WhenSorted(::testing::ElementsAre(1, 2, 3)));
```

`ElementsAre` will fail loudly if the size differs ("which has 4 elements"), unlike a hand-rolled
loop that would early-out and obscure the size mismatch.

### Pointwise — element-by-element comparison

```cpp
std::vector<double> got      = {1.0001, 2.0002, 3.0003};
std::vector<double> expected = {1.0,    2.0,    3.0};

EXPECT_THAT(got, ::testing::Pointwise(::testing::DoubleNear(0.001), expected));
```

### Custom matchers with `MATCHER_P`

```cpp
MATCHER_P(IsCloseTo, target,
          std::string(negation ? "isn't" : "is") + " close to "
          + ::testing::PrintToString(target)) {
    return std::abs(arg - target) <= 0.01 * std::abs(target);
}

EXPECT_THAT(measure(), IsCloseTo(100.0));
```

Variants:
- `MATCHER(name, description) { return ...; }`
- `MATCHER_P(name, p1, description) { ... }`
- `MATCHER_P2`, `MATCHER_P3`, ... up to 10 parameters

### Using matchers in `EXPECT_CALL`

```cpp
EXPECT_CALL(mock, save(::testing::Field(&User::age, ::testing::Ge(18))))
    .Times(1);
```

This is the highest-leverage matcher use: assert on the *shape* of an argument, not its exact
identity. The diagnostic on failure says exactly which sub-matcher tripped.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| `EXPECT_THAT(v, ContainerEq(expected))` for huge vectors | Diff dump explodes the log | `ElementsAreArray` with a slice, or count + sample assertions |
| Mixing matchers with raw values: `EXPECT_CALL(m, f(42, _))` | Works but `_` must come from `::testing` | Confirm `using ::testing::_;` is in scope |
| `Pointee(_)` to "ignore the value" | Still asserts the pointer is non-null | Combine with `Eq(nullptr)` if null is acceptable |
| Custom matcher ignoring `negation` | Inverted diagnostic reads "is X" when it actually wasn't | Use `negation ? "isn't" : "is"` in the description |
| Forgetting to import composition matchers | `AllOf` not found | `using namespace ::testing;` in test file is fine and idiomatic |

## See also

- https://google.github.io/googletest/gmock_cook_book.html#NewMatchers
- https://google.github.io/googletest/reference/matchers.html#container-matchers
