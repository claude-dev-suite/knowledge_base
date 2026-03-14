# Spring Cloud Eureka - Client

## Basic Client Setup

### Dependencies
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

### Configuration
```yaml
spring:
  application:
    name: order-service

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
```

## Instance Configuration

### Basic Instance
```yaml
eureka:
  instance:
    # Instance identification
    instance-id: ${spring.application.name}:${server.port}:${random.uuid}
    app-name: order-service

    # Network configuration
    prefer-ip-address: true
    ip-address: ${HOST_IP:localhost}
    hostname: ${HOSTNAME:localhost}
    non-secure-port: ${server.port:8080}
    secure-port: 443
    secure-port-enabled: false

    # Heartbeat configuration
    lease-renewal-interval-in-seconds: 30
    lease-expiration-duration-in-seconds: 90

    # Health check
    health-check-url-path: /actuator/health
    status-page-url-path: /actuator/info
```

### Instance Metadata
```yaml
eureka:
  instance:
    metadata-map:
      version: "2.0.0"
      environment: production
      region: us-east-1
      zone: us-east-1a
      weight: "100"
      custom-property: custom-value
```

### Accessing Metadata
```java
@Service
@RequiredArgsConstructor
public class ServiceMetadataService {

    private final DiscoveryClient discoveryClient;

    public String getServiceVersion(String serviceId) {
        return discoveryClient.getInstances(serviceId).stream()
            .findFirst()
            .map(instance -> instance.getMetadata().get("version"))
            .orElse("unknown");
    }

    public List<ServiceInstance> getInstancesByRegion(String serviceId, String region) {
        return discoveryClient.getInstances(serviceId).stream()
            .filter(instance -> region.equals(instance.getMetadata().get("region")))
            .toList();
    }
}
```

## Service Discovery

### DiscoveryClient
```java
@RestController
@RequiredArgsConstructor
public class ServiceInfoController {

    private final DiscoveryClient discoveryClient;

    @GetMapping("/services")
    public List<String> getServices() {
        return discoveryClient.getServices();
    }

    @GetMapping("/services/{serviceId}/instances")
    public List<Map<String, Object>> getInstances(@PathVariable String serviceId) {
        return discoveryClient.getInstances(serviceId).stream()
            .map(instance -> Map.<String, Object>of(
                "instanceId", instance.getInstanceId(),
                "host", instance.getHost(),
                "port", instance.getPort(),
                "uri", instance.getUri().toString(),
                "metadata", instance.getMetadata()
            ))
            .toList();
    }
}
```

### EurekaClient (Netflix API)
```java
@Service
public class EurekaClientService {

    @Autowired
    private EurekaClient eurekaClient;

    public InstanceInfo getNextServerFromEureka(String serviceId) {
        return eurekaClient.getNextServerFromEureka(serviceId, false);
    }

    public Application getApplication(String serviceId) {
        return eurekaClient.getApplication(serviceId);
    }
}
```

## Load Balancing

### RestTemplate with Load Balancing
```java
@Configuration
public class LoadBalancedConfig {

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplateBuilder()
            .setConnectTimeout(Duration.ofSeconds(5))
            .setReadTimeout(Duration.ofSeconds(10))
            .build();
    }
}

@Service
@RequiredArgsConstructor
public class UserClient {

    private final RestTemplate restTemplate;

    public User getUser(Long id) {
        return restTemplate.getForObject(
            "http://user-service/api/users/{id}",
            User.class,
            id
        );
    }

    public List<User> getAllUsers() {
        return restTemplate.exchange(
            "http://user-service/api/users",
            HttpMethod.GET,
            null,
            new ParameterizedTypeReference<List<User>>() {}
        ).getBody();
    }
}
```

### WebClient with Load Balancing
```java
@Configuration
public class WebClientConfig {

    @Bean
    @LoadBalanced
    public WebClient.Builder loadBalancedWebClientBuilder() {
        return WebClient.builder();
    }
}

@Service
public class UserClient {

    private final WebClient webClient;

    public UserClient(@LoadBalanced WebClient.Builder builder) {
        this.webClient = builder
            .baseUrl("http://user-service")
            .build();
    }

    public Mono<User> getUser(Long id) {
        return webClient.get()
            .uri("/api/users/{id}", id)
            .retrieve()
            .bodyToMono(User.class);
    }

    public Flux<User> getAllUsers() {
        return webClient.get()
            .uri("/api/users")
            .retrieve()
            .bodyToFlux(User.class);
    }
}
```

## Health Checks

### Actuator Health
```yaml
management:
  endpoint:
    health:
      show-details: always
  health:
    eureka:
      enabled: true

eureka:
  client:
    healthcheck:
      enabled: true
```

### Custom Health Check
```java
@Component
public class DatabaseHealthCheckHandler implements HealthCheckHandler {

    private final DataSource dataSource;

    @Override
    public InstanceInfo.InstanceStatus getStatus(InstanceInfo.InstanceStatus currentStatus) {
        try (Connection conn = dataSource.getConnection()) {
            if (conn.isValid(5)) {
                return InstanceInfo.InstanceStatus.UP;
            }
        } catch (SQLException e) {
            return InstanceInfo.InstanceStatus.DOWN;
        }
        return InstanceInfo.InstanceStatus.UNKNOWN;
    }
}
```

## Zones and Regions

```yaml
eureka:
  client:
    prefer-same-zone-eureka: true
    region: us-east
    availability-zones:
      us-east: us-east-1a,us-east-1b
    service-url:
      us-east-1a: http://eureka-1a:8761/eureka/
      us-east-1b: http://eureka-1b:8761/eureka/
  instance:
    metadata-map:
      zone: us-east-1a
```

## Multiple Eureka Servers

```yaml
eureka:
  client:
    service-url:
      defaultZone: http://eureka1:8761/eureka/,http://eureka2:8762/eureka/,http://eureka3:8763/eureka/
    eureka-server-connect-timeout-seconds: 5
    eureka-server-read-timeout-seconds: 8
```

## Client Configuration Reference

```yaml
eureka:
  client:
    # Registration
    enabled: true
    register-with-eureka: true
    registry-fetch-interval-seconds: 30

    # Caching
    cache-refresh-executor-thread-pool-size: 2
    cache-refresh-executor-exponential-back-off-bound: 10
    disable-delta: false

    # Timeouts
    eureka-server-connect-timeout-seconds: 5
    eureka-server-read-timeout-seconds: 8

    # Health
    healthcheck:
      enabled: true

    # Service URL
    service-url:
      defaultZone: http://localhost:8761/eureka/

  instance:
    # Identity
    instance-id: ${spring.application.name}:${random.uuid}
    prefer-ip-address: true

    # Lease
    lease-renewal-interval-in-seconds: 30
    lease-expiration-duration-in-seconds: 90

    # Endpoints
    status-page-url-path: /actuator/info
    health-check-url-path: /actuator/health
```

## Graceful Shutdown

```java
@Component
public class EurekaGracefulShutdown implements DisposableBean {

    @Autowired
    private EurekaClient eurekaClient;

    @Override
    public void destroy() throws Exception {
        // Deregister from Eureka before shutdown
        eurekaClient.shutdown();
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use prefer-ip-address in containers | Rely on hostnames in Docker |
| Configure proper lease intervals | Use very short intervals |
| Enable health checks | Ignore health status |
| Use instance metadata | Hardcode instance information |
| Configure multiple Eureka servers | Single point of failure |
| Test failover scenarios | Assume registration always works |
