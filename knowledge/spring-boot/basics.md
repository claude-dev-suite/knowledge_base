# Spring Boot 3 Basics - Comprehensive Guide

This comprehensive guide covers Spring Boot 3.x fundamentals based on official Spring Boot documentation.
It provides patterns, best practices, and practical examples for building production-ready applications.

---

## Table of Contents

1. [Application Setup](#1-application-setup)
2. [Configuration Properties](#2-configuration-properties)
3. [@ConfigurationProperties](#3-configurationproperties)
4. [Profiles](#4-profiles)
5. [REST Controllers](#5-rest-controllers)
6. [Request/Response Handling](#6-requestresponse-handling)
7. [Service Layer](#7-service-layer)
8. [Repository Layer](#8-repository-layer)
9. [Dependency Injection](#9-dependency-injection)
10. [Exception Handling](#10-exception-handling)
11. [Validation](#11-validation)
12. [Logging](#12-logging)
13. [Actuator Endpoints](#13-actuator-endpoints)
14. [Testing](#14-testing)
15. [Maven/Gradle Configuration](#15-mavengradle-configuration)
16. [Best Practices](#16-best-practices)

---

## 1. Application Setup

### @SpringBootApplication

The `@SpringBootApplication` annotation is a convenience annotation that combines three annotations:

- `@Configuration`: Tags the class as a source of bean definitions
- `@EnableAutoConfiguration`: Enables Spring Boot's auto-configuration mechanism
- `@ComponentScan`: Enables component scanning in the package and sub-packages

```java
package com.example.myapp;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

### Customizing SpringApplication

```java
@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(MyApplication.class);

        // Customize banner
        app.setBannerMode(Banner.Mode.OFF);

        // Add initializers
        app.addInitializers(new MyApplicationContextInitializer());

        // Set additional profiles
        app.setAdditionalProfiles("custom");

        // Disable lazy initialization
        app.setLazyInitialization(false);

        app.run(args);
    }
}
```

### Using SpringApplicationBuilder (Fluent API)

```java
@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        new SpringApplicationBuilder(MyApplication.class)
            .bannerMode(Banner.Mode.CONSOLE)
            .lazyInitialization(true)
            .profiles("production")
            .run(args);
    }
}
```

### Application Events and Listeners

```java
@Component
public class ApplicationStartupListener {

    @EventListener(ApplicationReadyEvent.class)
    public void onApplicationReady() {
        System.out.println("Application is ready to serve requests!");
    }

    @EventListener(ContextRefreshedEvent.class)
    public void onContextRefreshed(ContextRefreshedEvent event) {
        System.out.println("Context refreshed");
    }
}
```

### CommandLineRunner and ApplicationRunner

```java
@Component
@Order(1)
public class DataInitializer implements CommandLineRunner {

    @Override
    public void run(String... args) throws Exception {
        // Initialize data on startup
        System.out.println("CommandLineRunner executed with args: " + Arrays.toString(args));
    }
}

@Component
@Order(2)
public class ApplicationInitializer implements ApplicationRunner {

    @Override
    public void run(ApplicationArguments args) throws Exception {
        // Access to parsed arguments
        System.out.println("Non-option args: " + args.getNonOptionArgs());
        System.out.println("Option names: " + args.getOptionNames());
    }
}
```

### Enabling Additional Features

```java
@SpringBootApplication
@EnableJpaAuditing                    // Enable JPA auditing
@EnableAsync                          // Enable async processing
@EnableScheduling                     // Enable scheduled tasks
@EnableCaching                        // Enable caching
@EnableTransactionManagement          // Enable transaction management
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

---

## 2. Configuration Properties

### application.properties Format

```properties
# Server Configuration
server.port=8080
server.servlet.context-path=/api

# Database Configuration
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
spring.datasource.username=user
spring.datasource.password=secret
spring.datasource.driver-class-name=org.postgresql.Driver

# JPA Configuration
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.open-in-view=false

# Logging
logging.level.root=INFO
logging.level.com.example=DEBUG
logging.file.name=app.log
```

### application.yml Format (Recommended)

```yaml
spring:
  application:
    name: my-spring-boot-app

  # DataSource Configuration
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USER:defaultuser}
    password: ${DB_PASSWORD:defaultpass}
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
      idle-timeout: 300000
      connection-timeout: 20000
      max-lifetime: 1200000
      pool-name: MyHikariPool

  # JPA Configuration
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.PostgreSQLDialect
        jdbc:
          batch_size: 25
        order_inserts: true
        order_updates: true
    open-in-view: false

  # Flyway Migration
  flyway:
    enabled: true
    baseline-on-migrate: true
    locations: classpath:db/migration

  # Jackson Configuration
  jackson:
    serialization:
      write-dates-as-timestamps: false
      indent-output: true
    deserialization:
      fail-on-unknown-properties: false
    default-property-inclusion: non_null
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: UTC

# Server Configuration
server:
  port: 8080
  servlet:
    context-path: /api/v1
  compression:
    enabled: true
    mime-types: application/json,application/xml,text/html
    min-response-size: 1024
  error:
    include-message: always
    include-binding-errors: always
    include-stacktrace: on_param
    include-exception: false

# Logging Configuration
logging:
  level:
    root: INFO
    com.example: DEBUG
    org.springframework.web: INFO
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
  file:
    name: logs/application.log
    max-size: 10MB
    max-history: 30
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
    file: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"

# Spring Doc / OpenAPI
springdoc:
  api-docs:
    path: /v3/api-docs
    enabled: true
  swagger-ui:
    path: /swagger-ui.html
    enabled: true
    operations-sorter: method
    tags-sorter: alpha
```

### Environment Variable Substitution

```yaml
# Using environment variables with defaults
spring:
  datasource:
    url: ${DATABASE_URL:jdbc:postgresql://localhost:5432/mydb}
    username: ${DB_USER:defaultuser}
    password: ${DB_PASSWORD:defaultpass}

# Using system properties
server:
  port: ${SERVER_PORT:8080}

# Complex expressions with SpEL
app:
  feature:
    enabled: ${FEATURE_ENABLED:#{true}}
    timeout: ${TIMEOUT:#{30 * 1000}}
```

### Property Placeholders

```yaml
# Reference other properties
app:
  name: MyApp
  description: ${app.name} is a Spring Boot application
  full-description: "${app.name} v${app.version:1.0.0}"
```

---

## 3. @ConfigurationProperties

### Basic Type-Safe Configuration

```java
@ConfigurationProperties(prefix = "app")
@Validated
public class AppProperties {

    @NotBlank
    private String name;

    @NotBlank
    private String description;

    @Min(1)
    @Max(65535)
    private int port = 8080;

    private boolean featureEnabled = true;

    @Valid
    private Security security = new Security();

    @Valid
    private List<Server> servers = new ArrayList<>();

    // Getters and setters
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }

    public int getPort() { return port; }
    public void setPort(int port) { this.port = port; }

    public boolean isFeatureEnabled() { return featureEnabled; }
    public void setFeatureEnabled(boolean featureEnabled) { this.featureEnabled = featureEnabled; }

    public Security getSecurity() { return security; }
    public void setSecurity(Security security) { this.security = security; }

    public List<Server> getServers() { return servers; }
    public void setServers(List<Server> servers) { this.servers = servers; }

    // Nested configuration class
    public static class Security {
        private boolean enabled = true;

        @NotBlank
        private String secretKey;

        @DurationUnit(ChronoUnit.SECONDS)
        private Duration tokenExpiration = Duration.ofHours(1);

        // Getters and setters
        public boolean isEnabled() { return enabled; }
        public void setEnabled(boolean enabled) { this.enabled = enabled; }

        public String getSecretKey() { return secretKey; }
        public void setSecretKey(String secretKey) { this.secretKey = secretKey; }

        public Duration getTokenExpiration() { return tokenExpiration; }
        public void setTokenExpiration(Duration tokenExpiration) { this.tokenExpiration = tokenExpiration; }
    }

    public static class Server {
        private String host;
        private int port;

        // Getters and setters
        public String getHost() { return host; }
        public void setHost(String host) { this.host = host; }

        public int getPort() { return port; }
        public void setPort(int port) { this.port = port; }
    }
}
```

### Enabling Configuration Properties

```java
@SpringBootApplication
@EnableConfigurationProperties(AppProperties.class)
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

// Or using @ConfigurationPropertiesScan
@SpringBootApplication
@ConfigurationPropertiesScan("com.example.config")
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### Corresponding YAML Configuration

```yaml
app:
  name: MyApplication
  description: A sample Spring Boot application
  port: 8080
  feature-enabled: true
  security:
    enabled: true
    secret-key: mySecretKey123
    token-expiration: 3600s
  servers:
    - host: server1.example.com
      port: 8081
    - host: server2.example.com
      port: 8082
```

### Constructor Binding (Immutable Configuration)

```java
@ConfigurationProperties(prefix = "mail")
public record MailProperties(
    @DefaultValue("localhost") String host,
    @DefaultValue("25") int port,
    @DefaultValue("noreply@example.com") String from,
    Security security
) {
    public record Security(
        @DefaultValue("false") boolean enabled,
        String username,
        String password
    ) {}
}
```

### Using @Value for Simple Properties

```java
@Component
public class SimpleConfig {

    @Value("${app.name}")
    private String appName;

    @Value("${app.feature-enabled:true}")
    private boolean featureEnabled;

    @Value("${app.timeout:30}000")
    private long timeoutMs;

    @Value("#{'${app.allowed-origins}'.split(',')}")
    private List<String> allowedOrigins;

    @Value("#{${app.settings}}")
    private Map<String, String> settings;
}
```

---

## 4. Profiles

### Profile-Specific Configuration Files

Create separate files for each profile:
- `application.yml` - Default configuration
- `application-dev.yml` - Development profile
- `application-prod.yml` - Production profile
- `application-test.yml` - Test profile

```yaml
# application.yml (default)
spring:
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}

app:
  name: MyApp
  logging-enabled: true

---
# application-dev.yml
spring:
  config:
    activate:
      on-profile: dev

  datasource:
    url: jdbc:h2:mem:devdb
    driver-class-name: org.h2.Driver

  h2:
    console:
      enabled: true

  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true

logging:
  level:
    com.example: DEBUG
    org.hibernate.SQL: DEBUG

---
# application-prod.yml
spring:
  config:
    activate:
      on-profile: prod

  datasource:
    url: ${DATABASE_URL}
    username: ${DB_USER}
    password: ${DB_PASSWORD}

  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false

logging:
  level:
    root: WARN
    com.example: INFO
```

### Activating Profiles

```bash
# Command line argument
java -jar myapp.jar --spring.profiles.active=prod

# Environment variable
export SPRING_PROFILES_ACTIVE=prod

# JVM system property
java -Dspring.profiles.active=prod -jar myapp.jar

# In application.yml
spring:
  profiles:
    active: dev
```

### Profile-Specific Beans

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @Profile("dev")
    public DataSource devDataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .addScript("classpath:schema-dev.sql")
            .build();
    }

    @Bean
    @Profile("prod")
    public DataSource prodDataSource() {
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setJdbcUrl(System.getenv("DATABASE_URL"));
        dataSource.setUsername(System.getenv("DB_USER"));
        dataSource.setPassword(System.getenv("DB_PASSWORD"));
        return dataSource;
    }

    @Bean
    @Profile("!prod")  // Active for all profiles except prod
    public DataSource nonProdDataSource() {
        // Development/test data source
        return DataSourceBuilder.create()
            .url("jdbc:h2:mem:testdb")
            .build();
    }
}
```

### Profile Groups

```yaml
spring:
  profiles:
    group:
      production:
        - prod
        - proddb
        - prodmq
      development:
        - dev
        - devdb
        - devmq
```

### Conditional Beans with Profiles

```java
@Service
@Profile("dev")
public class MockEmailService implements EmailService {
    @Override
    public void send(String to, String subject, String body) {
        log.info("Mock email sent to {} with subject: {}", to, subject);
    }
}

@Service
@Profile("prod")
public class SmtpEmailService implements EmailService {
    @Override
    public void send(String to, String subject, String body) {
        // Real SMTP implementation
    }
}
```

---

## 5. REST Controllers

### Basic @RestController

```java
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
@Tag(name = "User Management", description = "APIs for managing users")
public class UserController {

    private final UserService userService;

    @GetMapping
    @Operation(summary = "Get all users", description = "Returns a paginated list of users")
    public ResponseEntity<Page<UserResponse>> findAll(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size,
            @RequestParam(defaultValue = "id") String sortBy,
            @RequestParam(defaultValue = "asc") String sortDir) {

        Sort sort = sortDir.equalsIgnoreCase("desc")
            ? Sort.by(sortBy).descending()
            : Sort.by(sortBy).ascending();

        Pageable pageable = PageRequest.of(page, size, sort);
        return ResponseEntity.ok(userService.findAll(pageable));
    }

    @GetMapping("/{id}")
    @Operation(summary = "Get user by ID")
    public ResponseEntity<UserResponse> findById(
            @PathVariable @Positive Long id) {
        return ResponseEntity.ok(userService.findById(id));
    }

    @PostMapping
    @Operation(summary = "Create a new user")
    public ResponseEntity<UserResponse> create(
            @Valid @RequestBody CreateUserRequest request) {
        UserResponse created = userService.create(request);
        URI location = ServletUriComponentsBuilder
            .fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(created.getId())
            .toUri();
        return ResponseEntity.created(location).body(created);
    }

    @PutMapping("/{id}")
    @Operation(summary = "Update an existing user")
    public ResponseEntity<UserResponse> update(
            @PathVariable @Positive Long id,
            @Valid @RequestBody UpdateUserRequest request) {
        return ResponseEntity.ok(userService.update(id, request));
    }

    @PatchMapping("/{id}")
    @Operation(summary = "Partially update a user")
    public ResponseEntity<UserResponse> partialUpdate(
            @PathVariable @Positive Long id,
            @RequestBody Map<String, Object> updates) {
        return ResponseEntity.ok(userService.partialUpdate(id, updates));
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    @Operation(summary = "Delete a user")
    public ResponseEntity<Void> delete(@PathVariable @Positive Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

### Request Mapping Variants

```java
@RestController
@RequestMapping("/api/v1")
public class ResourceController {

    // HTTP Methods
    @GetMapping("/resources")
    public List<Resource> getAll() { ... }

    @PostMapping("/resources")
    public Resource create(@RequestBody Resource resource) { ... }

    @PutMapping("/resources/{id}")
    public Resource update(@PathVariable Long id, @RequestBody Resource resource) { ... }

    @PatchMapping("/resources/{id}")
    public Resource patch(@PathVariable Long id, @RequestBody Map<String, Object> updates) { ... }

    @DeleteMapping("/resources/{id}")
    public void delete(@PathVariable Long id) { ... }

    // Content negotiation
    @GetMapping(value = "/data", produces = MediaType.APPLICATION_JSON_VALUE)
    public Data getJson() { ... }

    @GetMapping(value = "/data", produces = MediaType.APPLICATION_XML_VALUE)
    public Data getXml() { ... }

    @PostMapping(value = "/upload", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public Response upload(@RequestParam("file") MultipartFile file) { ... }

    // Multiple paths
    @GetMapping({"/path1", "/path2"})
    public Response multiplePaths() { ... }

    // Path patterns
    @GetMapping("/files/**")
    public Resource handleFiles(HttpServletRequest request) {
        String path = request.getRequestURI();
        // Handle file path
        return null;
    }
}
```

### Response Status Codes

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)  // 201
    public Order create(@RequestBody Order order) { ... }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)  // 204
    public void delete(@PathVariable Long id) { ... }

    @GetMapping("/{id}")
    public ResponseEntity<Order> findById(@PathVariable Long id) {
        return orderService.findById(id)
            .map(ResponseEntity::ok)  // 200
            .orElse(ResponseEntity.notFound().build());  // 404
    }

    @PostMapping("/validate")
    public ResponseEntity<ValidationResult> validate(@RequestBody Order order) {
        ValidationResult result = orderService.validate(order);
        if (result.isValid()) {
            return ResponseEntity.ok(result);
        }
        return ResponseEntity.badRequest().body(result);  // 400
    }
}
```

---

## 6. Request/Response Handling

### @PathVariable

```java
@RestController
@RequestMapping("/api/v1")
public class PathVariableExamples {

    // Basic path variable
    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) { ... }

    // Named path variable
    @GetMapping("/users/{userId}/orders/{orderId}")
    public Order getOrder(
            @PathVariable("userId") Long userId,
            @PathVariable("orderId") Long orderId) { ... }

    // Optional path variable
    @GetMapping({"/items", "/items/{id}"})
    public ResponseEntity<?> getItems(
            @PathVariable(required = false) Long id) {
        if (id == null) {
            return ResponseEntity.ok(itemService.findAll());
        }
        return ResponseEntity.ok(itemService.findById(id));
    }

    // Path variable with regex
    @GetMapping("/files/{filename:.+}")
    public Resource getFile(@PathVariable String filename) { ... }

    // Multiple path variables in Map
    @GetMapping("/entities/{type}/{id}")
    public Object getEntity(@PathVariable Map<String, String> pathVars) {
        String type = pathVars.get("type");
        String id = pathVars.get("id");
        return entityService.find(type, id);
    }
}
```

### @RequestParam

```java
@RestController
@RequestMapping("/api/v1/products")
public class RequestParamExamples {

    // Basic query parameter
    @GetMapping
    public List<Product> search(@RequestParam String query) { ... }

    // With default value
    @GetMapping("/list")
    public Page<Product> list(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) { ... }

    // Optional parameter
    @GetMapping("/filter")
    public List<Product> filter(
            @RequestParam(required = false) String category,
            @RequestParam(required = false) BigDecimal minPrice,
            @RequestParam(required = false) BigDecimal maxPrice) { ... }

    // List parameter
    @GetMapping("/by-ids")
    public List<Product> findByIds(@RequestParam List<Long> ids) { ... }
    // Called as: /by-ids?ids=1,2,3 or /by-ids?ids=1&ids=2&ids=3

    // Map of all parameters
    @GetMapping("/search")
    public List<Product> search(@RequestParam Map<String, String> params) {
        String query = params.get("q");
        String sort = params.getOrDefault("sort", "name");
        return productService.search(query, sort);
    }

    // Date parameters
    @GetMapping("/by-date")
    public List<Product> findByDate(
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate date) { ... }
}
```

### @RequestBody

```java
@RestController
@RequestMapping("/api/v1/users")
public class RequestBodyExamples {

    // Basic request body
    @PostMapping
    public User create(@RequestBody CreateUserRequest request) { ... }

    // With validation
    @PostMapping("/validated")
    public User createValidated(@Valid @RequestBody CreateUserRequest request) { ... }

    // Optional request body
    @PatchMapping("/{id}")
    public User update(
            @PathVariable Long id,
            @RequestBody(required = false) UpdateUserRequest request) { ... }

    // Generic JSON
    @PostMapping("/dynamic")
    public ResponseEntity<?> processDynamic(@RequestBody Map<String, Object> payload) {
        String type = (String) payload.get("type");
        // Process based on type
        return ResponseEntity.ok().build();
    }

    // List request body
    @PostMapping("/batch")
    public List<User> createBatch(@Valid @RequestBody List<CreateUserRequest> requests) { ... }
}
```

### @RequestHeader

```java
@RestController
@RequestMapping("/api/v1")
public class RequestHeaderExamples {

    @GetMapping("/info")
    public ResponseEntity<Map<String, String>> getInfo(
            @RequestHeader("User-Agent") String userAgent,
            @RequestHeader(value = "X-Request-ID", required = false) String requestId,
            @RequestHeader(value = "Accept-Language", defaultValue = "en") String language) {

        Map<String, String> info = new HashMap<>();
        info.put("userAgent", userAgent);
        info.put("requestId", requestId);
        info.put("language", language);
        return ResponseEntity.ok(info);
    }

    // All headers
    @GetMapping("/headers")
    public Map<String, String> getAllHeaders(@RequestHeader HttpHeaders headers) {
        return headers.toSingleValueMap();
    }
}
```

### Response Building

```java
@RestController
@RequestMapping("/api/v1/files")
public class ResponseExamples {

    // Custom headers
    @GetMapping("/{id}")
    public ResponseEntity<Resource> downloadFile(@PathVariable Long id) {
        Resource file = fileService.getFile(id);
        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + file.getFilename() + "\"")
            .header(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_OCTET_STREAM_VALUE)
            .body(file);
    }

    // Conditional response
    @GetMapping("/resource/{id}")
    public ResponseEntity<Resource> getResource(@PathVariable Long id) {
        return resourceService.findById(id)
            .map(resource -> ResponseEntity.ok()
                .cacheControl(CacheControl.maxAge(1, TimeUnit.HOURS))
                .eTag(resource.getVersion())
                .body(resource))
            .orElse(ResponseEntity.notFound().build());
    }

    // Streaming response
    @GetMapping(value = "/stream", produces = MediaType.APPLICATION_NDJSON_VALUE)
    public Flux<Event> streamEvents() {
        return eventService.getEventStream();
    }
}
```

---

## 7. Service Layer

### Basic Service Implementation

```java
public interface UserService {
    Page<UserResponse> findAll(Pageable pageable);
    UserResponse findById(Long id);
    UserResponse create(CreateUserRequest request);
    UserResponse update(Long id, UpdateUserRequest request);
    void delete(Long id);
}

@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
@Slf4j
public class UserServiceImpl implements UserService {

    private final UserRepository userRepository;
    private final UserMapper userMapper;
    private final PasswordEncoder passwordEncoder;
    private final ApplicationEventPublisher eventPublisher;

    @Override
    public Page<UserResponse> findAll(Pageable pageable) {
        log.debug("Finding all users with pageable: {}", pageable);
        return userRepository.findAll(pageable)
            .map(userMapper::toResponse);
    }

    @Override
    public UserResponse findById(Long id) {
        log.debug("Finding user by id: {}", id);
        return userRepository.findById(id)
            .map(userMapper::toResponse)
            .orElseThrow(() -> new ResourceNotFoundException("User", "id", id));
    }

    @Override
    @Transactional
    public UserResponse create(CreateUserRequest request) {
        log.info("Creating new user with email: {}", request.getEmail());

        if (userRepository.existsByEmail(request.getEmail())) {
            throw new DuplicateResourceException("User", "email", request.getEmail());
        }

        User user = userMapper.toEntity(request);
        user.setPassword(passwordEncoder.encode(request.getPassword()));
        user.setStatus(UserStatus.ACTIVE);

        User savedUser = userRepository.save(user);

        eventPublisher.publishEvent(new UserCreatedEvent(savedUser));

        log.info("Created user with id: {}", savedUser.getId());
        return userMapper.toResponse(savedUser);
    }

    @Override
    @Transactional
    public UserResponse update(Long id, UpdateUserRequest request) {
        log.info("Updating user with id: {}", id);

        User user = userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User", "id", id));

        userMapper.updateEntity(user, request);
        User updatedUser = userRepository.save(user);

        return userMapper.toResponse(updatedUser);
    }

    @Override
    @Transactional
    public void delete(Long id) {
        log.info("Deleting user with id: {}", id);

        if (!userRepository.existsById(id)) {
            throw new ResourceNotFoundException("User", "id", id);
        }

        userRepository.deleteById(id);
        eventPublisher.publishEvent(new UserDeletedEvent(id));
    }
}
```

### Transaction Management

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final InventoryService inventoryService;
    private final PaymentService paymentService;

    // Read-only transaction (optimized for reads)
    @Transactional(readOnly = true)
    public Order findById(Long id) {
        return orderRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Order", "id", id));
    }

    // Write transaction with rollback rules
    @Transactional(
        rollbackFor = Exception.class,
        noRollbackFor = NotificationException.class
    )
    public Order createOrder(CreateOrderRequest request) {
        Order order = new Order();
        // ... create order logic

        inventoryService.reserve(order.getItems());
        Order savedOrder = orderRepository.save(order);

        try {
            notificationService.sendOrderConfirmation(savedOrder);
        } catch (NotificationException e) {
            log.warn("Failed to send notification for order: {}", savedOrder.getId());
            // Don't rollback for notification failures
        }

        return savedOrder;
    }

    // Transaction with propagation
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logOrderAudit(Order order) {
        // Runs in a separate transaction
        auditRepository.save(new OrderAudit(order));
    }

    // Transaction with isolation level
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public void processPayment(Long orderId) {
        Order order = orderRepository.findByIdWithLock(orderId);
        paymentService.charge(order);
        order.setStatus(OrderStatus.PAID);
        orderRepository.save(order);
    }

    // Transaction with timeout
    @Transactional(timeout = 30)  // 30 seconds
    public void processLongRunningOperation(Long orderId) {
        // Long-running operation
    }
}
```

### Async Service Methods

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class EmailService {

    private final JavaMailSender mailSender;
    private final TemplateEngine templateEngine;

    @Async
    public CompletableFuture<Void> sendWelcomeEmail(User user) {
        log.info("Sending welcome email to: {}", user.getEmail());

        try {
            MimeMessage message = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(message, true);

            helper.setTo(user.getEmail());
            helper.setSubject("Welcome to Our Service");
            helper.setText(renderTemplate("welcome", Map.of("user", user)), true);

            mailSender.send(message);
            log.info("Welcome email sent successfully to: {}", user.getEmail());

            return CompletableFuture.completedFuture(null);
        } catch (Exception e) {
            log.error("Failed to send welcome email to: {}", user.getEmail(), e);
            return CompletableFuture.failedFuture(e);
        }
    }

    @Async("emailTaskExecutor")  // Use specific executor
    public void sendBulkEmails(List<EmailRequest> requests) {
        requests.forEach(this::sendEmail);
    }

    private String renderTemplate(String templateName, Map<String, Object> variables) {
        Context context = new Context();
        context.setVariables(variables);
        return templateEngine.process(templateName, context);
    }
}

// Async Configuration
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "emailTaskExecutor")
    public Executor emailTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(5);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("Email-");
        executor.initialize();
        return executor;
    }
}
```

---

## 8. Repository Layer

### Spring Data JPA Repository

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    // Derived query methods
    Optional<User> findByEmail(String email);

    List<User> findByStatus(UserStatus status);

    List<User> findByNameContainingIgnoreCase(String name);

    List<User> findByCreatedAtBetween(LocalDateTime start, LocalDateTime end);

    boolean existsByEmail(String email);

    long countByStatus(UserStatus status);

    // Pagination and sorting
    Page<User> findByStatus(UserStatus status, Pageable pageable);

    List<User> findByStatusOrderByCreatedAtDesc(UserStatus status);

    // Multiple conditions
    List<User> findByStatusAndRoleAndCreatedAtAfter(
        UserStatus status, UserRole role, LocalDateTime date);

    // Top/First
    List<User> findTop10ByStatusOrderByCreatedAtDesc(UserStatus status);

    Optional<User> findFirstByEmailOrderByCreatedAtDesc(String email);

    // Delete methods
    void deleteByStatus(UserStatus status);

    @Modifying
    @Query("DELETE FROM User u WHERE u.status = :status AND u.lastLoginAt < :date")
    int deleteInactiveUsers(@Param("status") UserStatus status, @Param("date") LocalDateTime date);
}
```

### Custom JPQL Queries

```java
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    // JPQL query
    @Query("SELECT o FROM Order o WHERE o.user.id = :userId AND o.status = :status")
    List<Order> findUserOrdersByStatus(
        @Param("userId") Long userId,
        @Param("status") OrderStatus status);

    // JPQL with joins
    @Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
    Optional<Order> findByIdWithItems(@Param("id") Long id);

    // JPQL aggregate
    @Query("SELECT SUM(o.total) FROM Order o WHERE o.user.id = :userId")
    BigDecimal getTotalSpentByUser(@Param("userId") Long userId);

    // JPQL with projection
    @Query("SELECT new com.example.dto.OrderSummary(o.id, o.status, o.total) " +
           "FROM Order o WHERE o.user.id = :userId")
    List<OrderSummary> findOrderSummariesByUser(@Param("userId") Long userId);

    // JPQL update
    @Modifying
    @Query("UPDATE Order o SET o.status = :status WHERE o.id = :id")
    int updateStatus(@Param("id") Long id, @Param("status") OrderStatus status);

    // Native query
    @Query(value = "SELECT * FROM orders WHERE created_at > :date", nativeQuery = true)
    List<Order> findRecentOrdersNative(@Param("date") LocalDateTime date);

    // Named query (defined in entity)
    List<Order> findPendingOrders();
}
```

### Custom Repository Implementation

```java
// Custom repository interface
public interface UserRepositoryCustom {
    List<User> searchUsers(UserSearchCriteria criteria);
    void bulkUpdateStatus(List<Long> ids, UserStatus status);
}

// Implementation
@RequiredArgsConstructor
public class UserRepositoryImpl implements UserRepositoryCustom {

    private final EntityManager entityManager;

    @Override
    public List<User> searchUsers(UserSearchCriteria criteria) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<User> query = cb.createQuery(User.class);
        Root<User> user = query.from(User.class);

        List<Predicate> predicates = new ArrayList<>();

        if (criteria.getName() != null) {
            predicates.add(cb.like(cb.lower(user.get("name")),
                "%" + criteria.getName().toLowerCase() + "%"));
        }

        if (criteria.getStatus() != null) {
            predicates.add(cb.equal(user.get("status"), criteria.getStatus()));
        }

        if (criteria.getCreatedAfter() != null) {
            predicates.add(cb.greaterThan(user.get("createdAt"), criteria.getCreatedAfter()));
        }

        query.where(predicates.toArray(new Predicate[0]));
        query.orderBy(cb.desc(user.get("createdAt")));

        return entityManager.createQuery(query)
            .setMaxResults(criteria.getLimit())
            .getResultList();
    }

    @Override
    @Transactional
    public void bulkUpdateStatus(List<Long> ids, UserStatus status) {
        String jpql = "UPDATE User u SET u.status = :status WHERE u.id IN :ids";
        entityManager.createQuery(jpql)
            .setParameter("status", status)
            .setParameter("ids", ids)
            .executeUpdate();
    }
}

// Combined repository interface
@Repository
public interface UserRepository extends JpaRepository<User, Long>, UserRepositoryCustom {
    // Standard JPA methods + custom methods
}
```

### JPA Entity Definitions

```java
@Entity
@Table(name = "users", indexes = {
    @Index(name = "idx_user_email", columnList = "email"),
    @Index(name = "idx_user_status", columnList = "status")
})
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
@EntityListeners(AuditingEntityListener.class)
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(unique = true, nullable = false, length = 255)
    private String email;

    @Column(nullable = false)
    private String password;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private UserStatus status = UserStatus.PENDING;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private UserRole role = UserRole.USER;

    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
    @Builder.Default
    private List<Order> orders = new ArrayList<>();

    @OneToOne(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    @JoinColumn(name = "profile_id")
    private UserProfile profile;

    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    @Builder.Default
    private Set<Role> roles = new HashSet<>();

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    @Version
    private Long version;

    // Helper methods for bidirectional relationships
    public void addOrder(Order order) {
        orders.add(order);
        order.setUser(this);
    }

    public void removeOrder(Order order) {
        orders.remove(order);
        order.setUser(null);
    }
}
```

---

## 9. Dependency Injection

### Constructor Injection (Recommended)

```java
@Service
public class UserService {

    private final UserRepository userRepository;
    private final EmailService emailService;
    private final PasswordEncoder passwordEncoder;

    // Constructor injection - preferred method
    public UserService(
            UserRepository userRepository,
            EmailService emailService,
            PasswordEncoder passwordEncoder) {
        this.userRepository = userRepository;
        this.emailService = emailService;
        this.passwordEncoder = passwordEncoder;
    }
}

// With Lombok @RequiredArgsConstructor
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;
    private final EmailService emailService;
    private final PasswordEncoder passwordEncoder;

    // Constructor is generated automatically
}
```

### Field Injection (Not Recommended for Production)

```java
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private EmailService emailService;

    // Not recommended: harder to test, hides dependencies
}
```

### Setter Injection

```java
@Service
public class UserService {

    private UserRepository userRepository;

    @Autowired
    public void setUserRepository(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    // Useful for optional dependencies
    @Autowired(required = false)
    public void setAuditService(AuditService auditService) {
        this.auditService = auditService;
    }
}
```

### Qualifier and Primary

```java
// Multiple implementations
public interface PaymentProcessor {
    void process(Payment payment);
}

@Service
@Primary  // Default when no qualifier specified
public class StripePaymentProcessor implements PaymentProcessor {
    @Override
    public void process(Payment payment) { ... }
}

@Service("paypal")
public class PayPalPaymentProcessor implements PaymentProcessor {
    @Override
    public void process(Payment payment) { ... }
}

// Using @Qualifier to select specific bean
@Service
@RequiredArgsConstructor
public class PaymentService {

    @Qualifier("paypal")
    private final PaymentProcessor paypalProcessor;

    private final PaymentProcessor defaultProcessor;  // Gets @Primary bean
}

// Using @Qualifier with constructor injection
@Service
public class PaymentService {

    private final PaymentProcessor stripeProcessor;
    private final PaymentProcessor paypalProcessor;

    public PaymentService(
            @Qualifier("stripe") PaymentProcessor stripeProcessor,
            @Qualifier("paypal") PaymentProcessor paypalProcessor) {
        this.stripeProcessor = stripeProcessor;
        this.paypalProcessor = paypalProcessor;
    }
}
```

### Optional Dependencies

```java
@Service
@RequiredArgsConstructor
public class NotificationService {

    private final Optional<SmsService> smsService;
    private final Optional<PushNotificationService> pushService;

    public void notify(User user, String message) {
        // Email is always available
        emailService.send(user.getEmail(), message);

        // SMS is optional
        smsService.ifPresent(sms -> sms.send(user.getPhone(), message));

        // Push notifications are optional
        pushService.ifPresent(push -> push.send(user.getDeviceToken(), message));
    }
}
```

### Bean Configuration

```java
@Configuration
public class AppConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    @Scope("prototype")  // New instance each time
    public RequestContext requestContext() {
        return new RequestContext();
    }

    @Bean
    @ConditionalOnProperty(name = "app.cache.enabled", havingValue = "true")
    public CacheManager cacheManager() {
        return new ConcurrentMapCacheManager("users", "products");
    }

    @Bean
    @ConditionalOnMissingBean(EmailService.class)
    public EmailService defaultEmailService() {
        return new ConsoleEmailService();
    }
}
```

---

## 10. Exception Handling

### Global Exception Handler with @ControllerAdvice

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleResourceNotFound(ResourceNotFoundException ex, WebRequest request) {
        log.warn("Resource not found: {}", ex.getMessage());
        return ErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(HttpStatus.NOT_FOUND.value())
            .error("Not Found")
            .message(ex.getMessage())
            .path(extractPath(request))
            .build();
    }

    @ExceptionHandler(DuplicateResourceException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    public ErrorResponse handleDuplicateResource(DuplicateResourceException ex, WebRequest request) {
        log.warn("Duplicate resource: {}", ex.getMessage());
        return ErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(HttpStatus.CONFLICT.value())
            .error("Conflict")
            .message(ex.getMessage())
            .path(extractPath(request))
            .build();
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ValidationErrorResponse handleValidationErrors(
            MethodArgumentNotValidException ex, WebRequest request) {

        List<FieldError> fieldErrors = ex.getBindingResult().getFieldErrors()
            .stream()
            .map(error -> new FieldError(
                error.getField(),
                error.getDefaultMessage(),
                error.getRejectedValue()
            ))
            .collect(Collectors.toList());

        log.warn("Validation failed: {}", fieldErrors);

        return ValidationErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(HttpStatus.BAD_REQUEST.value())
            .error("Validation Failed")
            .message("Request validation failed")
            .path(extractPath(request))
            .errors(fieldErrors)
            .build();
    }

    @ExceptionHandler(ConstraintViolationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ValidationErrorResponse handleConstraintViolation(
            ConstraintViolationException ex, WebRequest request) {

        List<FieldError> fieldErrors = ex.getConstraintViolations()
            .stream()
            .map(violation -> new FieldError(
                violation.getPropertyPath().toString(),
                violation.getMessage(),
                violation.getInvalidValue()
            ))
            .collect(Collectors.toList());

        return ValidationErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(HttpStatus.BAD_REQUEST.value())
            .error("Validation Failed")
            .message("Constraint violation")
            .path(extractPath(request))
            .errors(fieldErrors)
            .build();
    }

    @ExceptionHandler(AccessDeniedException.class)
    @ResponseStatus(HttpStatus.FORBIDDEN)
    public ErrorResponse handleAccessDenied(AccessDeniedException ex, WebRequest request) {
        log.warn("Access denied: {}", ex.getMessage());
        return ErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(HttpStatus.FORBIDDEN.value())
            .error("Forbidden")
            .message("Access denied")
            .path(extractPath(request))
            .build();
    }

    @ExceptionHandler(HttpRequestMethodNotSupportedException.class)
    @ResponseStatus(HttpStatus.METHOD_NOT_ALLOWED)
    public ErrorResponse handleMethodNotSupported(
            HttpRequestMethodNotSupportedException ex, WebRequest request) {
        return ErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(HttpStatus.METHOD_NOT_ALLOWED.value())
            .error("Method Not Allowed")
            .message(ex.getMessage())
            .path(extractPath(request))
            .build();
    }

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGenericException(Exception ex, WebRequest request) {
        log.error("Unexpected error occurred", ex);
        return ErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(HttpStatus.INTERNAL_SERVER_ERROR.value())
            .error("Internal Server Error")
            .message("An unexpected error occurred")
            .path(extractPath(request))
            .build();
    }

    private String extractPath(WebRequest request) {
        return ((ServletWebRequest) request).getRequest().getRequestURI();
    }
}
```

### Custom Exception Classes

```java
public class ResourceNotFoundException extends RuntimeException {

    private final String resourceName;
    private final String fieldName;
    private final Object fieldValue;

    public ResourceNotFoundException(String resourceName, String fieldName, Object fieldValue) {
        super(String.format("%s not found with %s: '%s'", resourceName, fieldName, fieldValue));
        this.resourceName = resourceName;
        this.fieldName = fieldName;
        this.fieldValue = fieldValue;
    }
}

public class DuplicateResourceException extends RuntimeException {

    public DuplicateResourceException(String resourceName, String fieldName, Object fieldValue) {
        super(String.format("%s already exists with %s: '%s'", resourceName, fieldName, fieldValue));
    }
}

public class BusinessException extends RuntimeException {

    private final String errorCode;
    private final Map<String, Object> details;

    public BusinessException(String errorCode, String message) {
        this(errorCode, message, Collections.emptyMap());
    }

    public BusinessException(String errorCode, String message, Map<String, Object> details) {
        super(message);
        this.errorCode = errorCode;
        this.details = details;
    }
}
```

### Error Response DTOs

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ErrorResponse {
    private LocalDateTime timestamp;
    private int status;
    private String error;
    private String message;
    private String path;
}

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ValidationErrorResponse {
    private LocalDateTime timestamp;
    private int status;
    private String error;
    private String message;
    private String path;
    private List<FieldError> errors;
}

@Data
@AllArgsConstructor
@NoArgsConstructor
public class FieldError {
    private String field;
    private String message;
    private Object rejectedValue;
}
```

---

## 11. Validation

### Request Validation with Bean Validation

```java
@Data
public class CreateUserRequest {

    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 100, message = "Name must be between 2 and 100 characters")
    private String name;

    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    private String email;

    @NotBlank(message = "Password is required")
    @Size(min = 8, max = 128, message = "Password must be between 8 and 128 characters")
    @Pattern(
        regexp = "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]+$",
        message = "Password must contain at least one uppercase, lowercase, digit, and special character"
    )
    private String password;

    @NotNull(message = "Age is required")
    @Min(value = 18, message = "Age must be at least 18")
    @Max(value = 150, message = "Age must be at most 150")
    private Integer age;

    @Past(message = "Birth date must be in the past")
    private LocalDate birthDate;

    @Valid  // Cascade validation to nested object
    @NotNull(message = "Address is required")
    private AddressRequest address;

    @Valid
    @Size(max = 5, message = "Maximum 5 phone numbers allowed")
    private List<PhoneRequest> phoneNumbers;
}

@Data
public class AddressRequest {

    @NotBlank(message = "Street is required")
    private String street;

    @NotBlank(message = "City is required")
    private String city;

    @NotBlank(message = "Country is required")
    @Size(min = 2, max = 2, message = "Country code must be 2 characters")
    private String countryCode;

    @Pattern(regexp = "^\\d{5}(-\\d{4})?$", message = "Invalid ZIP code format")
    private String zipCode;
}
```

### Custom Validators

```java
// Custom annotation
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UniqueEmailValidator.class)
@Documented
public @interface UniqueEmail {
    String message() default "Email already exists";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// Validator implementation
@Component
@RequiredArgsConstructor
public class UniqueEmailValidator implements ConstraintValidator<UniqueEmail, String> {

    private final UserRepository userRepository;

    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        if (email == null) {
            return true;  // Let @NotNull handle null check
        }
        return !userRepository.existsByEmail(email);
    }
}

// Usage
@Data
public class CreateUserRequest {

    @NotBlank
    @Email
    @UniqueEmail
    private String email;
}
```

### Cross-Field Validation

```java
// Class-level constraint
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PasswordMatchValidator.class)
public @interface PasswordMatch {
    String message() default "Passwords do not match";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

@Component
public class PasswordMatchValidator implements ConstraintValidator<PasswordMatch, ChangePasswordRequest> {

    @Override
    public boolean isValid(ChangePasswordRequest request, ConstraintValidatorContext context) {
        if (request.getNewPassword() == null || request.getConfirmPassword() == null) {
            return true;
        }

        boolean isValid = request.getNewPassword().equals(request.getConfirmPassword());

        if (!isValid) {
            context.disableDefaultConstraintViolation();
            context.buildConstraintViolationWithTemplate("Passwords do not match")
                .addPropertyNode("confirmPassword")
                .addConstraintViolation();
        }

        return isValid;
    }
}

@Data
@PasswordMatch
public class ChangePasswordRequest {

    @NotBlank
    private String currentPassword;

    @NotBlank
    @Size(min = 8)
    private String newPassword;

    @NotBlank
    private String confirmPassword;
}
```

### Validation Groups

```java
// Define validation groups
public interface ValidationGroups {
    interface Create {}
    interface Update {}
}

@Data
public class UserRequest {

    @Null(groups = ValidationGroups.Create.class, message = "ID must be null for creation")
    @NotNull(groups = ValidationGroups.Update.class, message = "ID is required for update")
    private Long id;

    @NotBlank(groups = {ValidationGroups.Create.class, ValidationGroups.Update.class})
    private String name;

    @NotBlank(groups = ValidationGroups.Create.class)
    @Email
    private String email;  // Email only required for creation
}

// Using in controller
@RestController
@RequestMapping("/api/users")
public class UserController {

    @PostMapping
    public User create(@Validated(ValidationGroups.Create.class) @RequestBody UserRequest request) {
        return userService.create(request);
    }

    @PutMapping("/{id}")
    public User update(
            @PathVariable Long id,
            @Validated(ValidationGroups.Update.class) @RequestBody UserRequest request) {
        return userService.update(id, request);
    }
}
```

### Service Layer Validation

```java
@Service
@Validated  // Enable validation at service level
@RequiredArgsConstructor
public class UserService {

    public User findById(@NotNull @Positive Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User", "id", id));
    }

    public User create(@Valid CreateUserRequest request) {
        // Method-level validation
        return userRepository.save(mapper.toEntity(request));
    }

    public List<User> findByIds(@NotEmpty List<@Positive Long> ids) {
        return userRepository.findAllById(ids);
    }
}
```

---

## 12. Logging

### Logback Configuration (logback-spring.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true" scanPeriod="30 seconds">

    <!-- Properties -->
    <springProperty scope="context" name="APP_NAME" source="spring.application.name"/>
    <property name="LOG_PATH" value="${LOG_PATH:-logs}"/>
    <property name="LOG_FILE" value="${LOG_FILE:-${APP_NAME}}"/>

    <!-- Console Appender -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %highlight(%-5level) [%thread] %cyan(%logger{36}) - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- File Appender -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/${LOG_FILE}.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${LOG_PATH}/${LOG_FILE}-%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>3GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%thread] %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- JSON Appender for production -->
    <appender name="JSON" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/${LOG_FILE}-json.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${LOG_PATH}/${LOG_FILE}-json-%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdc>true</includeMdc>
            <customFields>{"application":"${APP_NAME}"}</customFields>
        </encoder>
    </appender>

    <!-- Async Appender -->
    <appender name="ASYNC_FILE" class="ch.qos.logback.classic.AsyncAppender">
        <queueSize>512</queueSize>
        <discardingThreshold>0</discardingThreshold>
        <appender-ref ref="FILE"/>
    </appender>

    <!-- Profile-specific configuration -->
    <springProfile name="dev">
        <root level="DEBUG">
            <appender-ref ref="CONSOLE"/>
        </root>
        <logger name="com.example" level="DEBUG"/>
        <logger name="org.springframework.web" level="DEBUG"/>
        <logger name="org.hibernate.SQL" level="DEBUG"/>
        <logger name="org.hibernate.type.descriptor.sql.BasicBinder" level="TRACE"/>
    </springProfile>

    <springProfile name="prod">
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
            <appender-ref ref="ASYNC_FILE"/>
            <appender-ref ref="JSON"/>
        </root>
        <logger name="com.example" level="INFO"/>
        <logger name="org.springframework" level="WARN"/>
        <logger name="org.hibernate" level="WARN"/>
    </springProfile>

</configuration>
```

### Using SLF4J with Lombok

```java
@Service
@Slf4j
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;

    public User findById(Long id) {
        log.debug("Finding user by id: {}", id);

        return userRepository.findById(id)
            .map(user -> {
                log.debug("Found user: {}", user.getEmail());
                return user;
            })
            .orElseThrow(() -> {
                log.warn("User not found with id: {}", id);
                return new ResourceNotFoundException("User", "id", id);
            });
    }

    @Transactional
    public User create(CreateUserRequest request) {
        log.info("Creating new user with email: {}", request.getEmail());

        try {
            User user = mapper.toEntity(request);
            User savedUser = userRepository.save(user);
            log.info("Successfully created user with id: {}", savedUser.getId());
            return savedUser;
        } catch (DataIntegrityViolationException e) {
            log.error("Failed to create user due to data integrity violation", e);
            throw new DuplicateResourceException("User", "email", request.getEmail());
        }
    }

    public void processUsers(List<Long> userIds) {
        log.info("Processing {} users", userIds.size());

        long startTime = System.currentTimeMillis();
        int processed = 0;
        int failed = 0;

        for (Long userId : userIds) {
            try {
                processUser(userId);
                processed++;
            } catch (Exception e) {
                log.error("Failed to process user {}: {}", userId, e.getMessage());
                failed++;
            }
        }

        long duration = System.currentTimeMillis() - startTime;
        log.info("Completed processing users. Processed: {}, Failed: {}, Duration: {}ms",
            processed, failed, duration);
    }
}
```

### MDC (Mapped Diagnostic Context) for Request Tracing

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class RequestLoggingFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        try {
            String requestId = request.getHeader("X-Request-ID");
            if (requestId == null) {
                requestId = UUID.randomUUID().toString();
            }

            MDC.put("requestId", requestId);
            MDC.put("clientIP", request.getRemoteAddr());
            MDC.put("method", request.getMethod());
            MDC.put("uri", request.getRequestURI());

            response.setHeader("X-Request-ID", requestId);

            filterChain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
}
```

---

## 13. Actuator Endpoints

### Adding Actuator Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### Actuator Configuration

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,env,loggers
      base-path: /actuator
  endpoint:
    health:
      show-details: when_authorized
      show-components: when_authorized
    env:
      show-values: when_authorized
    loggers:
      enabled: true
  health:
    db:
      enabled: true
    diskspace:
      enabled: true
      threshold: 10MB
    redis:
      enabled: true
  info:
    env:
      enabled: true
    git:
      mode: full
    build:
      enabled: true
  metrics:
    tags:
      application: ${spring.application.name}
    export:
      prometheus:
        enabled: true

# Application Info
info:
  app:
    name: ${spring.application.name}
    description: Spring Boot Application
    version: '@project.version@'
    java:
      version: ${java.version}
  contact:
    email: support@example.com
```

### Custom Health Indicators

```java
@Component
public class CustomServiceHealthIndicator implements HealthIndicator {

    private final ExternalServiceClient externalService;

    public CustomServiceHealthIndicator(ExternalServiceClient externalService) {
        this.externalService = externalService;
    }

    @Override
    public Health health() {
        try {
            boolean isHealthy = externalService.ping();
            if (isHealthy) {
                return Health.up()
                    .withDetail("service", "External Service")
                    .withDetail("status", "Available")
                    .build();
            } else {
                return Health.down()
                    .withDetail("service", "External Service")
                    .withDetail("status", "Unavailable")
                    .build();
            }
        } catch (Exception e) {
            return Health.down()
                .withDetail("service", "External Service")
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}

// Composite Health Contributor
@Component
public class ServicesHealthContributor implements CompositeHealthContributor {

    private final Map<String, HealthContributor> contributors;

    public ServicesHealthContributor(
            DatabaseHealthIndicator dbHealth,
            CacheHealthIndicator cacheHealth,
            MessageQueueHealthIndicator mqHealth) {

        this.contributors = Map.of(
            "database", dbHealth,
            "cache", cacheHealth,
            "messageQueue", mqHealth
        );
    }

    @Override
    public HealthContributor getContributor(String name) {
        return contributors.get(name);
    }

    @Override
    public Iterator<NamedContributor<HealthContributor>> iterator() {
        return contributors.entrySet().stream()
            .map(entry -> NamedContributor.of(entry.getKey(), entry.getValue()))
            .iterator();
    }
}
```

### Custom Metrics

```java
@Component
@RequiredArgsConstructor
public class OrderMetrics {

    private final MeterRegistry meterRegistry;
    private final AtomicLong activeOrders = new AtomicLong(0);

    @PostConstruct
    public void init() {
        // Gauge - current value
        Gauge.builder("orders.active", activeOrders, AtomicLong::get)
            .description("Number of active orders")
            .tag("type", "gauge")
            .register(meterRegistry);
    }

    public void recordOrderCreated(Order order) {
        // Counter - incrementing value
        meterRegistry.counter("orders.created",
            "status", order.getStatus().name(),
            "channel", order.getChannel()
        ).increment();

        activeOrders.incrementAndGet();
    }

    public void recordOrderCompleted(Order order, long processingTimeMs) {
        // Timer - duration
        meterRegistry.timer("orders.processing.time",
            "status", order.getStatus().name()
        ).record(processingTimeMs, TimeUnit.MILLISECONDS);

        // Distribution summary - value distribution
        meterRegistry.summary("orders.total.amount",
            "currency", order.getCurrency()
        ).record(order.getTotal().doubleValue());

        activeOrders.decrementAndGet();
    }
}
```

### Custom Actuator Endpoint

```java
@Component
@Endpoint(id = "features")
public class FeaturesEndpoint {

    private final Map<String, Feature> features;

    public FeaturesEndpoint(FeatureService featureService) {
        this.features = featureService.getAllFeatures();
    }

    @ReadOperation
    public Map<String, Feature> features() {
        return features;
    }

    @ReadOperation
    public Feature feature(@Selector String name) {
        return features.get(name);
    }

    @WriteOperation
    public void configureFeature(@Selector String name, boolean enabled) {
        Feature feature = features.get(name);
        if (feature != null) {
            feature.setEnabled(enabled);
        }
    }

    @DeleteOperation
    public void deleteFeature(@Selector String name) {
        features.remove(name);
    }
}
```

---

## 14. Testing

### Unit Testing with @SpringBootTest

```java
@SpringBootTest
@ActiveProfiles("test")
class UserServiceIntegrationTest {

    @Autowired
    private UserService userService;

    @Autowired
    private UserRepository userRepository;

    @BeforeEach
    void setUp() {
        userRepository.deleteAll();
    }

    @Test
    void shouldCreateUser() {
        // Given
        CreateUserRequest request = CreateUserRequest.builder()
            .name("John Doe")
            .email("john@example.com")
            .password("Password123!")
            .build();

        // When
        UserResponse response = userService.create(request);

        // Then
        assertThat(response).isNotNull();
        assertThat(response.getId()).isNotNull();
        assertThat(response.getName()).isEqualTo("John Doe");
        assertThat(response.getEmail()).isEqualTo("john@example.com");

        // Verify persistence
        Optional<User> savedUser = userRepository.findById(response.getId());
        assertThat(savedUser).isPresent();
    }

    @Test
    void shouldThrowExceptionForDuplicateEmail() {
        // Given
        CreateUserRequest request = CreateUserRequest.builder()
            .name("John Doe")
            .email("john@example.com")
            .password("Password123!")
            .build();

        userService.create(request);

        // When/Then
        assertThrows(DuplicateResourceException.class, () -> {
            userService.create(request);
        });
    }
}
```

### Controller Testing with @WebMvcTest

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    @Autowired
    private ObjectMapper objectMapper;

    @Test
    void shouldGetUserById() throws Exception {
        // Given
        UserResponse user = UserResponse.builder()
            .id(1L)
            .name("John Doe")
            .email("john@example.com")
            .build();

        when(userService.findById(1L)).thenReturn(user);

        // When/Then
        mockMvc.perform(get("/api/v1/users/{id}", 1L)
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.name").value("John Doe"))
            .andExpect(jsonPath("$.email").value("john@example.com"));
    }

    @Test
    void shouldReturn404WhenUserNotFound() throws Exception {
        // Given
        when(userService.findById(999L))
            .thenThrow(new ResourceNotFoundException("User", "id", 999L));

        // When/Then
        mockMvc.perform(get("/api/v1/users/{id}", 999L)
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.error").value("Not Found"));
    }

    @Test
    void shouldCreateUser() throws Exception {
        // Given
        CreateUserRequest request = CreateUserRequest.builder()
            .name("John Doe")
            .email("john@example.com")
            .password("Password123!")
            .build();

        UserResponse response = UserResponse.builder()
            .id(1L)
            .name("John Doe")
            .email("john@example.com")
            .build();

        when(userService.create(any(CreateUserRequest.class))).thenReturn(response);

        // When/Then
        mockMvc.perform(post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(header().exists("Location"))
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.name").value("John Doe"));
    }

    @Test
    void shouldReturnValidationErrorsForInvalidRequest() throws Exception {
        // Given
        CreateUserRequest request = CreateUserRequest.builder()
            .name("")  // Invalid: blank
            .email("invalid-email")  // Invalid: not an email
            .password("123")  // Invalid: too short
            .build();

        // When/Then
        mockMvc.perform(post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors").isArray())
            .andExpect(jsonPath("$.errors.length()").value(3));
    }
}
```

### Repository Testing with @DataJpaTest

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class UserRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private TestEntityManager entityManager;

    @Test
    void shouldFindUserByEmail() {
        // Given
        User user = User.builder()
            .name("John Doe")
            .email("john@example.com")
            .password("encoded-password")
            .status(UserStatus.ACTIVE)
            .build();
        entityManager.persistAndFlush(user);

        // When
        Optional<User> found = userRepository.findByEmail("john@example.com");

        // Then
        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("John Doe");
    }

    @Test
    void shouldReturnEmptyWhenEmailNotFound() {
        // When
        Optional<User> found = userRepository.findByEmail("nonexistent@example.com");

        // Then
        assertThat(found).isEmpty();
    }

    @Test
    void shouldFindUsersByStatus() {
        // Given
        User activeUser = User.builder()
            .name("Active User")
            .email("active@example.com")
            .password("password")
            .status(UserStatus.ACTIVE)
            .build();

        User inactiveUser = User.builder()
            .name("Inactive User")
            .email("inactive@example.com")
            .password("password")
            .status(UserStatus.INACTIVE)
            .build();

        entityManager.persist(activeUser);
        entityManager.persist(inactiveUser);
        entityManager.flush();

        // When
        List<User> activeUsers = userRepository.findByStatus(UserStatus.ACTIVE);

        // Then
        assertThat(activeUsers).hasSize(1);
        assertThat(activeUsers.get(0).getName()).isEqualTo("Active User");
    }
}
```

### Mocking with @MockBean and @SpyBean

```java
@SpringBootTest
class OrderServiceTest {

    @Autowired
    private OrderService orderService;

    @MockBean
    private PaymentService paymentService;

    @SpyBean
    private OrderRepository orderRepository;

    @Test
    void shouldProcessOrderWithMockedPayment() {
        // Given
        Order order = new Order();
        order.setId(1L);
        order.setTotal(BigDecimal.valueOf(100));

        when(paymentService.processPayment(any())).thenReturn(true);

        // When
        Order result = orderService.processOrder(order);

        // Then
        assertThat(result.getStatus()).isEqualTo(OrderStatus.PAID);
        verify(paymentService).processPayment(any());
        verify(orderRepository).save(any());
    }
}
```

### Test Configuration

```java
@TestConfiguration
public class TestConfig {

    @Bean
    @Primary
    public Clock testClock() {
        return Clock.fixed(
            Instant.parse("2024-01-15T10:00:00Z"),
            ZoneId.of("UTC")
        );
    }

    @Bean
    @Primary
    public PasswordEncoder testPasswordEncoder() {
        return new BCryptPasswordEncoder(4);  // Lower strength for faster tests
    }
}

// application-test.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true
  flyway:
    enabled: false
```

---

## 15. Maven/Gradle Configuration

### Maven (pom.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>my-spring-boot-app</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>My Spring Boot Application</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>21</java.version>
        <mapstruct.version>1.5.5.Final</mapstruct.version>
        <lombok.version>1.18.30</lombok.version>
    </properties>

    <dependencies>
        <!-- Spring Boot Starters -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>

        <!-- Database -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>

        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-core</artifactId>
        </dependency>

        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- MapStruct -->
        <dependency>
            <groupId>org.mapstruct</groupId>
            <artifactId>mapstruct</artifactId>
            <version>${mapstruct.version}</version>
        </dependency>

        <!-- OpenAPI / Swagger -->
        <dependency>
            <groupId>org.springdoc</groupId>
            <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
            <version>2.3.0</version>
        </dependency>

        <!-- Micrometer for metrics -->
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-prometheus</artifactId>
        </dependency>

        <!-- Testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>postgresql</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>

        <!-- H2 for testing -->
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>${java.version}</source>
                    <target>${java.version}</target>
                    <annotationProcessorPaths>
                        <path>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                            <version>${lombok.version}</version>
                        </path>
                        <path>
                            <groupId>org.mapstruct</groupId>
                            <artifactId>mapstruct-processor</artifactId>
                            <version>${mapstruct.version}</version>
                        </path>
                        <path>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok-mapstruct-binding</artifactId>
                            <version>0.2.0</version>
                        </path>
                    </annotationProcessorPaths>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

### Gradle (build.gradle)

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.0'
    id 'io.spring.dependency-management' version '1.1.4'
}

group = 'com.example'
version = '1.0.0-SNAPSHOT'

java {
    sourceCompatibility = '21'
}

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
}

repositories {
    mavenCentral()
}

ext {
    mapstructVersion = '1.5.5.Final'
}

dependencies {
    // Spring Boot Starters
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'org.springframework.boot:spring-boot-starter-security'

    // Database
    runtimeOnly 'org.postgresql:postgresql'
    implementation 'org.flywaydb:flyway-core'

    // Lombok
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'

    // MapStruct
    implementation "org.mapstruct:mapstruct:${mapstructVersion}"
    annotationProcessor "org.mapstruct:mapstruct-processor:${mapstructVersion}"
    annotationProcessor 'org.projectlombok:lombok-mapstruct-binding:0.2.0'

    // OpenAPI / Swagger
    implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.3.0'

    // Micrometer for metrics
    implementation 'io.micrometer:micrometer-registry-prometheus'

    // Testing
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.security:spring-security-test'
    testImplementation 'org.testcontainers:postgresql'
    testImplementation 'org.testcontainers:junit-jupiter'
    testRuntimeOnly 'com.h2database:h2'
}

tasks.named('test') {
    useJUnitPlatform()
}

bootJar {
    archiveFileName = "${project.name}.jar"
}
```

---

## 16. Best Practices

### Project Structure

```
src/
├── main/
│   ├── java/
│   │   └── com/example/myapp/
│   │       ├── MyApplication.java
│   │       ├── config/
│   │       │   ├── SecurityConfig.java
│   │       │   ├── WebConfig.java
│   │       │   └── AsyncConfig.java
│   │       ├── controller/
│   │       │   ├── UserController.java
│   │       │   └── OrderController.java
│   │       ├── service/
│   │       │   ├── UserService.java
│   │       │   ├── impl/
│   │       │   │   └── UserServiceImpl.java
│   │       │   └── OrderService.java
│   │       ├── repository/
│   │       │   ├── UserRepository.java
│   │       │   └── OrderRepository.java
│   │       ├── entity/
│   │       │   ├── User.java
│   │       │   └── Order.java
│   │       ├── dto/
│   │       │   ├── request/
│   │       │   │   ├── CreateUserRequest.java
│   │       │   │   └── UpdateUserRequest.java
│   │       │   └── response/
│   │       │       └── UserResponse.java
│   │       ├── mapper/
│   │       │   └── UserMapper.java
│   │       ├── exception/
│   │       │   ├── GlobalExceptionHandler.java
│   │       │   ├── ResourceNotFoundException.java
│   │       │   └── DuplicateResourceException.java
│   │       └── util/
│   │           └── DateUtils.java
│   └── resources/
│       ├── application.yml
│       ├── application-dev.yml
│       ├── application-prod.yml
│       ├── db/migration/
│       │   └── V1__init.sql
│       └── logback-spring.xml
└── test/
    ├── java/
    │   └── com/example/myapp/
    │       ├── controller/
    │       │   └── UserControllerTest.java
    │       ├── service/
    │       │   └── UserServiceTest.java
    │       └── repository/
    │           └── UserRepositoryTest.java
    └── resources/
        └── application-test.yml
```

### Coding Best Practices

```java
// 1. Use constructor injection (with Lombok)
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserRepository userRepository;  // Immutable
}

// 2. Use DTOs for API boundaries
public record CreateUserRequest(
    @NotBlank String name,
    @Email String email
) {}

// 3. Use MapStruct for mapping
@Mapper(componentModel = "spring")
public interface UserMapper {
    UserResponse toResponse(User entity);
    User toEntity(CreateUserRequest request);

    @BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
    void updateEntity(@MappingTarget User entity, UpdateUserRequest request);
}

// 4. Handle Optional properly
public UserResponse findById(Long id) {
    return userRepository.findById(id)
        .map(userMapper::toResponse)
        .orElseThrow(() -> new ResourceNotFoundException("User", "id", id));
}

// 5. Use meaningful HTTP status codes
@PostMapping
public ResponseEntity<UserResponse> create(@Valid @RequestBody CreateUserRequest request) {
    UserResponse created = userService.create(request);
    URI location = ServletUriComponentsBuilder.fromCurrentRequest()
        .path("/{id}")
        .buildAndExpand(created.getId())
        .toUri();
    return ResponseEntity.created(location).body(created);  // 201 Created
}

// 6. Use pagination for lists
@GetMapping
public Page<UserResponse> findAll(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size) {
    return userService.findAll(PageRequest.of(page, size));
}

// 7. Use transactions appropriately
@Service
@Transactional(readOnly = true)  // Default read-only
public class UserService {

    @Transactional  // Write transaction
    public UserResponse create(CreateUserRequest request) { ... }
}

// 8. Validate at multiple levels
@RestController
@Validated  // Enable method-level validation
public class UserController {

    @GetMapping("/{id}")
    public UserResponse findById(@PathVariable @Positive Long id) { ... }
}

// 9. Use proper logging
@Slf4j
@Service
public class UserService {
    public void process(Long id) {
        log.debug("Processing user: {}", id);  // Debug for detailed info
        log.info("User {} processed successfully", id);  // Info for important events
        log.warn("User {} has expired session", id);  // Warn for potential issues
        log.error("Failed to process user {}", id, exception);  // Error with stack trace
    }
}

// 10. Return proper error responses
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ValidationErrorResponse handleValidation(MethodArgumentNotValidException ex) {
        // Return structured error response
    }
}
```

### Security Best Practices

```java
// 1. Never expose sensitive data
@JsonIgnore
private String password;

// 2. Use HTTPS in production
server:
  ssl:
    enabled: true
    key-store: classpath:keystore.p12

// 3. Configure CORS properly
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://example.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowCredentials(true);
    }
}

// 4. Use environment variables for secrets
spring:
  datasource:
    password: ${DB_PASSWORD}

// 5. Enable security headers
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .headers(headers -> headers
                .contentSecurityPolicy(csp -> csp.policyDirectives("default-src 'self'"))
                .frameOptions(frame -> frame.deny())
            )
            .build();
    }
}
```

### Performance Best Practices

```java
// 1. Use lazy loading for relationships
@ManyToOne(fetch = FetchType.LAZY)
private User user;

