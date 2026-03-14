# Spring Cloud Eureka - Server

## Standalone Server

```yaml
server:
  port: 8761

spring:
  application:
    name: eureka-server

eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

## High Availability Cluster

### Peer-Aware Configuration

```yaml
# eureka-server-1 (application-peer1.yml)
server:
  port: 8761

spring:
  application:
    name: eureka-server

eureka:
  instance:
    hostname: eureka1.example.com
    prefer-ip-address: false
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://eureka2.example.com:8762/eureka/,http://eureka3.example.com:8763/eureka/

---
# eureka-server-2 (application-peer2.yml)
server:
  port: 8762

eureka:
  instance:
    hostname: eureka2.example.com
  client:
    service-url:
      defaultZone: http://eureka1.example.com:8761/eureka/,http://eureka3.example.com:8763/eureka/

---
# eureka-server-3 (application-peer3.yml)
server:
  port: 8763

eureka:
  instance:
    hostname: eureka3.example.com
  client:
    service-url:
      defaultZone: http://eureka1.example.com:8761/eureka/,http://eureka2.example.com:8762/eureka/
```

### Docker Compose for HA
```yaml
version: '3.8'
services:
  eureka1:
    image: eureka-server:latest
    environment:
      - SPRING_PROFILES_ACTIVE=peer1
      - EUREKA_INSTANCE_HOSTNAME=eureka1
      - EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://eureka2:8761/eureka/,http://eureka3:8761/eureka/
    ports:
      - "8761:8761"

  eureka2:
    image: eureka-server:latest
    environment:
      - SPRING_PROFILES_ACTIVE=peer2
      - EUREKA_INSTANCE_HOSTNAME=eureka2
      - EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://eureka1:8761/eureka/,http://eureka3:8761/eureka/
    ports:
      - "8762:8761"

  eureka3:
    image: eureka-server:latest
    environment:
      - SPRING_PROFILES_ACTIVE=peer3
      - EUREKA_INSTANCE_HOSTNAME=eureka3
      - EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://eureka1:8761/eureka/,http://eureka2:8761/eureka/
    ports:
      - "8763:8761"
```

## Security

### Basic Authentication
```java
@Configuration
@EnableWebSecurity
public class EurekaSecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.ignoringRequestMatchers("/eureka/**"))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health").permitAll()
                .anyRequest().authenticated()
            )
            .httpBasic(Customizer.withDefaults());
        return http.build();
    }

    @Bean
    public UserDetailsService userDetailsService() {
        UserDetails user = User.builder()
            .username("eureka")
            .password("{bcrypt}$2a$10$...")
            .roles("ADMIN")
            .build();
        return new InMemoryUserDetailsManager(user);
    }
}
```

### Client with Authentication
```yaml
eureka:
  client:
    service-url:
      defaultZone: http://eureka:password@localhost:8761/eureka/
```

## Self-Preservation Configuration

```yaml
eureka:
  server:
    # Enable self-preservation (recommended for production)
    enable-self-preservation: true

    # Percentage of renewals required before preservation kicks in
    renewal-percent-threshold: 0.85

    # How often to update the threshold
    renewal-threshold-update-interval-ms: 900000

    # How often to evict expired instances
    eviction-interval-timer-in-ms: 60000

    # Number of renewals per minute expected
    expected-client-renewal-interval-seconds: 30
```

## Rate Limiting

```yaml
eureka:
  server:
    rate-limiter-enabled: true
    rate-limiter-burst-size: 10
    rate-limiter-registry-fetch-average-rate: 500
    rate-limiter-throttle-standard-clients: false
    rate-limiter-full-fetch-average-rate: 100
```

## Response Cache

```yaml
eureka:
  server:
    response-cache-auto-expiration-in-seconds: 180
    response-cache-update-interval-ms: 30000
    use-read-only-response-cache: true
```

## Dashboard Customization

```yaml
eureka:
  dashboard:
    enabled: true
    path: /dashboard

# Access at http://localhost:8761/dashboard
```

## Metrics and Monitoring

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  endpoint:
    health:
      show-details: always
  metrics:
    tags:
      application: ${spring.application.name}
```

### Key Metrics
- `eureka.server.registry.lastMinute` - Registrations in last minute
- `eureka.server.renewsLastMin` - Renewals in last minute
- `eureka.server.canceledLastMin` - Cancellations in last minute

## Server Configuration Reference

```yaml
eureka:
  server:
    # Self-preservation
    enable-self-preservation: true
    renewal-percent-threshold: 0.85
    renewal-threshold-update-interval-ms: 900000

    # Eviction
    eviction-interval-timer-in-ms: 60000

    # Response caching
    response-cache-auto-expiration-in-seconds: 180
    response-cache-update-interval-ms: 30000
    use-read-only-response-cache: true

    # Peer synchronization
    peer-eureka-nodes-update-interval-ms: 600000
    peer-node-read-timeout-ms: 200
    peer-node-connect-timeout-ms: 200

    # Rate limiting
    rate-limiter-enabled: true
    rate-limiter-burst-size: 10

    # Replication
    max-threads-for-peer-replication: 20
    min-threads-for-peer-replication: 5
    number-of-replication-retries: 5
    peer-node-total-connections: 1000
    peer-node-total-connections-per-host: 500

  instance:
    hostname: ${HOSTNAME:localhost}
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${random.value}

  client:
    register-with-eureka: true  # false for standalone
    fetch-registry: true        # false for standalone
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

## Best Practices

| Do | Don't |
|----|-------|
| Deploy odd number of peers (3, 5) | Even number causes split-brain |
| Enable self-preservation in prod | Disable in production |
| Use dedicated servers for Eureka | Co-locate with application services |
| Configure proper timeouts | Use defaults for HA |
| Monitor registry size | Ignore capacity planning |
| Secure dashboard and endpoints | Leave unsecured in prod |
