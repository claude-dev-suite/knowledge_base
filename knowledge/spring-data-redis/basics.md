# Spring Data Redis - Basics

## Overview

Spring Data Redis provides easy configuration and access to Redis from Spring applications. It offers both low-level and high-level abstractions for interacting with Redis.

## Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

For reactive support:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
```

## Configuration

### application.yml
```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      password: ${REDIS_PASSWORD:}
      database: 0
      timeout: 2000ms
      lettuce:
        pool:
          max-active: 8
          max-idle: 8
          min-idle: 0
          max-wait: -1ms
```

### Cluster Configuration
```yaml
spring:
  data:
    redis:
      cluster:
        nodes:
          - node1:6379
          - node2:6379
          - node3:6379
        max-redirects: 3
```

### Sentinel Configuration
```yaml
spring:
  data:
    redis:
      sentinel:
        master: mymaster
        nodes:
          - sentinel1:26379
          - sentinel2:26379
          - sentinel3:26379
```

## Java Configuration

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        RedisStandaloneConfiguration config = new RedisStandaloneConfiguration();
        config.setHostName("localhost");
        config.setPort(6379);
        config.setPassword(RedisPassword.of("secret"));

        LettuceClientConfiguration clientConfig = LettuceClientConfiguration.builder()
            .commandTimeout(Duration.ofSeconds(2))
            .shutdownTimeout(Duration.ZERO)
            .build();

        return new LettuceConnectionFactory(config, clientConfig);
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);

        // Key serializer
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());

        // Value serializer
        Jackson2JsonRedisSerializer<Object> serializer =
            new Jackson2JsonRedisSerializer<>(Object.class);

        ObjectMapper mapper = new ObjectMapper();
        mapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        mapper.activateDefaultTyping(
            LaissezFaireSubTypeValidator.instance,
            ObjectMapper.DefaultTyping.NON_FINAL
        );
        serializer.setObjectMapper(mapper);

        template.setValueSerializer(serializer);
        template.setHashValueSerializer(serializer);

        template.afterPropertiesSet();
        return template;
    }
}
```

## Connection Factories

### Lettuce (Default)
```java
@Bean
public LettuceConnectionFactory lettuceConnectionFactory() {
    LettuceClientConfiguration config = LettuceClientConfiguration.builder()
        .useSsl()
        .and()
        .commandTimeout(Duration.ofSeconds(2))
        .shutdownTimeout(Duration.ZERO)
        .build();

    return new LettuceConnectionFactory(
        new RedisStandaloneConfiguration("localhost", 6379),
        config
    );
}
```

### Jedis
```java
@Bean
public JedisConnectionFactory jedisConnectionFactory() {
    JedisClientConfiguration config = JedisClientConfiguration.builder()
        .connectTimeout(Duration.ofSeconds(2))
        .readTimeout(Duration.ofSeconds(2))
        .usePooling()
        .poolConfig(new JedisPoolConfig())
        .build();

    return new JedisConnectionFactory(
        new RedisStandaloneConfiguration("localhost", 6379),
        config
    );
}
```

## Serializers

### Available Serializers
```java
// String serializer
StringRedisSerializer stringSerializer = new StringRedisSerializer();

// JSON serializer
Jackson2JsonRedisSerializer<User> jsonSerializer =
    new Jackson2JsonRedisSerializer<>(User.class);

// Generic JSON serializer with type info
GenericJackson2JsonRedisSerializer genericJsonSerializer =
    new GenericJackson2JsonRedisSerializer();

// Java serialization (not recommended)
JdkSerializationRedisSerializer jdkSerializer =
    new JdkSerializationRedisSerializer();

// Custom serializer
public class CustomSerializer implements RedisSerializer<MyObject> {
    @Override
    public byte[] serialize(MyObject value) {
        // Custom serialization logic
    }

    @Override
    public MyObject deserialize(byte[] bytes) {
        // Custom deserialization logic
    }
}
```

## Basic Operations

```java
@Service
@RequiredArgsConstructor
public class RedisService {

    private final RedisTemplate<String, Object> redisTemplate;

    // String operations
    public void setString(String key, String value) {
        redisTemplate.opsForValue().set(key, value);
    }

    public void setWithExpiry(String key, Object value, Duration ttl) {
        redisTemplate.opsForValue().set(key, value, ttl);
    }

    public Object get(String key) {
        return redisTemplate.opsForValue().get(key);
    }

    // Delete
    public Boolean delete(String key) {
        return redisTemplate.delete(key);
    }

    // Check existence
    public Boolean exists(String key) {
        return redisTemplate.hasKey(key);
    }

    // Set expiry
    public Boolean expire(String key, Duration timeout) {
        return redisTemplate.expire(key, timeout);
    }

    // Get TTL
    public Long getExpire(String key) {
        return redisTemplate.getExpire(key);
    }
}
```

## Data Structures

