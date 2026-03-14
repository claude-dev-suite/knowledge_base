# Testcontainers @ServiceConnection

> Source: https://docs.spring.io/spring-boot/reference/testing/testcontainers.html

## Overview

`@ServiceConnection` is a Spring Boot 3.1+ annotation that automatically creates `ConnectionDetails` beans, eliminating the need for `@DynamicPropertySource` boilerplate.

## Basic Usage

### Before (Legacy)

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

### After (With @ServiceConnection)

```java
@Testcontainers
@SpringBootTest
class ModernTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16");

    // No @DynamicPropertySource needed!
}
```

## Supported Containers

### Databases

```java
// PostgreSQL - Creates JdbcConnectionDetails + R2dbcConnectionDetails
@Container
@ServiceConnection
static PostgreSQLContainer<?> postgres =
    new PostgreSQLContainer<>("postgres:16-alpine");

// MySQL - Creates JdbcConnectionDetails + R2dbcConnectionDetails
@Container
@ServiceConnection
static MySQLContainer<?> mysql =
    new MySQLContainer<>("mysql:8.0");

// MongoDB - Creates MongoConnectionDetails
@Container
@ServiceConnection
static MongoDBContainer mongo =
    new MongoDBContainer("mongo:7.0");

// Cassandra - Creates CassandraConnectionDetails
@Container
@ServiceConnection
static CassandraContainer<?> cassandra =
    new CassandraContainer<>("cassandra:4.1");
```

### Messaging

```java
// Kafka - Creates KafkaConnectionDetails
@Container
@ServiceConnection
static KafkaContainer kafka =
    new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));

// RabbitMQ - Creates RabbitConnectionDetails
@Container
@ServiceConnection
static RabbitMQContainer rabbitmq =
    new RabbitMQContainer("rabbitmq:3.12-management");

// Pulsar - Creates PulsarConnectionDetails
@Container
@ServiceConnection
static PulsarContainer pulsar =
    new PulsarContainer("apachepulsar/pulsar:3.1.0");
```

### Cache & Search

```java
// Redis - Creates RedisConnectionDetails
@Container
@ServiceConnection
static GenericContainer<?> redis =
    new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);

// Elasticsearch - Creates ElasticsearchConnectionDetails
@Container
@ServiceConnection
static ElasticsearchContainer elasticsearch =
    new ElasticsearchContainer("elasticsearch:8.11.0");
```

## GenericContainer with Name

For `GenericContainer`, specify the connection name:

```java
// Redis
@Container
@ServiceConnection(name = "redis")
static GenericContainer<?> redis =
    new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);

// Custom service
@Container
@ServiceConnection(name = "openzipkin/zipkin")
static GenericContainer<?> zipkin =
    new GenericContainer<>("openzipkin/zipkin:latest")
        .withExposedPorts(9411);
```

## Type-Specific Connection

Specify which connection type to create:

```java
// Only JDBC (no R2DBC)
@Container
@ServiceConnection(type = JdbcConnectionDetails.class)
static PostgreSQLContainer<?> postgres =
    new PostgreSQLContainer<>("postgres:16");

// Only Redis
@Container
@ServiceConnection(type = RedisConnectionDetails.class)
static GenericContainer<?> redis =
    new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);
```

## Spring Bean Configuration

Use with `@TestConfiguration` for Spring-managed lifecycle:

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

    @Bean
    @ServiceConnection(name = "redis")
    GenericContainer<?> redisContainer() {
        return new GenericContainer<>("redis:7-alpine")
            .withExposedPorts(6379);
    }
}
```

Usage:

```java
@SpringBootTest
@Import(TestContainersConfig.class)
class MyIntegrationTest {
    // All containers auto-configured
}
```

## Container Configuration Interfaces

Share configurations via interfaces:

```java
interface DatabaseContainers {
    @Container
    @ServiceConnection
    PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16-alpine");
}

interface CacheContainers {
    @Container
    @ServiceConnection(name = "redis")
    GenericContainer<?> redis =
        new GenericContainer<>("redis:7-alpine")
            .withExposedPorts(6379);
}

@TestConfiguration(proxyBeanMethods = false)
@ImportTestcontainers({DatabaseContainers.class, CacheContainers.class})
class TestContainersConfig {}
```

## Multiple Services of Same Type

```java
@TestConfiguration(proxyBeanMethods = false)
class MultipleRedisConfig {

    @Bean
    @ServiceConnection(name = "redis")
    @Qualifier("cache")
    GenericContainer<?> cacheRedis() {
        return new GenericContainer<>("redis:7-alpine")
            .withExposedPorts(6379);
    }

    @Bean
    @ServiceConnection(name = "redis")
    @Qualifier("session")
    GenericContainer<?> sessionRedis() {
        return new GenericContainer<>("redis:7-alpine")
            .withExposedPorts(6379);
    }
}
```

## Connection Details Lookup

Spring Boot looks up connection by:
1. Container class name (e.g., `PostgreSQLContainer`)
2. Image name (e.g., `postgres`)
3. `@ServiceConnection(name = "...")` parameter

## Supported Connection Details

| ConnectionDetails Class | Auto-configured Properties |
|------------------------|---------------------------|
| `JdbcConnectionDetails` | `spring.datasource.*` |
| `R2dbcConnectionDetails` | `spring.r2dbc.*` |
| `MongoConnectionDetails` | `spring.data.mongodb.*` |
| `RedisConnectionDetails` | `spring.data.redis.*` |
| `KafkaConnectionDetails` | `spring.kafka.*` |
| `RabbitConnectionDetails` | `spring.rabbitmq.*` |
| `ElasticsearchConnectionDetails` | `spring.elasticsearch.*` |
| `CassandraConnectionDetails` | `spring.cassandra.*` |
| `Neo4jConnectionDetails` | `spring.neo4j.*` |
| `PulsarConnectionDetails` | `spring.pulsar.*` |

## Best Practices

1. **Always use `@ServiceConnection`** over `@DynamicPropertySource`
2. **Use Spring Bean config** for better lifecycle control
3. **Specify `name` for GenericContainer** with custom images
4. **Use `type` parameter** when you need specific connection only
5. **Set `proxyBeanMethods = false`** in test configurations
