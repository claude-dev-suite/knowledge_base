# Spring Boot Testcontainers Integration

> Source: https://docs.spring.io/spring-boot/reference/testing/testcontainers.html

## Overview

Spring Boot 3.1+ provides first-class support for Testcontainers, enabling automatic service connection configuration and lifecycle management.

## Setup

### Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

## @ServiceConnection (Recommended)

The `@ServiceConnection` annotation automatically configures connection properties, replacing `@DynamicPropertySource`.

### Basic Usage

```java
@SpringBootTest
@Testcontainers
class MyIntegrationTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16-alpine");

    @Test
    void test() {
        // Connection auto-configured via @ServiceConnection
    }
}
```

### Supported Containers

| Container | Connection Details Created |
|-----------|---------------------------|
| `PostgreSQLContainer` | JDBC + R2DBC DataSource |
| `MySQLContainer` | JDBC + R2DBC DataSource |
| `MariaDBContainer` | JDBC + R2DBC DataSource |
| `MongoDBContainer` | MongoConnectionDetails |
| `KafkaContainer` | KafkaConnectionDetails |
| `RedisContainer` | RedisConnectionDetails |
| `RabbitMQContainer` | RabbitConnectionDetails |
| `ElasticsearchContainer` | ElasticsearchConnectionDetails |
| `CassandraContainer` | CassandraConnectionDetails |
| `Neo4jContainer` | Neo4jConnectionDetails |
| `PulsarContainer` | PulsarConnectionDetails |

### GenericContainer with @ServiceConnection

For custom images, specify the connection name:

```java
@Container
@ServiceConnection(name = "redis")
static GenericContainer<?> redis =
    new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);
```

### Type-Specific Connection

```java
@Container
@ServiceConnection(type = JdbcConnectionDetails.class)
static PostgreSQLContainer<?> postgres =
    new PostgreSQLContainer<>("postgres:16");
```

## Lifecycle Management

### Spring Bean Lifecycle (Recommended)

Containers as Spring beans have their lifecycle managed by Spring:

```java
@TestConfiguration(proxyBeanMethods = false)
public class TestContainersConfig {

    @Bean
    @ServiceConnection
    PostgreSQLContainer<?> postgresContainer() {
        return new PostgreSQLContainer<>("postgres:16-alpine");
    }

    @Bean
    @ServiceConnection
    MongoDBContainer mongoContainer() {
        return new MongoDBContainer("mongo:7.0");
    }
}

@SpringBootTest
@Import(TestContainersConfig.class)
class MyIntegrationTest {
    // Containers started before beans, stopped after beans
}
```

**Benefits:**
- Containers start **before** application beans
- Containers stop **after** application beans destroyed
- Proper cleanup order prevents connection loss

### JUnit Extension Lifecycle

```java
@Testcontainers
@SpringBootTest
class MyIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16");
}
```

**Warning:** JUnit lifecycle doesn't guarantee shutdown order. Use Spring Bean lifecycle for containers that application beans depend on.

## Container Configuration Interfaces

Share container configurations across tests:

```java
interface PostgresContainers {
    @Container
    @ServiceConnection
    PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16-alpine");
}

interface MongoContainers {
    @Container
    @ServiceConnection
    MongoDBContainer mongo =
        new MongoDBContainer("mongo:7.0");
}

@TestConfiguration(proxyBeanMethods = false)
@ImportTestcontainers({PostgresContainers.class, MongoContainers.class})
class TestContainersConfig {}
```

## Legacy: @DynamicPropertySource

Before Spring Boot 3.1, use `@DynamicPropertySource`:

```java
@Testcontainers
@SpringBootTest
class LegacyTest {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}
```

## SSL/TLS Support

```java
@Testcontainers
@SpringBootTest
class SecureRedisTest {

    @Container
    @ServiceConnection
    @PemKeyStore(certificate = "classpath:client.crt", privateKey = "classpath:client.key")
    @PemTrustStore("classpath:ca.crt")
    static RedisContainer redis = new SecureRedisContainer("redis:7");
}
```

## Multiple Containers

```java
@SpringBootTest
@Testcontainers
class FullStackIntegrationTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16-alpine");

    @Container
    @ServiceConnection
    static GenericContainer<?> redis =
        new GenericContainer<>("redis:7-alpine")
            .withExposedPorts(6379);

    @Container
    @ServiceConnection
    static KafkaContainer kafka =
        new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));
}
```

## Development Mode

Use Testcontainers during local development:

```java
@TestConfiguration(proxyBeanMethods = false)
public class TestApplication {

    @Bean
    @ServiceConnection
    PostgreSQLContainer<?> postgresContainer() {
        return new PostgreSQLContainer<>("postgres:16-alpine")
            .withReuse(true); // Keep running between restarts
    }

    public static void main(String[] args) {
        SpringApplication.from(MyApplication::main)
            .with(TestApplication.class)
            .run(args);
    }
}
```

Run with: `./mvnw spring-boot:test-run`

## Best Practices

1. **Use `@ServiceConnection`** instead of `@DynamicPropertySource`
2. **Use Spring Bean lifecycle** for production-like tests
3. **Use static containers** for test class reuse
4. **Enable container reuse** in development
5. **Use alpine images** for faster startup
6. **Set `proxyBeanMethods = false`** in test configurations
