# Testcontainers Basics

> Source: https://testcontainers.com/guides/getting-started-with-testcontainers-for-java/

## Overview

Testcontainers is a Java library that supports JUnit tests, providing lightweight, throwaway instances of databases, message brokers, web browsers, or anything that can run in a Docker container.

## Setup

### Maven

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <version>1.19.3</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>1.19.3</version>
    <scope>test</scope>
</dependency>
```

### Database Modules

```xml
<!-- PostgreSQL -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>

<!-- MySQL -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>mysql</artifactId>
    <scope>test</scope>
</dependency>

<!-- MongoDB -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>mongodb</artifactId>
    <scope>test</scope>
</dependency>
```

## JUnit 5 Integration

### @Testcontainers Annotation

```java
@Testcontainers
class MyIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16-alpine");

    @Test
    void test() {
        String jdbcUrl = postgres.getJdbcUrl();
        String username = postgres.getUsername();
        String password = postgres.getPassword();
        // Use connection details
    }
}
```

### Static vs Instance Containers

**Static (shared across test methods):**
```java
@Container
static PostgreSQLContainer<?> postgres =
    new PostgreSQLContainer<>("postgres:16");
```

**Instance (new container per test method):**
```java
@Container
PostgreSQLContainer<?> postgres =
    new PostgreSQLContainer<>("postgres:16");
```

## Database Containers

### PostgreSQL

```java
@Container
static PostgreSQLContainer<?> postgres =
    new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test")
        .withInitScript("init.sql");  // Run SQL script on startup
```

### MySQL

```java
@Container
static MySQLContainer<?> mysql =
    new MySQLContainer<>("mysql:8.0")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
```

### MongoDB

```java
@Container
static MongoDBContainer mongo =
    new MongoDBContainer("mongo:7.0");

// Get connection string
String connectionString = mongo.getReplicaSetUrl();
```

## Messaging Containers

### Kafka

```java
@Container
static KafkaContainer kafka =
    new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.5.0"))
        .withKraft();  // Use KRaft mode (no ZooKeeper)

// Get bootstrap servers
String bootstrapServers = kafka.getBootstrapServers();
```

### RabbitMQ

```java
@Container
static RabbitMQContainer rabbitmq =
    new RabbitMQContainer("rabbitmq:3.12-management")
        .withExposedPorts(5672, 15672);

String amqpUrl = rabbitmq.getAmqpUrl();
```

## GenericContainer

For images without dedicated modules:

```java
@Container
static GenericContainer<?> redis =
    new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);

// Get mapped port
Integer mappedPort = redis.getMappedPort(6379);
String host = redis.getHost();
```

### With Environment Variables

```java
@Container
static GenericContainer<?> app =
    new GenericContainer<>("myapp:latest")
        .withExposedPorts(8080)
        .withEnv("DATABASE_URL", "jdbc:postgresql://db:5432/test")
        .withEnv("REDIS_HOST", "redis");
```

### With Commands

```java
@Container
static GenericContainer<?> container =
    new GenericContainer<>("alpine:latest")
        .withCommand("tail", "-f", "/dev/null");
```

## Wait Strategies

### Wait for Port

```java
new GenericContainer<>("myapp:latest")
    .waitingFor(Wait.forListeningPort());
```

### Wait for HTTP

```java
new GenericContainer<>("myapp:latest")
    .waitingFor(Wait.forHttp("/health")
        .forPort(8080)
        .forStatusCode(200));
```

### Wait for Log Message

```java
new GenericContainer<>("myapp:latest")
    .waitingFor(Wait.forLogMessage(".*Started Application.*", 1));
```

### With Timeout

```java
new GenericContainer<>("myapp:latest")
    .waitingFor(Wait.forHttp("/health"))
    .withStartupTimeout(Duration.ofMinutes(2));
```

## Container Networks

```java
static Network network = Network.newNetwork();

@Container
static PostgreSQLContainer<?> postgres =
    new PostgreSQLContainer<>("postgres:16")
        .withNetwork(network)
        .withNetworkAliases("postgres");

@Container
static GenericContainer<?> app =
    new GenericContainer<>("myapp:latest")
        .withNetwork(network)
        .dependsOn(postgres)
        .withEnv("DATABASE_HOST", "postgres");
```

## Docker Compose

```java
@Container
static DockerComposeContainer<?> compose =
    new DockerComposeContainer<>(new File("docker-compose-test.yml"))
        .withExposedService("postgres", 5432)
        .withExposedService("redis", 6379);

@Test
void test() {
    String pgHost = compose.getServiceHost("postgres", 5432);
    int pgPort = compose.getServicePort("postgres", 5432);
}
```

## Resource Management

### Singleton Containers

```java
public abstract class AbstractIntegrationTest {

    static PostgreSQLContainer<?> postgres;

    static {
        postgres = new PostgreSQLContainer<>("postgres:16-alpine");
        postgres.start();
    }
}
```

### Container Reuse

```java
static PostgreSQLContainer<?> postgres =
    new PostgreSQLContainer<>("postgres:16-alpine")
        .withReuse(true);
```

Requires `~/.testcontainers.properties`:
```
testcontainers.reuse.enable=true
```

## Best Practices

1. **Use static containers** for shared state across tests
2. **Use alpine images** for faster downloads
3. **Enable reuse** during development
4. **Use wait strategies** for reliable startup
5. **Clean up test data** between tests, not containers
6. **Use specific image tags** instead of `latest`
