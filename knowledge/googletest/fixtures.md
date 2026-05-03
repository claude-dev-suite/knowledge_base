# GoogleTest - Fixtures, Suite Setup, Test Lifecycles

> **Source**: https://google.github.io/googletest/advanced.html#sharing-resources-between-tests-in-the-same-test-suite
> **Skill**: dev-suite skill `testing/googletest` — see SKILL.md for the always-loaded quick reference.

## What this covers

The four lifecycle hooks (`SetUp`, `TearDown`, `SetUpTestSuite`, `TearDownTestSuite`), their
exact ordering, when shared state is dangerous vs valid, and the `Environment` mechanism for
process-wide setup such as opening a shared GPU context or starting a test container.

## Deep dive

### Fixture lifecycle for `TEST_F(MyFixture, ...)`

For each test in the suite, GoogleTest performs:

1. Construct a fresh `MyFixture` instance.
2. Call `SetUp()`.
3. Run the test body.
4. Call `TearDown()` (even if the body threw or fatally asserted).
5. Destroy the instance.

The constructor and `SetUp` differ in failure semantics: `ASSERT_*` in a constructor cannot abort
the test — the test body still runs. Use `SetUp()` if any setup step should be able to skip the
test via `GTEST_SKIP()` or fatally fail via `ASSERT_*`.

```cpp
class HttpClientTest : public ::testing::Test {
protected:
    void SetUp() override {
        if (!server_.start()) GTEST_SKIP() << "test server unavailable";
        ASSERT_TRUE(client_.connect(server_.endpoint()));
    }
    void TearDown() override { client_.disconnect(); server_.stop(); }

    TestServer server_;
    HttpClient client_;
};
```

### Suite-level setup (run once per suite)

```cpp
class GpuTest : public ::testing::Test {
protected:
    static void SetUpTestSuite() {
        // Called ONCE before the first test of the suite
        ctx_ = std::make_unique<GpuContext>();
        ASSERT_TRUE(ctx_->initialize());
    }
    static void TearDownTestSuite() {
        ctx_.reset();
    }

    static std::unique_ptr<GpuContext> ctx_;   // shared across tests
};
std::unique_ptr<GpuContext> GpuTest::ctx_;

TEST_F(GpuTest, AllocatesBuffer) {
    EXPECT_TRUE(ctx_->alloc(1024));
}
```

Rules: the static state **must be reset** between tests (or tests must be order-independent), and
`SetUpTestSuite` runs lazily — if no test in the suite is selected by `--gtest_filter`, it is
skipped entirely.

### Environment — process-wide, suite-agnostic setup

```cpp
class TestContainerEnv : public ::testing::Environment {
public:
    void SetUp() override   { container_.start(); }
    void TearDown() override { container_.stop(); }
private:
    PostgresTestContainer container_;
};

int main(int argc, char** argv) {
    ::testing::InitGoogleTest(&argc, argv);
    ::testing::AddGlobalTestEnvironment(new TestContainerEnv);
    return RUN_ALL_TESTS();
}
```

`Environment::SetUp` runs once before any suite, `TearDown` after the last suite — ideal for
heavyweight resources shared across suites (databases, license servers).

### Skipping at runtime

```cpp
TEST(NetworkTest, RequiresIPv6) {
    if (!has_ipv6_loopback()) GTEST_SKIP() << "no IPv6 loopback";
    // ...
}
```

`GTEST_SKIP()` works in fixtures, suite setup, and inside test bodies. It produces a `SKIPPED`
status in CTest output rather than a failure.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Mutating `static` suite state in tests | Order dependence; flaky in `--gtest_shuffle` | Reset in `SetUp` or use per-test members |
| `ASSERT_*` inside the fixture constructor | Has no abort effect — test body still runs | Move the assertion into `SetUp()` |
| Forgetting to define the static member out-of-line | Linker error | `std::unique_ptr<X> Suite::ctx_;` at namespace scope |
| Calling `ctx_->...` after `TearDownTestSuite` ran | Already destroyed | Don't reference suite state from `Environment::TearDown` |
| Heavy state in `SetUp` for fast tests | Multiplies cost by N tests | Move to `SetUpTestSuite` if per-test reset isn't required |

## See also

- https://google.github.io/googletest/advanced.html#global-set-up-and-tear-down
- https://google.github.io/googletest/reference/testing.html#Test
