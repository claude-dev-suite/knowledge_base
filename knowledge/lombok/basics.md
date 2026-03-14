# Project Lombok

> Official Documentation: https://projectlombok.org/features/

## Overview

Project Lombok is a Java library that automatically plugs into your editor and build tools, reducing boilerplate code by generating getters, setters, constructors, builders, loggers, and more at compile time through annotations.

**Maven Dependency:**
```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.30</version>
    <scope>provided</scope>
</dependency>
```

**Gradle Dependency:**
```groovy
compileOnly 'org.projectlombok:lombok:1.18.30'
annotationProcessor 'org.projectlombok:lombok:1.18.30'
```

---

## Core Annotations

### @Getter and @Setter

Generates getter and setter methods for fields.

```java
public class User {
    @Getter @Setter
    private String name;

    @Getter(AccessLevel.PROTECTED)  // Custom access level
    @Setter(AccessLevel.PRIVATE)
    private String password;

    @Getter(lazy = true)  // Lazy initialization
    private final double[] cachedData = expensiveCalculation();
}

// Class-level application
@Getter @Setter
public class Product {
    private Long id;
    private String name;
    private BigDecimal price;
}
```

### @ToString

Generates a `toString()` method.

```java
@ToString
public class Order {
    private Long id;
    private String status;

    @ToString.Exclude  // Exclude from toString
    private String internalCode;
}

@ToString(callSuper = true)  // Include parent class
@ToString(onlyExplicitlyIncluded = true)  // Only include marked fields
public class SpecialOrder extends Order {
    @ToString.Include
    private String priority;

    @ToString.Include(name = "type", rank = 1)  // Custom name and order
    private String orderType;
}
```

### @EqualsAndHashCode

Generates `equals()` and `hashCode()` methods.

```java
@EqualsAndHashCode
public class Customer {
    private Long id;
    private String email;

    @EqualsAndHashCode.Exclude  // Exclude from equality check
    private LocalDateTime lastLogin;
}

@EqualsAndHashCode(onlyExplicitlyIncluded = true)
public class Account {
    @EqualsAndHashCode.Include
    private String accountNumber;  // Only this field used for equality

    private String name;
    private BigDecimal balance;
}

@EqualsAndHashCode(callSuper = true)  // Include parent fields
public class PremiumCustomer extends Customer {
    private String tier;
}
```

---

## Constructor Annotations

### @NoArgsConstructor

Generates a no-argument constructor.

```java
@NoArgsConstructor
public class Config {
    private String setting;
}

@NoArgsConstructor(access = AccessLevel.PROTECTED)  // For JPA entities
public class Entity {
    private Long id;
}

@NoArgsConstructor(force = true)  // Initialize final fields to defaults
public class Immutable {
    private final String name;  // Will be null
    private final int count;    // Will be 0
}
```

### @AllArgsConstructor

Generates a constructor with all fields.

```java
@AllArgsConstructor
public class Event {
    private String name;
    private LocalDateTime timestamp;
    private String payload;
}

@AllArgsConstructor(staticName = "of")  // Static factory method
public class Point {
    private int x;
    private int y;
}
// Usage: Point p = Point.of(10, 20);
```

### @RequiredArgsConstructor

Generates constructor for final and @NonNull fields.

```java
@RequiredArgsConstructor
public class Service {
    private final Repository repository;  // Included
    private final Mapper mapper;          // Included
    private String optionalField;         // Not included
}

@RequiredArgsConstructor(staticName = "create")
public class Handler {
    private final @NonNull Processor processor;  // @NonNull also included
}
```

---

## Compound Annotations

### @Data

Combines `@Getter`, `@Setter`, `@ToString`, `@EqualsAndHashCode`, and `@RequiredArgsConstructor`.

```java
@Data
public class Person {
    private final Long id;      // Getter only (final)
    private String name;        // Getter + Setter
    private String email;       // Getter + Setter
    // Constructor: new Person(id)
}
```

### @Value

Creates an immutable class: all fields are `private final`, no setters, class is `final`.

```java
@Value
public class ImmutableConfig {
    String host;        // Implicitly private final
    int port;
    String protocol;
}

// Equivalent to:
public final class ImmutableConfig {
    private final String host;
    private final int port;
    private final String protocol;
    // All-args constructor, getters, equals, hashCode, toString
}
```

---

## Builder Pattern

### @Builder

Implements the builder pattern.

```java
@Builder
public class Request {
    private String url;
    private String method;
    private Map<String, String> headers;
    private String body;
}

// Usage:
Request request = Request.builder()
    .url("https://api.example.com")
    .method("POST")
    .body("{\"key\":\"value\"}")
    .build();
```

### Builder with Defaults

