# Testcontainers Lifecycle Management

> Source: https://java.testcontainers.org/test_framework_integration/junit_5/

## Overview

Understanding container lifecycle is critical for reliable tests and optimal performance.

## JUnit 5 Lifecycle

### Static Containers (Class-level)

Shared across all test methods in a class:

```java
@Testcontainers
class MyTest {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16");

    @Test
    void test1() {
        // Same container
    }

    @Test
    void test2() {
        // Same container
    }
}
```

**Lifecycle:**
1. Container starts **once** before first test
2. All tests share the same container
3. Container stops after last test

### Instance Containers (Method-level)

New container for each test method:

```java
@Testcontainers
class MyTest {

    @Container
    PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16");

    @Test
    void test1() {
        // Fresh container
    }

    @Test
    void test2() {
        // Different fresh container
    }
}
```

**Lifecycle:**
1. Container starts before **each** test
2. Container stops after **each** test
3. Complete isolation between tests

## Spring Boot Lifecycle

### JUnit Extension (Not Recommended for Complex Tests)

```java
@Testcontainers
@SpringBootTest
class MyTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16");
}
```

**Warning:** JUnit extension doesn't guarantee shutdown order relative to Spring beans.

### Spring Bean Lifecycle (Recommended)

```java
@TestConfiguration(proxyBeanMethods = false)
class TestContainersConfig {

    @Bean
    @ServiceConnection
    PostgreSQLContainer<?> postgresContainer() {
        return new PostgreSQLContainer<>("postgres:16");
    }
}

@SpringBootTest
@Import(TestContainersConfig.class)
class MyTest {
    // Container lifecycle managed by Spring
}
```

**Benefits:**
- Containers start **before** application beans
- Containers stop **after** all beans destroyed
- Prevents connection loss during shutdown

## Lifecycle Comparison

| Approach | Start | Stop | Best For |
|----------|-------|------|----------|
| Static `@Container` | Before class | After class | Fast tests, shared state |
| Instance `@Container` | Before each test | After each test | Isolation |
| Spring Bean | With context | After context | Integration tests |
| Singleton pattern | Once per JVM | Never (reuse) | Development |

## Singleton Pattern (Cross-class Sharing)

```java
public abstract class AbstractIntegrationTest {

    static final PostgreSQLContainer<?> POSTGRES;

    static {
        POSTGRES = new PostgreSQLContainer<>("postgres:16-alpine");
        POSTGRES.start();
        // Don't call stop() - container reused across classes
    }

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);
    }
}

class Test1 extends AbstractIntegrationTest {
    // Uses shared POSTGRES container
}

class Test2 extends AbstractIntegrationTest {
    // Uses same shared POSTGRES container
}
```

## Container Reuse (Development)

Keep containers running between test runs:

```java
static PostgreSQLContainer<?> postgres =
    new PostgreSQLContainer<>("postgres:16-alpine")
        .withReuse(true);
```

Enable in `~/.testcontainers.properties`:
```properties
testcontainers.reuse.enable=true
```

**Benefits:**
- Faster subsequent test runs
- No container startup delay
- Keeps data between runs (be careful!)

**Cleanup:**
```bash
# Stop all reusable containers
docker stop $(docker ps -q --filter "label=org.testcontainers.reuse=true")
```

## Parallel Test Execution

### Isolated Containers (Safe)

```java
@Execution(ExecutionMode.CONCURRENT)
class ParallelTest {

    @Container
    PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16");

    // Each test gets its own container - safe for parallel
}
```

### Shared Container (Requires Isolation)

```java
@Execution(ExecutionMode.CONCURRENT)
class ParallelTest {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16");

    @BeforeEach
    void isolateData() {
        // Clean or isolate data between tests
        jdbcTemplate.execute("TRUNCATE users CASCADE");
    }
}
```

## Lifecycle Hooks

### Custom Startup Logic

```java
@Container
static PostgreSQLContainer<?> postgres =
    new PostgreSQLContainer<>("postgres:16") {
        @Override
        public void start() {
            super.start();
            // Custom initialization
            System.out.println("Container started: " + getJdbcUrl());
        }
    };
```

### @BeforeAll / @AfterAll with Containers

```java
@Testcontainers
class MyTest {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16");

    @BeforeAll
    static void setupSchema() {
        // Container already started
        try (Connection conn = DriverManager.getConnection(
                postgres.getJdbcUrl(),
                postgres.getUsername(),
                postgres.getPassword())) {
            conn.createStatement().execute("CREATE SCHEMA test");
        }
    }

    @AfterAll
    static void cleanup() {
        // Container still running
        // Cleanup if needed
    }
}
```

## Troubleshooting Lifecycle Issues

### Container Not Ready

```java
new GenericContainer<>("myapp:latest")
    .waitingFor(Wait.forHttp("/health").forStatusCode(200))
    .withStartupTimeout(Duration.ofMinutes(2));
```

### Connection Refused After Shutdown

Use Spring Bean lifecycle instead of JUnit:

```java
// BAD: JUnit manages lifecycle
@Container
@ServiceConnection
static PostgreSQLContainer<?> postgres = ...;

// GOOD: Spring manages lifecycle
@Bean
@ServiceConnection
PostgreSQLContainer<?> postgres() { return ...; }
```

### Slow Tests

Enable reuse or use singleton pattern:

```java
static PostgreSQLContainer<?> postgres =
    new PostgreSQLContainer<>("postgres:16-alpine")
        .withReuse(true);
```

## Best Practices

1. **Use static containers** for most integration tests
2. **Use Spring Bean lifecycle** for complex Spring Boot apps
3. **Use singleton pattern** for cross-class sharing
4. **Enable reuse** during development
5. **Use instance containers** only when isolation is critical
6. **Add proper wait strategies** for reliable startup
7. **Clean data, not containers** for test isolation
