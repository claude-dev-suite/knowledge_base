# Spring Cloud Eureka - Basics

## Overview

Spring Cloud Netflix Eureka provides service discovery for microservices. The Eureka Server maintains a registry of services, and clients register themselves and discover other services.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       Service Discovery                                  │
│                                                                         │
│  ┌─────────────────┐         ┌─────────────────┐                       │
│  │  Eureka Server  │◀───────▶│  Eureka Server  │  (HA Cluster)         │
│  │  (Registry)     │         │  (Peer)         │                       │
│  └────────┬────────┘         └─────────────────┘                       │
│           │                                                             │
│           │  Register / Heartbeat / Fetch                              │
│           │                                                             │
│  ┌────────▼────────┐    ┌────────────────┐    ┌────────────────┐       │
│  │  Order Service  │───▶│  User Service  │◀───│  Gateway       │       │
│  │  (Client)       │    │  (Client)      │    │  (Client)      │       │
│  └─────────────────┘    └────────────────┘    └────────────────┘       │
└─────────────────────────────────────────────────────────────────────────┘
```

## Eureka Server

### Dependencies
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

### Enable Server
```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

### Server Configuration
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
    register-with-eureka: false  # Server doesn't register itself
    fetch-registry: false        # Server doesn't need to fetch
    service-url:
      defaultZone: http://localhost:8761/eureka/
  server:
    enable-self-preservation: true
    eviction-interval-timer-in-ms: 60000
```

## Eureka Client

### Dependencies
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

### Client Configuration
```yaml
spring:
  application:
    name: order-service

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
    registry-fetch-interval-seconds: 30
    instance-info-replication-interval-seconds: 30
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 30
    lease-expiration-duration-in-seconds: 90
```

## Service Discovery

### Using DiscoveryClient
```java
@Service
@RequiredArgsConstructor
public class ServiceDiscoveryService {

    private final DiscoveryClient discoveryClient;

    public List<String> getServiceInstances(String serviceId) {
        return discoveryClient.getInstances(serviceId).stream()
            .map(instance -> instance.getHost() + ":" + instance.getPort())
            .toList();
    }

    public List<String> getServices() {
        return discoveryClient.getServices();
    }
}
```

### Load Balanced RestTemplate
```java
@Configuration
public class RestTemplateConfig {

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

@Service
public class UserServiceClient {

    private final RestTemplate restTemplate;

    public User getUser(Long id) {
        // Uses service name instead of host:port
        return restTemplate.getForObject(
            "http://user-service/api/users/{id}",
            User.class,
            id
        );
    }
}
```

### WebClient with Load Balancing
```java
@Configuration
public class WebClientConfig {

    @Bean
    @LoadBalanced
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder();
    }
}

@Service
public class UserServiceClient {

    private final WebClient webClient;

    public UserServiceClient(WebClient.Builder builder) {
        this.webClient = builder.baseUrl("http://user-service").build();
    }

    public Mono<User> getUser(Long id) {
        return webClient.get()
            .uri("/api/users/{id}", id)
            .retrieve()
            .bodyToMono(User.class);
    }
}
```

## Health and Status

### Instance Metadata
```yaml
eureka:
  instance:
    metadata-map:
      version: "1.0.0"
      environment: production
      region: us-east-1
```

### Custom Health Check
```java
@Component
public class CustomHealthCheckHandler implements HealthCheckHandler {

    private final CustomHealthIndicator healthIndicator;

    @Override
    public InstanceInfo.InstanceStatus getStatus(InstanceInfo.InstanceStatus currentStatus) {
        if (healthIndicator.isHealthy()) {
            return InstanceInfo.InstanceStatus.UP;
        }
        return InstanceInfo.InstanceStatus.DOWN;
    }
}
```

## Self-Preservation Mode

Eureka enters self-preservation when it stops receiving heartbeats from a significant portion of instances:

```yaml
eureka:
  server:
    enable-self-preservation: true
    renewal-percent-threshold: 0.85
    renewal-threshold-update-interval-ms: 900000
```

## Best Practices

| Do | Don't |
|----|-------|
| Use HA setup in production | Single Eureka server |
| Configure proper lease times | Use default values blindly |
| Use prefer-ip-address | Rely on hostnames |
| Monitor Eureka dashboard | Ignore registration status |
| Configure self-preservation | Disable self-preservation in prod |
| Use health checks | Assume services are always up |

## Production Checklist

- [ ] HA cluster configured
- [ ] Proper lease intervals set
- [ ] Self-preservation enabled
- [ ] Health checks configured
- [ ] Security enabled
- [ ] Monitoring and alerting set up
- [ ] DNS or load balancer for servers
