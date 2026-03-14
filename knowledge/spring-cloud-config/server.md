# Spring Cloud Config - Server

## Server Setup

### Enable Config Server
```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

## Backend Types

### Git Backend
```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/org/config-repo
          default-label: main
          search-paths:
            - '{application}'
            - 'shared'
          username: ${GIT_USERNAME}
          password: ${GIT_PASSWORD}
          clone-on-start: true
          force-pull: true
          timeout: 10
```

### Multiple Git Repositories
```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/org/default-config
          repos:
            special-service:
              pattern: special-*
              uri: https://github.com/org/special-config
            team-a:
              pattern: team-a-*
              uri: https://github.com/org/team-a-config
              clone-on-start: true
```

### Native (File System) Backend
```yaml
spring:
  profiles:
    active: native
  cloud:
    config:
      server:
        native:
          search-locations: file:///config-repo, classpath:/config
```

### JDBC Backend
```yaml
spring:
  profiles:
    active: jdbc
  cloud:
    config:
      server:
        jdbc:
          sql: SELECT KEY, VALUE from PROPERTIES where APPLICATION=? and PROFILE=? and LABEL=?

  datasource:
    url: jdbc:postgresql://localhost/config_db
    username: ${DB_USER}
    password: ${DB_PASSWORD}
```

Schema:
```sql
CREATE TABLE PROPERTIES (
    ID SERIAL PRIMARY KEY,
    APPLICATION VARCHAR(128) NOT NULL,
    PROFILE VARCHAR(128) NOT NULL,
    LABEL VARCHAR(128) NOT NULL,
    KEY VARCHAR(128) NOT NULL,
    VALUE VARCHAR(1024)
);
```

### Vault Backend
```yaml
spring:
  profiles:
    active: vault
  cloud:
    config:
      server:
        vault:
          host: localhost
          port: 8200
          scheme: https
          authentication: TOKEN
          token: ${VAULT_TOKEN}
          kv-version: 2
          default-key: application
          backend: secret
```

### Composite Backend
```yaml
spring:
  profiles:
    active: git, vault
  cloud:
    config:
      server:
        composite:
          - type: git
            uri: https://github.com/org/config-repo
          - type: vault
            host: localhost
            port: 8200
```

## Security

### Basic Authentication
```yaml
spring:
  security:
    user:
      name: configuser
      password: ${CONFIG_SERVER_PASSWORD}
```

### OAuth2
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt());
        return http.build();
    }
}
```

## Encryption

### Symmetric Key
```yaml
encrypt:
  key: ${ENCRYPT_KEY}
```

### Asymmetric Key (RSA)
```yaml
encrypt:
  key-store:
    location: classpath:/keystore.jks
    password: ${KEYSTORE_PASSWORD}
    alias: configkey
    secret: ${KEY_PASSWORD}
```

### Generate Keystore
```bash
keytool -genkeypair -alias configkey -keyalg RSA \
  -keystore keystore.jks -storepass changeit \
  -keypass changeit -dname "CN=Config Server"
```

### Encrypt/Decrypt Values
```java
@RestController
public class EncryptionController {

    @Autowired
    private TextEncryptor encryptor;

    @PostMapping("/encrypt-value")
    public String encrypt(@RequestBody String value) {
        return encryptor.encrypt(value);
    }

    @PostMapping("/decrypt-value")
    public String decrypt(@RequestBody String value) {
        return encryptor.decrypt(value);
    }
}
```

## Health Monitoring

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, env
  endpoint:
    health:
      show-details: always
```

### Custom Health Indicator
```java
@Component
public class GitRepositoryHealthIndicator implements HealthIndicator {

    @Autowired
    private EnvironmentRepository environmentRepository;

    @Override
    public Health health() {
        try {
            environmentRepository.findOne("application", "default", null);
            return Health.up()
                .withDetail("repository", "available")
                .build();
        } catch (Exception e) {
            return Health.down()
                .withException(e)
                .build();
        }
    }
}
```

## High Availability

### Multiple Config Servers
```yaml
# Client configuration
spring:
  cloud:
    config:
      uri:
        - http://config-server-1:8888
        - http://config-server-2:8888
      fail-fast: true
      retry:
        max-attempts: 6
```

### Behind Load Balancer
```yaml
# Client with discovery
spring:
  cloud:
    config:
      discovery:
        enabled: true
        service-id: config-server
      fail-fast: true
```

## Webhooks (Auto Refresh)

### GitHub Webhook
```java
@RestController
public class WebhookController {

    @Autowired
    private ApplicationEventPublisher publisher;

    @PostMapping("/webhook")
    public void handleGithubWebhook(@RequestBody String payload) {
        // Validate webhook signature
        // Trigger refresh
        publisher.publishEvent(new RefreshRemoteApplicationEvent(
            this, "config-server", "*:**"));
    }
}
```

## Server Configuration Reference

```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/org/config
          default-label: main
          search-paths: '{application}'
          clone-on-start: true
          force-pull: true
          timeout: 10
          skip-ssl-validation: false
          delete-untracked-branches: true
          refresh-rate: 0  # 0 = fetch every request

        # Override bootstrap
        bootstrap: false

        # Accept specific patterns
        accept-empty: true

        # Prefix for all properties
        prefix: /config

encrypt:
  fail-on-error: true

management:
  endpoints:
    web:
      exposure:
        include: health, refresh, busrefresh
```

## Best Practices

| Do | Don't |
|----|-------|
| Use dedicated config repos | Mix config with application code |
| Encrypt sensitive values | Store secrets in plain text |
| Configure HA for production | Single point of failure |
| Enable webhooks for auto-refresh | Manual refresh only |
| Use Vault for secrets | Store credentials in Git |
| Version config changes | Lose configuration history |