```java
@Builder
public class ServerConfig {
    @Builder.Default
    private String host = "localhost";

    @Builder.Default
    private int port = 8080;

    @Builder.Default
    private boolean ssl = false;

    @Builder.Default
    private List<String> allowedOrigins = new ArrayList<>();
}

// Usage: defaults are applied
ServerConfig config = ServerConfig.builder().build();
// host="localhost", port=8080, ssl=false
```

### Builder Customization

```java
@Builder(
    builderMethodName = "create",      // Custom builder method name
    buildMethodName = "execute",       // Custom build method name
    builderClassName = "Creator",      // Custom builder class name
    toBuilder = true                   // Enable toBuilder()
)
public class Task {
    private String name;
    private String description;
}

// Usage:
Task task = Task.create()
    .name("My Task")
    .execute();

// Copy and modify with toBuilder
Task modified = task.toBuilder()
    .description("Updated description")
    .execute();
```

### @Singular for Collections

```java
@Builder
public class Email {
    private String subject;
    private String body;

    @Singular  // Generates singular add method
    private List<String> recipients;

    @Singular("header")  // Custom singular name
    private Map<String, String> headers;
}

// Usage:
Email email = Email.builder()
    .subject("Hello")
    .body("Content")
    .recipient("user1@example.com")    // Add one at a time
    .recipient("user2@example.com")
    .header("X-Priority", "high")      // Singular name for map
    .header("X-Sender", "system")
    .build();
```

### @SuperBuilder for Inheritance

Standard `@Builder` does not support inheritance. Use `@SuperBuilder` instead.

```java
@SuperBuilder
public class Animal {
    private String name;
    private int age;
}

@SuperBuilder
public class Dog extends Animal {
    private String breed;
    private boolean trained;
}

// Usage:
Dog dog = Dog.builder()
    .name("Buddy")        // From parent
    .age(3)               // From parent
    .breed("Labrador")    // From child
    .trained(true)        // From child
    .build();
```

**SuperBuilder with toBuilder:**
```java
@SuperBuilder(toBuilder = true)
public class Vehicle {
    private String brand;
    private String model;
}

@SuperBuilder(toBuilder = true)
public class Car extends Vehicle {
    private int doors;
    private String fuelType;
}

Car car = Car.builder().brand("Toyota").model("Camry").doors(4).build();
Car modified = car.toBuilder().fuelType("Hybrid").build();
```

---

## Additional Annotations

### @With

Creates immutable "wither" methods (copy with one field changed).

```java
@Value
@With
public class Point {
    int x;
    int y;
}

Point p1 = new Point(10, 20);
Point p2 = p1.withX(15);  // New instance: (15, 20)
Point p3 = p1.withY(25);  // New instance: (10, 25)
```

### @Accessors

Customizes getter/setter generation.

```java
@Data
@Accessors(fluent = true)  // No get/set prefix, returns this
public class FluentBean {
    private String name;
    private int age;
}
// Usage: bean.name("John").age(30); String n = bean.name();

@Data
@Accessors(chain = true)  // Setters return this
public class ChainBean {
    private String name;
    private int age;
}
// Usage: bean.setName("John").setAge(30);

@Data
@Accessors(prefix = "m")  // Strip prefix from method names
public class PrefixedBean {
    private String mName;   // getName(), setName()
    private int mAge;       // getAge(), setAge()
}
```

---

## Logging Annotations

Lombok provides annotations for various logging frameworks.

```java
@Slf4j  // SLF4J (most common)
public class UserService {
    public void process() {
        log.debug("Debug message");
        log.info("Info message with param: {}", param);
        log.error("Error occurred", exception);
    }
}

@Log4j2  // Apache Log4j 2
public class OrderService {
    public void process() {
        log.info("Processing order");
    }
}

@Log  // java.util.logging
@CommonsLog  // Apache Commons Logging
@JBossLog  // JBoss Logging
@Flogger  // Google Flogger
@CustomLog  // Custom logger (requires lombok.config)
```

**Custom Logger Configuration (lombok.config):**
```properties
lombok.log.custom.declaration = com.example.MyLogger my.LoggerFactory.create(TYPE)
```

---

## Utility Annotations

### @Cleanup

Automatic resource cleanup.

```java
public void copyFile(String src, String dest) throws IOException {
    @Cleanup InputStream in = new FileInputStream(src);
    @Cleanup OutputStream out = new FileOutputStream(dest);

    byte[] buffer = new byte[1024];
    int len;
    while ((len = in.read(buffer)) != -1) {
        out.write(buffer, 0, len);
    }
}
// Streams automatically closed, even on exception
```

### @SneakyThrows

Throws checked exceptions without declaring them.

