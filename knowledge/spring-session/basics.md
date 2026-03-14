# Spring Session - Basics

## Overview

Spring Session provides a transparent replacement for HttpSession, enabling session storage in Redis, JDBC, or other backends. This allows session sharing across multiple application instances.

## Dependencies

### Redis
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

### JDBC
```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-jdbc</artifactId>
</dependency>
```

## Configuration

### Redis Configuration
```yaml
spring:
  session:
    store-type: redis
    redis:
      namespace: spring:session
      flush-mode: on-save
    timeout: 30m

  redis:
    host: localhost
    port: 6379
```

### JDBC Configuration
```yaml
spring:
  session:
    store-type: jdbc
    jdbc:
      initialize-schema: always
      table-name: SPRING_SESSION
    timeout: 30m
```

## Enable Spring Session

```java
@Configuration
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)
public class SessionConfig {

    @Bean
    public LettuceConnectionFactory connectionFactory() {
        return new LettuceConnectionFactory();
    }
}
```

## Usage

### Storing Session Data
```java
@RestController
@RequestMapping("/api")
public class SessionController {

    @PostMapping("/login")
    public ResponseEntity<String> login(HttpSession session, @RequestBody LoginRequest request) {
        User user = authService.authenticate(request);

        session.setAttribute("currentUser", user);
        session.setAttribute("loginTime", Instant.now());

        return ResponseEntity.ok("Logged in");
    }

    @GetMapping("/profile")
    public ResponseEntity<User> getProfile(HttpSession session) {
        User user = (User) session.getAttribute("currentUser");

        if (user == null) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }

        return ResponseEntity.ok(user);
    }

    @PostMapping("/logout")
    public ResponseEntity<String> logout(HttpSession session) {
        session.invalidate();
        return ResponseEntity.ok("Logged out");
    }
}
```

### @SessionScope Bean
```java
@Component
@SessionScope
public class ShoppingCart {

    private List<CartItem> items = new ArrayList<>();

    public void addItem(CartItem item) {
        items.add(item);
    }

    public List<CartItem> getItems() {
        return Collections.unmodifiableList(items);
    }

    public void clear() {
        items.clear();
    }
}
```

## Session Events

```java
@Component
@Slf4j
public class SessionEventListener {

    @EventListener
    public void onSessionCreated(SessionCreatedEvent event) {
        log.info("Session created: {}", event.getSessionId());
    }

    @EventListener
    public void onSessionDeleted(SessionDeletedEvent event) {
        log.info("Session deleted: {}", event.getSessionId());
    }

    @EventListener
    public void onSessionExpired(SessionExpiredEvent event) {
        log.info("Session expired: {}", event.getSessionId());
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Store minimal data in session | Store large objects |
| Use serializable objects | Store non-serializable data |
| Configure appropriate timeout | Use very long sessions |
| Clean up sessions properly | Leave orphan sessions |
| Use session events for cleanup | Manual cleanup only |
| Secure session cookies | Use default cookie settings |

## Production Checklist

- [ ] Session store configured (Redis/JDBC)
- [ ] Session timeout appropriate
- [ ] Cookie security enabled (HttpOnly, Secure)
- [ ] Session events logged
- [ ] Cleanup strategy in place
- [ ] HA for session store
