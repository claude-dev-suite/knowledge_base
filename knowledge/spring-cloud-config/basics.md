# Spring Cloud Config - Basics

## Overview

Spring Cloud Config provides server-side and client-side support for externalized configuration in a distributed system. The Config Server provides a central place to manage external properties for applications across all environments.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Spring Cloud Config                                │
│                                                                         │
│  ┌────────────────┐    ┌──────────────────┐    ┌───────────────────┐   │
│  │  Config Server │◀───│  Git/Vault/JDBC  │    │  Config Client    │   │
│  │  (Central Hub) │    │  (Backend Store) │    │  (Microservice)   │   │
│  └───────┬────────┘    └──────────────────┘    └─────────┬─────────┘   │
│          │                                               │             │
│          └───────────────────────────────────────────────┘             │
│                         HTTP/HTTPS                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

## Config Server Dependencies

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

## Config Client Dependencies

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

## Git Backend Configuration

### Server Configuration
```yaml
server:
  port: 8888

spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/myorg/config-repo
          default-label: main
          search-paths: '{application}'
          username: ${GIT_USERNAME}
          password: ${GIT_PASSWORD}
          clone-on-start: true
          timeout: 10
```

### Repository Structure
```
config-repo/
├── application.yml           # Shared config for all apps
├── application-dev.yml       # Shared dev profile
├── application-prod.yml      # Shared prod profile
├── order-service.yml         # Order service default
├── order-service-dev.yml     # Order service dev
├── order-service-prod.yml    # Order service prod
└── user-service/
    ├── application.yml       # Search path config
    └── application-dev.yml
```

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

## Client Configuration

### bootstrap.yml (Client)
```yaml
spring:
  application:
    name: order-service
  profiles:
    active: dev
  cloud:
    config:
      uri: http://localhost:8888
      fail-fast: true
      retry:
        initial-interval: 1000
        max-attempts: 6
        max-interval: 2000
        multiplier: 1.1
```

### application.yml (Spring Boot 2.4+)
```yaml
spring:
  application:
    name: order-service
  config:
    import: optional:configserver:http://localhost:8888
  cloud:
    config:
      fail-fast: true
```

## Configuration Endpoints

```bash
# Get configuration
GET http://localhost:8888/{application}/{profile}
GET http://localhost:8888/{application}/{profile}/{label}

# Examples
GET http://localhost:8888/order-service/dev
GET http://localhost:8888/order-service/prod/main
```

## Property Encryption

### Setup
```yaml
spring:
  cloud:
    config:
      server:
        encrypt:
          enabled: true

encrypt:
  key: ${ENCRYPT_KEY}  # Symmetric key
```

### Encrypt/Decrypt Endpoints
```bash
# Encrypt a value
curl -X POST http://localhost:8888/encrypt -d "mysecret"
# Returns: AQB5...encrypted...value

# Decrypt a value
curl -X POST http://localhost:8888/decrypt -d "AQB5...encrypted...value"
```

### Store Encrypted Values
```yaml
# In Git repository
spring:
  datasource:
    password: '{cipher}AQB5...encrypted...value'
```

## Refresh Configuration

### Manual Refresh (Single Service)
```bash
POST http://localhost:8080/actuator/refresh
```

### Bus Refresh (All Services)
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

```bash
POST http://localhost:8888/actuator/busrefresh
```

### @RefreshScope
```java
@RestController
@RefreshScope
public class MyController {

    @Value("${my.config.value}")
    private String configValue;  // Refreshed on /actuator/refresh

    @GetMapping("/config")
    public String getConfig() {
        return configValue;
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use encryption for secrets | Store secrets in plain text |
| Configure fail-fast for critical services | Silently fail on config errors |
| Use Git for versioning | Use file system without versioning |
| Separate configs per environment | Mix environment configs |
| Enable retry for resilience | Single retry attempt |
| Use service discovery for config server | Hardcode config server URL |

## Production Checklist

- [ ] TLS/HTTPS enabled
- [ ] Secrets encrypted
- [ ] Git authentication configured
- [ ] Retry and fail-fast configured
- [ ] Health check endpoints enabled
- [ ] Config server highly available
- [ ] Bus refresh configured
- [ ] Audit logging enabled