```java
@SneakyThrows  // All checked exceptions
public String readFile(String path) {
    return new String(Files.readAllBytes(Paths.get(path)));
}

@SneakyThrows(IOException.class)  // Specific exception
public void writeFile(String path, String content) {
    Files.write(Paths.get(path), content.getBytes());
}
```

**Warning:** Use sparingly. Can make error handling unclear.

### @Synchronized

Safer than synchronized keyword.

```java
public class Counter {
    private final Object readLock = new Object();

    @Synchronized  // Uses $lock object (auto-created)
    public void increment() {
        count++;
    }

    @Synchronized("readLock")  // Uses specified lock
    public int getCount() {
        return count;
    }
}
```

### @NonNull

Generates null checks.

```java
public class Validator {
    public void process(@NonNull String input) {
        // Throws NullPointerException if input is null
        System.out.println(input.length());
    }
}

@RequiredArgsConstructor
public class Service {
    private final @NonNull Repository repo;  // Constructor includes null check
}
```

---

## JPA Entity Patterns

### Basic Entity

```java
@Entity
@Table(name = "users")
@Data
@NoArgsConstructor(access = AccessLevel.PROTECTED)  // JPA requirement
@AllArgsConstructor
@Builder
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String username;

    @Column(nullable = false)
    private String email;

    @Builder.Default
    @Enumerated(EnumType.STRING)
    private UserStatus status = UserStatus.ACTIVE;

    @Builder.Default
    private LocalDateTime createdAt = LocalDateTime.now();
}
```

### Entity with Relationships

```java
@Entity
@Data
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@AllArgsConstructor
@Builder
@EqualsAndHashCode(onlyExplicitlyIncluded = true)  // Avoid lazy loading issues
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @EqualsAndHashCode.Include
    private Long id;

    private String orderNumber;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")
    @ToString.Exclude  // Prevent lazy loading in toString
    private Customer customer;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    @ToString.Exclude
    @Builder.Default
    private List<OrderItem> items = new ArrayList<>();

    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this);
    }
}
```

### Auditing with Lombok

```java
@MappedSuperclass
@Data
@NoArgsConstructor
@AllArgsConstructor
@SuperBuilder
public abstract class BaseEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    @Version
    private Long version;
}

@Entity
@Data
@NoArgsConstructor
@AllArgsConstructor
@SuperBuilder
@EqualsAndHashCode(callSuper = true, onlyExplicitlyIncluded = true)
@ToString(callSuper = true)
public class Product extends BaseEntity {
    @EqualsAndHashCode.Include
    private String sku;
    private String name;
    private BigDecimal price;
}
```

---

## IDE Setup

### IntelliJ IDEA

1. **Install Plugin:**
   - Go to `File > Settings > Plugins`
   - Search for "Lombok"
   - Install and restart IDE

2. **Enable Annotation Processing:**
   - Go to `File > Settings > Build, Execution, Deployment > Compiler > Annotation Processors`
   - Check "Enable annotation processing"

### VS Code

1. **Install Extension:**
   - Install "Lombok Annotations Support for VS Code" extension
   - Or use "Extension Pack for Java" which includes Lombok support

2. **Ensure Java Extension Pack is configured correctly**

### Eclipse

1. **Run Lombok Installer:**
   ```bash
   java -jar lombok.jar
   ```
2. Select Eclipse installation and install

---

## Configuration (lombok.config)

Create `lombok.config` in project root for project-wide settings.

```properties
# Stop looking for config files in parent directories
config.stopBubbling = true

# Disable specific features
lombok.getter.flagUsage = warning
lombok.setter.flagUsage = warning
lombok.data.flagUsage = error  # Forbid @Data

# Add @Generated annotation to generated code
lombok.addLombokGeneratedAnnotation = true

# Default field annotation for @NonNull
lombok.nonNull.exceptionType = IllegalArgumentException

# Builder configuration
lombok.builder.className = *Builder
lombok.singular.auto = true

# Accessors configuration
lombok.accessors.chain = true
lombok.accessors.fluent = false

# Logging
lombok.log.fieldName = logger
lombok.log.fieldIsStatic = true

# ToString configuration
lombok.toString.includeFieldNames = true
lombok.toString.doNotUseGetters = false

# EqualsAndHashCode
lombok.equalsAndHashCode.doNotUseGetters = false
lombok.equalsAndHashCode.callSuper = warn

# Copy annotations to generated code
lombok.copyableAnnotations += org.springframework.beans.factory.annotation.Qualifier
lombok.copyableAnnotations += org.springframework.beans.factory.annotation.Value
```

---

## Delombok

Convert Lombok-annotated code to vanilla Java.

### Maven

