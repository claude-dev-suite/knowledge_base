# Spring Session - Redis

## Configuration

### Dependencies
```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

### application.yml
```yaml
spring:
  session:
    store-type: redis
    redis:
      namespace: myapp:session
      flush-mode: on-save  # or immediate
      save-mode: on-set-attribute  # or always
    timeout: 30m

  redis:
    host: ${REDIS_HOST:localhost}
    port: ${REDIS_PORT:6379}
    password: ${REDIS_PASSWORD:}
    ssl:
      enabled: false
    lettuce:
      pool:
        max-active: 8
        max-idle: 8
        min-idle: 0
        max-wait: -1ms
```

### Java Configuration

```java
@Configuration
@EnableRedisHttpSession(
    maxInactiveIntervalInSeconds = 1800,
    redisNamespace = "myapp:session",
    flushMode = FlushMode.ON_SAVE,
    saveMode = SaveMode.ON_SET_ATTRIBUTE
)
public class RedisSessionConfig {

    @Bean
    public LettuceConnectionFactory connectionFactory() {
        RedisStandaloneConfiguration config = new RedisStandaloneConfiguration();
        config.setHostName("localhost");
        config.setPort(6379);
        return new LettuceConnectionFactory(config);
    }
}
```

## Redis Cluster Configuration

```java
@Configuration
@EnableRedisHttpSession
public class RedisClusterSessionConfig {

    @Bean
    public LettuceConnectionFactory connectionFactory() {
        RedisClusterConfiguration clusterConfig = new RedisClusterConfiguration();
        clusterConfig.clusterNode("redis1", 6379);
        clusterConfig.clusterNode("redis2", 6379);
        clusterConfig.clusterNode("redis3", 6379);

        LettuceClientConfiguration clientConfig = LettuceClientConfiguration.builder()
            .commandTimeout(Duration.ofSeconds(2))
            .build();

        return new LettuceConnectionFactory(clusterConfig, clientConfig);
    }
}
```

## Redis Sentinel Configuration

```java
@Bean
public LettuceConnectionFactory connectionFactory() {
    RedisSentinelConfiguration sentinelConfig = new RedisSentinelConfiguration()
        .master("mymaster")
        .sentinel("sentinel1", 26379)
        .sentinel("sentinel2", 26379)
        .sentinel("sentinel3", 26379);

    return new LettuceConnectionFactory(sentinelConfig);
}
```

## Custom Serialization

### JSON Serializer
```java
@Configuration
@EnableRedisHttpSession
public class RedisSessionConfig {

    @Bean
    public RedisSerializer<Object> springSessionDefaultRedisSerializer() {
        return new GenericJackson2JsonRedisSerializer(objectMapper());
    }

    private ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.registerModule(new JavaTimeModule());
        mapper.activateDefaultTyping(
            mapper.getPolymorphicTypeValidator(),
            ObjectMapper.DefaultTyping.NON_FINAL,
            JsonTypeInfo.As.PROPERTY
        );
        return mapper;
    }
}
```

## Session Data Structure in Redis

```
# Session hash
spring:session:sessions:<session-id>
  creationTime: 1703600000000
  lastAccessedTime: 1703603600000
  maxInactiveInterval: 1800
  sessionAttr:currentUser: {"id":1,"name":"John"}
  sessionAttr:cart: {"items":[...]}

# Expiration index
spring:session:expirations:<expiration-time>
  - session-id-1
  - session-id-2

# Principal index (if using FindByIndexNameSessionRepository)
spring:session:index:principal:<username>
  - session-id
```

## Session Lookup

```java
@Service
@RequiredArgsConstructor
public class SessionManagementService {

    private final FindByIndexNameSessionRepository<? extends Session> sessionRepository;

    public Map<String, ? extends Session> findByUsername(String username) {
        return sessionRepository.findByPrincipalName(username);
    }

    public void invalidateUserSessions(String username) {
        Map<String, ? extends Session> sessions = findByUsername(username);
        sessions.values().forEach(session -> {
            sessionRepository.deleteById(session.getId());
        });
    }

    public int getActiveSessionCount(String username) {
        return findByUsername(username).size();
    }
}
```

### Enable Principal Index
```java
@Configuration
@EnableRedisHttpSession
public class RedisSessionConfig {

    @Bean
    public RedisIndexedSessionRepository sessionRepository(
            RedisOperations<Object, Object> sessionRedisOperations) {
        return new RedisIndexedSessionRepository(sessionRedisOperations);
    }

    @Bean
    public HttpSessionIdResolver httpSessionIdResolver() {
        return HeaderHttpSessionIdResolver.xAuthToken();
    }
}
```

## Session Cookie Configuration

```java
@Bean
public CookieSerializer cookieSerializer() {
    DefaultCookieSerializer serializer = new DefaultCookieSerializer();
    serializer.setCookieName("SESSION");
    serializer.setCookiePath("/");
    serializer.setDomainNamePattern("^.+?\\.(\\w+\\.[a-z]+)$");
    serializer.setUseHttpOnlyCookie(true);
    serializer.setUseSecureCookie(true);
    serializer.setSameSite("Strict");
    serializer.setCookieMaxAge(1800);
    return serializer;
}
```

## Health Check

```yaml
management:
  health:
    redis:
      enabled: true
  endpoint:
    health:
      show-details: always
```

## Best Practices

| Do | Don't |
|----|-------|
| Use connection pooling | Create new connections per request |
| Configure proper timeouts | Use infinite timeouts |
| Use JSON serialization | Java serialization in production |
| Index by principal for lookups | Scan all sessions |
| Monitor Redis memory | Ignore memory usage |
| Configure session cleanup | Let sessions accumulate |
