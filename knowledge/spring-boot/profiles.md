# Spring Profiles Reference

## Activating Profiles

```yaml
# application.yml
spring:
  profiles:
    active: dev

# Or via environment variable
SPRING_PROFILES_ACTIVE=prod

# Or via command line
java -jar app.jar --spring.profiles.active=prod

# Multiple profiles
spring:
  profiles:
    active: dev,metrics,swagger
```

## Profile-Specific Configuration

```yaml
# application.yml (default)
server:
  port: 8080

spring:
  datasource:
    url: jdbc:h2:mem:testdb

---
# application-dev.yml
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:postgresql://localhost:5432/myapp_dev
    username: dev_user
    password: dev_pass

logging:
  level:
    com.example: DEBUG

---
# application-prod.yml
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: ${DATABASE_URL}
    username: ${DATABASE_USER}
    password: ${DATABASE_PASSWORD}

logging:
  level:
    root: WARN
    com.example: INFO
```

## Profile-Specific Beans

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @Profile("dev")
    public DataSource devDataSource() {
        return DataSourceBuilder.create()
            .url("jdbc:h2:mem:devdb")
            .driverClassName("org.h2.Driver")
            .build();
    }

    @Bean
    @Profile("prod")
    public DataSource prodDataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl(System.getenv("DATABASE_URL"));
        ds.setMaximumPoolSize(20);
        return ds;
    }
}
```

## Negated Profiles

```java
@Component
@Profile("!prod")  // Active in all profiles except prod
public class DevOnlyComponent { }

@Component
@Profile("!prod & !staging")  // Not in prod AND not in staging
public class LocalOnlyComponent { }
```

## Profile Groups

```yaml
# application.yml
spring:
  profiles:
    group:
      production:
        - prod
        - metrics
        - security
      development:
        - dev
        - swagger
        - h2console

# Activating 'production' activates: prod, metrics, security
```

## Conditional Configuration

```java
@Configuration
@Profile("swagger")
public class SwaggerConfig {
    @Bean
    public OpenAPI openAPI() {
        return new OpenAPI()
            .info(new Info().title("API").version("1.0"));
    }
}

@Configuration
@Profile("metrics")
public class MetricsConfig {
    @Bean
    public MeterRegistry meterRegistry() {
        return new SimpleMeterRegistry();
    }
}
```

## Programmatic Profile Check

```java
@Service
@RequiredArgsConstructor
public class NotificationService {

    private final Environment environment;

    public void sendNotification(String message) {
        if (environment.acceptsProfiles(Profiles.of("prod"))) {
            // Send real notification
            emailService.send(message);
        } else {
            // Log only in dev
            log.info("Would send notification: {}", message);
        }
    }
}
```

## Profile-Specific Properties Files

```
src/main/resources/
├── application.yml           # Default
├── application-dev.yml       # Development
├── application-test.yml      # Testing
├── application-staging.yml   # Staging
├── application-prod.yml      # Production
└── application-local.yml     # Local overrides
```

## Default Profile

```yaml
# application.yml
spring:
  profiles:
    default: dev  # Used when no profile is explicitly set
```

## Include Additional Profiles

```yaml
# application-dev.yml
spring:
  profiles:
    include:
      - swagger
      - h2console
```

## Testing with Profiles

```java
@SpringBootTest
@ActiveProfiles("test")
class UserServiceTest {
    // Uses application-test.yml configuration
}

@DataJpaTest
@ActiveProfiles({"test", "h2"})
class UserRepositoryTest {
    // Uses test + h2 profiles
}
```

## Common Profile Patterns

| Profile | Purpose |
|---------|---------|
| dev | Local development |
| test | Unit/integration tests |
| staging | Pre-production |
| prod | Production |
| swagger | Enable API docs |
| metrics | Enable monitoring |
| debug | Extra logging |
| docker | Container config |