```xml
<plugin>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok-maven-plugin</artifactId>
    <version>1.18.20.0</version>
    <executions>
        <execution>
            <phase>generate-sources</phase>
            <goals>
                <goal>delombok</goal>
            </goals>
            <configuration>
                <sourceDirectory>src/main/java</sourceDirectory>
                <outputDirectory>${project.build.directory}/delombok</outputDirectory>
                <addOutputDirectory>false</addOutputDirectory>
            </configuration>
        </execution>
    </executions>
</plugin>
```

### Command Line

```bash
java -jar lombok.jar delombok src/main/java -d target/delombok
```

### Use Cases for Delombok

- Generating Javadoc
- Code review (seeing generated code)
- Migration away from Lombok
- Debugging issues

---

## Best Practices

1. **Use @RequiredArgsConstructor for Dependency Injection**
   ```java
   @Service
   @RequiredArgsConstructor
   public class UserService {
       private final UserRepository userRepository;
       private final PasswordEncoder passwordEncoder;
   }
   ```

2. **Prefer @Value for DTOs and Immutable Objects**
   ```java
   @Value
   @Builder
   public class UserResponse {
       Long id;
       String name;
       String email;
   }
   ```

3. **Use @EqualsAndHashCode.Include for JPA Entities**
   ```java
   @Entity
   @Data
   @EqualsAndHashCode(onlyExplicitlyIncluded = true)
   public class Entity {
       @Id
       @EqualsAndHashCode.Include
       private Long id;
   }
   ```

4. **Always Use @Builder.Default for Non-Null Collections**
   ```java
   @Builder
   public class Config {
       @Builder.Default
       private List<String> items = new ArrayList<>();
   }
   ```

5. **Exclude Lazy-Loaded Fields from ToString**
   ```java
   @ToString.Exclude
   @ManyToOne(fetch = FetchType.LAZY)
   private Parent parent;
   ```

6. **Use @SuperBuilder for Class Hierarchies**
   ```java
   @SuperBuilder
   public abstract class BaseRequest { }

   @SuperBuilder
   public class CreateUserRequest extends BaseRequest { }
   ```

---

## Common Pitfalls

### 1. @Data with JPA Entities

**Problem:** `@Data` generates `equals`/`hashCode` using all fields, causing issues with lazy loading.

**Solution:**
```java
@Entity
@Getter @Setter
@NoArgsConstructor
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
public class Entity {
    @Id
    @EqualsAndHashCode.Include
    private Long id;
}
```

### 2. Builder.Default Not Working

**Problem:** Default values not applied when using builder.

**Wrong:**
```java
@Builder
public class Config {
    private List<String> items = new ArrayList<>();  // Ignored by builder!
}
```

**Correct:**
```java
@Builder
public class Config {
    @Builder.Default
    private List<String> items = new ArrayList<>();
}
```

### 3. Circular References in ToString

**Problem:** `StackOverflowError` from bidirectional relationships.

**Solution:**
```java
@Entity
@Data
public class Parent {
    @OneToMany(mappedBy = "parent")
    @ToString.Exclude
    private List<Child> children;
}
```

### 4. @Builder with Inheritance

**Problem:** `@Builder` does not work with inheritance.

**Wrong:**
```java
@Builder
public class Parent { }

@Builder
public class Child extends Parent { }  // Compilation error
```

**Correct:**
```java
@SuperBuilder
public class Parent { }

@SuperBuilder
public class Child extends Parent { }
```

### 5. @RequiredArgsConstructor with @Value

**Problem:** Using both creates duplicate constructors.

**Solution:** `@Value` already includes `@AllArgsConstructor`, so just use `@Value`:
```java
@Value
public class Immutable {
    String name;
    int value;
}
```

### 6. Null Collections from Builder

**Problem:** Collections are null when not set in builder.

**Solution:**
```java
@Builder
public class Request {
    @Builder.Default
    @Singular
    private List<String> tags = new ArrayList<>();
}
```

### 7. IDE Not Recognizing Generated Methods

**Problem:** IDE shows errors for Lombok-generated methods.

**Solution:**
- Install Lombok plugin for your IDE
- Enable annotation processing
- Rebuild the project

### 8. Jackson Deserialization Issues

**Problem:** Jackson cannot deserialize `@Value` classes.

**Solution:**
```java
@Value
@Builder
@Jacksonized  // Lombok 1.18.14+
public class Response {
    String status;
    String message;
}
```

Or manually configure:
```java
@Value
@Builder
@JsonDeserialize(builder = Response.ResponseBuilder.class)
public class Response {
    String status;
    String message;

    @JsonPOJOBuilder(withPrefix = "")
    public static class ResponseBuilder { }
}
```