### Hash Operations
```java
public void hashOperations() {
    HashOperations<String, String, Object> hashOps = redisTemplate.opsForHash();

    // Put single field
    hashOps.put("user:1", "name", "John");
    hashOps.put("user:1", "email", "john@example.com");

    // Put all fields
    Map<String, Object> userData = Map.of(
        "name", "John",
        "email", "john@example.com",
        "age", 30
    );
    hashOps.putAll("user:1", userData);

    // Get single field
    Object name = hashOps.get("user:1", "name");

    // Get all fields
    Map<String, Object> allFields = hashOps.entries("user:1");

    // Delete field
    hashOps.delete("user:1", "age");
}
```

### List Operations
```java
public void listOperations() {
    ListOperations<String, Object> listOps = redisTemplate.opsForList();

    // Push to left/right
    listOps.leftPush("queue", "item1");
    listOps.rightPush("queue", "item2");

    // Push multiple
    listOps.rightPushAll("queue", "item3", "item4", "item5");

    // Pop from left/right
    Object item = listOps.leftPop("queue");

    // Blocking pop
    Object blockedItem = listOps.leftPop("queue", Duration.ofSeconds(5));

    // Range
    List<Object> items = listOps.range("queue", 0, -1);

    // Size
    Long size = listOps.size("queue");
}
```

### Set Operations
```java
public void setOperations() {
    SetOperations<String, Object> setOps = redisTemplate.opsForSet();

    // Add members
    setOps.add("tags", "java", "spring", "redis");

    // Check membership
    Boolean isMember = setOps.isMember("tags", "java");

    // Get all members
    Set<Object> members = setOps.members("tags");

    // Remove
    setOps.remove("tags", "redis");

    // Set operations
    Set<Object> intersection = setOps.intersect("tags1", "tags2");
    Set<Object> union = setOps.union("tags1", "tags2");
    Set<Object> difference = setOps.difference("tags1", "tags2");
}
```

### Sorted Set Operations
```java
public void zSetOperations() {
    ZSetOperations<String, Object> zSetOps = redisTemplate.opsForZSet();

    // Add with score
    zSetOps.add("leaderboard", "player1", 100.0);
    zSetOps.add("leaderboard", "player2", 85.0);

    // Add multiple
    Set<ZSetOperations.TypedTuple<Object>> tuples = Set.of(
        ZSetOperations.TypedTuple.of("player3", 95.0),
        ZSetOperations.TypedTuple.of("player4", 80.0)
    );
    zSetOps.add("leaderboard", tuples);

    // Get rank
    Long rank = zSetOps.rank("leaderboard", "player1");
    Long reverseRank = zSetOps.reverseRank("leaderboard", "player1");

    // Get score
    Double score = zSetOps.score("leaderboard", "player1");

    // Range by rank
    Set<Object> topPlayers = zSetOps.reverseRange("leaderboard", 0, 9);

    // Range by score
    Set<Object> highScorers = zSetOps.rangeByScore("leaderboard", 80, 100);

    // Increment score
    zSetOps.incrementScore("leaderboard", "player1", 10.0);
}
```

## Reactive Redis

```java
@Configuration
public class ReactiveRedisConfig {

    @Bean
    public ReactiveRedisTemplate<String, Object> reactiveRedisTemplate(
            ReactiveRedisConnectionFactory factory) {

        StringRedisSerializer keySerializer = new StringRedisSerializer();
        Jackson2JsonRedisSerializer<Object> valueSerializer =
            new Jackson2JsonRedisSerializer<>(Object.class);

        RedisSerializationContext<String, Object> context =
            RedisSerializationContext.<String, Object>newSerializationContext()
                .key(keySerializer)
                .value(valueSerializer)
                .hashKey(keySerializer)
                .hashValue(valueSerializer)
                .build();

        return new ReactiveRedisTemplate<>(factory, context);
    }
}

@Service
@RequiredArgsConstructor
public class ReactiveRedisService {

    private final ReactiveRedisTemplate<String, Object> redisTemplate;

    public Mono<Boolean> set(String key, Object value) {
        return redisTemplate.opsForValue().set(key, value);
    }

    public Mono<Object> get(String key) {
        return redisTemplate.opsForValue().get(key);
    }

    public Flux<Object> getAll(List<String> keys) {
        return redisTemplate.opsForValue().multiGet(keys)
            .flatMapMany(Flux::fromIterable);
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use appropriate serializers | Use JDK serialization |
| Configure connection pooling | Create new connections |
| Set TTL for cache entries | Let cache grow unbounded |
| Use pipelining for bulk ops | Individual commands in loops |
| Monitor memory usage | Ignore memory limits |
| Use key prefixes/namespaces | Flat key structure |

## Production Checklist

- [ ] Connection pooling configured
- [ ] Appropriate timeouts set
- [ ] Serializers chosen and tested
- [ ] SSL/TLS enabled
- [ ] Memory limits configured
- [ ] Eviction policy set
- [ ] Monitoring enabled
- [ ] Failover tested (Sentinel/Cluster)