// 2. Use fetch joins to avoid N+1
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.user.id = :userId")
List<Order> findByUserIdWithItems(@Param("userId") Long userId);

// 3. Use projections for read-only queries
interface UserSummary {
    Long getId();
    String getName();
    String getEmail();
}

List<UserSummary> findAllProjectedBy();

// 4. Enable query caching
@Cacheable("users")
public User findById(Long id) { ... }

// 5. Use batch operations
@Modifying
@Query("UPDATE User u SET u.status = :status WHERE u.id IN :ids")
int bulkUpdateStatus(@Param("ids") List<Long> ids, @Param("status") UserStatus status);

// 6. Configure connection pooling
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
```

---

## Quick Reference - Key Annotations

| Annotation | Purpose | Layer |
|------------|---------|-------|
| `@SpringBootApplication` | Main application class | Application |
| `@RestController` | REST API controller | Web |
| `@RequestMapping` | URL mapping | Web |
| `@GetMapping/@PostMapping` | HTTP method mapping | Web |
| `@PathVariable` | URL path parameter | Web |
| `@RequestParam` | Query parameter | Web |
| `@RequestBody` | Request body binding | Web |
| `@Valid` | Enable validation | Web |
| `@Service` | Business logic component | Service |
| `@Transactional` | Transaction management | Service |
| `@Repository` | Data access component | Repository |
| `@Entity` | JPA entity | Entity |
| `@Autowired` | Dependency injection | All |
| `@Value` | Property injection | All |
| `@ConfigurationProperties` | Type-safe configuration | Configuration |
| `@Profile` | Profile-specific beans | Configuration |
| `@ControllerAdvice` | Global exception handling | Web |
| `@ExceptionHandler` | Exception handler method | Web |
| `@Async` | Async method execution | Service |
| `@Scheduled` | Scheduled task | Service |
| `@Cacheable` | Method result caching | Service |

---

## Additional Resources

- [Official Spring Boot Documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Spring Boot Guides](https://spring.io/guides)
- [Spring Data JPA Reference](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)
- [Spring Security Reference](https://docs.spring.io/spring-security/reference/)
- [Baeldung Spring Tutorials](https://www.baeldung.com/spring-tutorial)
