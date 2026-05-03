# GoogleTest - GMock Fundamentals

> **Source**: https://google.github.io/googletest/gmock_for_dummies.html
> **Skill**: dev-suite skill `testing/googletest` — see SKILL.md for the always-loaded quick reference.

## What this covers

The deeper mechanics of `MOCK_METHOD`, `EXPECT_CALL`, sequencing, default actions, and the
`NiceMock` / `NaggyMock` / `StrictMock` strictness model — including when each is the right
default for a particular layer of your codebase.

## Deep dive

### `MOCK_METHOD` macro signature

```cpp
class IRepo {
public:
    virtual ~IRepo() = default;
    virtual std::optional<User> find(int id) const = 0;
    virtual void save(const User& u) = 0;
    virtual int  count(std::string_view tag) const noexcept = 0;
};

class MockRepo : public IRepo {
public:
    MOCK_METHOD(std::optional<User>, find,  (int id),               (const, override));
    MOCK_METHOD(void,                save,  (const User& u),        (override));
    MOCK_METHOD(int,                 count, (std::string_view tag), (const, noexcept, override));
};
```

The fourth argument is a **parenthesized list** of qualifiers: `const`, `override`, `noexcept`,
`ref`, `final`, and the calling convention. Forgetting `override` is the most common cause of a
mock that "compiles but never gets called" — the derived `find` doesn't override anything because
the signature doesn't match the base.

### Mocking methods with parenthesized return/argument types

`MOCK_METHOD` is a macro and the comma in `std::map<int, int>` confuses the preprocessor.
Wrap with extra parens:

```cpp
MOCK_METHOD((std::map<int, int>), histogram, (), (const, override));
//          ^^^^^^^^^^^^^^^^^^^^   parens around return type
MOCK_METHOD(void, set, ((std::pair<int, int>) p), (override));
```

### Default actions and `ON_CALL`

`EXPECT_CALL` sets *an expectation* (usually scoped to a single test). `ON_CALL` sets a
*default action* without expecting any call:

```cpp
ON_CALL(mock, find(::testing::_))
    .WillByDefault(::testing::Return(std::nullopt));

EXPECT_CALL(mock, find(42))
    .WillOnce(::testing::Return(User{42, "alice"}));
```

The pattern: configure default behaviors in `SetUp()` with `ON_CALL`, then in each test add the
specific `EXPECT_CALL` that matters for that scenario. This avoids "boilerplate
`EXPECT_CALL(_).Times(AnyNumber())`" in every test.

### Strictness modes

```cpp
using ::testing::NiceMock;
using ::testing::NaggyMock;
using ::testing::StrictMock;

NiceMock<MockRepo> nice;     // unexpected calls: silent (returns default)
NaggyMock<MockRepo> naggy;   // unexpected calls: warning (this is the DEFAULT)
StrictMock<MockRepo> strict; // unexpected calls: test failure
```

Guidance:
- **NiceMock** for collaborator mocks: most tests don't care that the logger was called 17 times.
- **StrictMock** for the surface under test: e.g. testing that an event publisher emits exactly
  the events you expect, no extras.
- **NaggyMock** is rarely correct as the default — it floods CI with warnings.

### Sequencing

```cpp
using ::testing::InSequence;

TEST(WorkflowTest, OrdersOperations) {
    MockDb db;
    {
        InSequence seq;
        EXPECT_CALL(db, begin());
        EXPECT_CALL(db, write(::testing::_));
        EXPECT_CALL(db, commit());
    }
    perform_workflow(db);
}
```

`InSequence` enforces order across all expectations in scope. For partial ordering, use named
`Sequence` objects:

```cpp
::testing::Sequence s1, s2;
EXPECT_CALL(db, begin()).InSequence(s1, s2);
EXPECT_CALL(db, write_users()).InSequence(s1);
EXPECT_CALL(db, write_logs()).InSequence(s2);
EXPECT_CALL(db, commit()).InSequence(s1, s2);
```

### `WillOnce` chain vs `WillRepeatedly`

```cpp
EXPECT_CALL(repo, count(::testing::_))
    .WillOnce(::testing::Return(0))
    .WillOnce(::testing::Return(1))
    .WillRepeatedly(::testing::Return(2));   // 3rd call onward
```

If `WillOnce` runs out and there is no `WillRepeatedly`, gmock returns the default value AND emits
a warning (or failure under StrictMock).

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| `MOCK_METHOD` without `override` | Doesn't override the virtual; SUT calls the abstract base | Always add `override` to the qualifier list |
| `EXPECT_CALL` after `func()` is invoked | gmock requires expectations *before* the action | Set all `EXPECT_CALL` in arrange phase |
| Mocking concrete (non-virtual) methods | Calls go through the original | Introduce an interface, or use a template seam |
| Comma inside the return type | Preprocessor splits the macro args | Wrap return type in extra parens |
| `StrictMock` everywhere | One harmless extra log call breaks 30 tests | Default to `NiceMock`; `StrictMock` only for narrow contracts |
| `ON_CALL` without `EXPECT_CALL` for a method you also want to verify | Test passes regardless of whether SUT called it | Add an explicit `EXPECT_CALL` |

## See also

- https://google.github.io/googletest/gmock_cook_book.html
- https://google.github.io/googletest/reference/mocking.html
