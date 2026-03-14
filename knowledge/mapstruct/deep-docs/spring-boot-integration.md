# MapStruct Spring Boot Integration

## Overview

This guide covers best practices for integrating MapStruct with Spring Boot projects, including dependency injection, transaction handling, and repository access within mappers.

---

## Project Setup

### Maven Dependencies

```xml
<properties>
    <java.version>17</java.version>
    <mapstruct.version>1.6.2</mapstruct.version>
    <lombok.version>1.18.30</lombok.version>
    <lombok-mapstruct-binding.version>0.2.0</lombok-mapstruct-binding.version>
</properties>

<dependencies>
    <!-- Spring Boot -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- MapStruct -->
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct</artifactId>
        <version>${mapstruct.version}</version>
    </dependency>

    <!-- Lombok -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>${lombok.version}</version>
        <optional>true</optional>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.13.0</version>
            <configuration>
                <source>17</source>
                <target>17</target>
                <annotationProcessorPaths>
                    <!-- IMPORTANT: Order matters! -->
                    <!-- 1. Lombok first -->
                    <path>
                        <groupId>org.projectlombok</groupId>
                        <artifactId>lombok</artifactId>
                        <version>${lombok.version}</version>
                    </path>
                    <!-- 2. MapStruct processor -->
                    <path>
                        <groupId>org.mapstruct</groupId>
                        <artifactId>mapstruct-processor</artifactId>
                        <version>${mapstruct.version}</version>
                    </path>
                    <!-- 3. Binding for compatibility -->
                    <path>
                        <groupId>org.projectlombok</groupId>
                        <artifactId>lombok-mapstruct-binding</artifactId>
                        <version>${lombok-mapstruct-binding.version}</version>
                    </path>
                </annotationProcessorPaths>
                <compilerArgs>
                    <!-- Set default component model to spring -->
                    <arg>-Amapstruct.defaultComponentModel=spring</arg>
                </compilerArgs>
            </configuration>
        </plugin>
    </plugins>
</build>
```

---

## Component Model

### Automatic Spring Bean Registration

```java
// With componentModel = "spring", MapStruct generates a @Component
@Mapper(componentModel = "spring")
public interface UserMapper {
    UserDTO toDTO(User entity);
}

// Generated implementation
@Component
public class UserMapperImpl implements UserMapper {
    @Override
    public UserDTO toDTO(User entity) {
        // Implementation
    }
}
```

### Injecting Mappers in Services

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;
    private final UserMapper userMapper; // Injected by Spring

    @Transactional(readOnly = true)
    public UserDTO findById(Long id) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User", "id", id));
        return userMapper.toDTO(user);
    }

    @Transactional
    public UserDTO create(CreateUserDTO dto) {
        User user = userMapper.toEntity(dto);
        User saved = userRepository.save(user);
        return userMapper.toDTO(saved);
    }

    @Transactional
    public UserDTO update(Long id, UpdateUserDTO dto) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User", "id", id));
        userMapper.updateEntity(dto, user);
        return userMapper.toDTO(user);
    }
}
```

---

## Dependency Injection in Mappers

### Injecting Spring Beans

```java
@Mapper(componentModel = "spring")
public abstract class OrderMapper {

    @Autowired
    protected ProductRepository productRepository;

    @Autowired
    protected CustomerRepository customerRepository;

    @Mapping(target = "products", source = "productIds")
    @Mapping(target = "customer", source = "customerId")
    public abstract Order toEntity(CreateOrderDTO dto);

    // Custom mapping method with repository access
    protected Set<Product> mapProducts(Set<Long> productIds) {
        if (productIds == null || productIds.isEmpty()) {
            return new HashSet<>();
        }
        return new HashSet<>(productRepository.findAllById(productIds));
    }

    protected Customer mapCustomer(Long customerId) {
        if (customerId == null) {
            return null;
        }
        return customerRepository.findById(customerId)
            .orElseThrow(() -> new ResourceNotFoundException("Customer", "id", customerId));
    }
}
```

### Using Other Mappers

```java
@Mapper(componentModel = "spring", uses = {AddressMapper.class, ContactInfoMapper.class})
public interface CustomerMapper {

    // Automatically uses AddressMapper and ContactInfoMapper
    CustomerDTO toDTO(Customer entity);

    Customer toEntity(CreateCustomerDTO dto);
}

@Mapper(componentModel = "spring")
public interface AddressMapper {
    AddressDTO toDTO(Address entity);
    Address toEntity(AddressDTO dto);
}

@Mapper(componentModel = "spring")
public interface ContactInfoMapper {
    ContactInfoDTO toDTO(ContactInfo entity);
    ContactInfo toEntity(ContactInfoDTO dto);
}
```

---

## Transaction Handling

### Mapper Methods and Transactions

```java
@Mapper(componentModel = "spring")
public abstract class UserMapper {

    @Autowired
    protected RoleRepository roleRepository;

    // Simple mapping - no transaction needed
    public abstract UserDTO toDTO(User entity);

    // Mapping with database access - should be called within a transaction
    @Mapping(target = "roles", source = "roleIds")
    public abstract User toEntity(CreateUserDTO dto);

    protected Set<Role> mapRoles(Set<Long> roleIds) {
        // This accesses the database
        // Should be called within an active transaction
        return new HashSet<>(roleRepository.findAllById(roleIds));
    }
}

// Service ensures transaction
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;
    private final UserMapper userMapper;

    @Transactional // Mapper's database access happens within this transaction
    public UserDTO create(CreateUserDTO dto) {
        User user = userMapper.toEntity(dto); // Accesses roleRepository
        User saved = userRepository.save(user);
        return userMapper.toDTO(saved);
    }
}
```

### Lazy Loading Considerations

```java
@Mapper(componentModel = "spring")
public abstract class OrderMapper {

    @Transactional(readOnly = true) // Enable lazy loading
    public OrderDTO toDTOWithDetails(Order order) {
        // Trigger lazy loading before mapping
        order.getItems().size(); // Initialize collection
        order.getCustomer().getName(); // Initialize customer
        return toDTO(order);
    }

    protected abstract OrderDTO toDTO(Order order);
}

// Better approach: Use @EntityGraph or fetch join in repository
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    @EntityGraph(attributePaths = {"items", "customer"})
    Optional<Order> findWithDetailsById(Long id);
}

// Service
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final OrderMapper orderMapper;

    @Transactional(readOnly = true)
    public OrderDTO findById(Long id) {
        Order order = orderRepository.findWithDetailsById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Order", "id", id));
        return orderMapper.toDTO(order); // No lazy loading issues
    }
}
```

---

## Configuration Classes

### Custom MapStruct Configuration

```java
@Configuration
public class MapStructConfig {

    // Register custom mapper configuration if needed
    @Bean
    public MapperConfig mapperConfig() {
        return new MapperConfig() {
            @Override
            public Class<? extends Annotation> annotationType() {
                return MapperConfig.class;
            }

            @Override
            public String componentModel() {
                return "spring";
            }

            @Override
            public MappingInheritanceStrategy mappingInheritanceStrategy() {
                return MappingInheritanceStrategy.AUTO_INHERIT_FROM_CONFIG;
            }

            @Override
            public CollectionMappingStrategy collectionMappingStrategy() {
                return CollectionMappingStrategy.ADDER_PREFERRED;
            }

            @Override
            public NullValuePropertyMappingStrategy nullValuePropertyMappingStrategy() {
                return NullValuePropertyMappingStrategy.IGNORE;
            }

            @Override
            public ReportingPolicy unmappedTargetPolicy() {
                return ReportingPolicy.IGNORE;
            }
        };
    }
}
```

### Shared Mapper Configuration

```java
@MapperConfig(
    componentModel = "spring",
    unmappedTargetPolicy = ReportingPolicy.IGNORE,
    nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE
)
public interface CentralMapperConfig {
}

// Use in mappers
@Mapper(config = CentralMapperConfig.class)
public interface UserMapper {
    UserDTO toDTO(User entity);
}

@Mapper(config = CentralMapperConfig.class)
public interface ProductMapper {
    ProductDTO toDTO(Product entity);
}
```

---

## REST Controller Integration

### Controller Layer

```java
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    @GetMapping("/{id}")
    public ResponseEntity<UserDTO> findById(@PathVariable Long id) {
        return ResponseEntity.ok(userService.findById(id));
    }

    @GetMapping
    public ResponseEntity<List<UserDTO>> findAll(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        return ResponseEntity.ok(userService.findAll(page, size));
    }

    @PostMapping
    public ResponseEntity<UserDTO> create(@Valid @RequestBody CreateUserDTO dto) {
        UserDTO created = userService.create(dto);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }

    @PutMapping("/{id}")
    public ResponseEntity<UserDTO> update(
            @PathVariable Long id,
            @Valid @RequestBody UpdateUserDTO dto) {
        return ResponseEntity.ok(userService.update(id, dto));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

### Service Layer with Mappers

```java
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class UserServiceImpl implements UserService {

    private final UserRepository userRepository;
    private final UserMapper userMapper;

    @Override
    public List<UserDTO> findAll(int page, int size) {
        return userRepository.findAll(PageRequest.of(page, size))
            .map(userMapper::toDTO) // Method reference
            .getContent();
    }

    @Override
    public UserDTO findById(Long id) {
        return userRepository.findById(id)
            .map(userMapper::toDTO)
            .orElseThrow(() -> new ResourceNotFoundException("User", "id", id));
    }

    @Override
    @Transactional
    public UserDTO create(CreateUserDTO dto) {
        User user = userMapper.toEntity(dto);
        User saved = userRepository.save(user);
        return userMapper.toDTO(saved);
    }

    @Override
    @Transactional
    public UserDTO update(Long id, UpdateUserDTO dto) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User", "id", id));
        userMapper.updateEntity(dto, user);
        return userMapper.toDTO(userRepository.save(user));
    }

    @Override
    @Transactional
    public void delete(Long id) {
        if (!userRepository.existsById(id)) {
            throw new ResourceNotFoundException("User", "id", id);
        }
        userRepository.deleteById(id);
    }
}
```

---

## Testing

### Unit Testing Mappers

```java
@SpringBootTest(classes = {UserMapperImpl.class})
class UserMapperTest {

    @Autowired
    private UserMapper userMapper;

    @Test
    void shouldMapUserToDTO() {
        User user = User.builder()
            .id(1L)
            .name("John Doe")
            .email("john@example.com")
            .build();

        UserDTO dto = userMapper.toDTO(user);

        assertThat(dto.getId()).isEqualTo(1L);
        assertThat(dto.getName()).isEqualTo("John Doe");
    }
}
```

### Integration Testing with Service Layer

```java
@SpringBootTest
@Transactional
class UserServiceIntegrationTest {

    @Autowired
    private UserService userService;

    @Autowired
    private UserRepository userRepository;

    @Test
    void shouldCreateUserWithMapper() {
        CreateUserDTO dto = CreateUserDTO.builder()
            .name("Jane Doe")
            .email("jane@example.com")
            .password("password123")
            .build();

        UserDTO created = userService.create(dto);

        assertThat(created.getId()).isNotNull();
        assertThat(created.getName()).isEqualTo("Jane Doe");

        User user = userRepository.findById(created.getId()).orElseThrow();
        assertThat(user.getName()).isEqualTo("Jane Doe");
    }
}
```

---

## Common Patterns

### Pattern 1: Entity to Response DTO

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
    UserDTO toDTO(User entity);
}
```

### Pattern 2: Request DTO to Entity

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    User toEntity(CreateUserDTO dto);
}
```

### Pattern 3: Update Entity from DTO

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
    @BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
    void updateEntity(UpdateUserDTO dto, @MappingTarget User entity);
}
```

### Pattern 4: Collection Mapping

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
    List<UserDTO> toDTOList(List<User> entities);

    default Page<UserDTO> toDTOPage(Page<User> entities) {
        return entities.map(this::toDTO);
    }
}
```

---

## Best Practices

1. ✅ Always use `componentModel = "spring"`
2. ✅ Configure annotation processors in correct order
3. ✅ Call mapper methods within service transactions
4. ✅ Use `@EntityGraph` or fetch joins to avoid lazy loading issues
5. ✅ Exclude audit fields from request DTOs
6. ✅ Use `@MappingTarget` for updates
7. ✅ Test mappers with unit and integration tests
8. ✅ Keep business logic in services, not mappers
9. ✅ Use method references in streams: `.map(mapper::toDTO)`
10. ✅ Handle null values and empty collections

## Anti-Patterns

1. ❌ Complex business logic in mappers
2. ❌ Direct repository access without transactions
3. ❌ Ignoring lazy loading issues
4. ❌ Not testing mapper integration
5. ❌ Forgetting to exclude audit fields
6. ❌ Not using shared configuration
7. ❌ Mixing presentation logic with mapping

---

## References

- [MapStruct Spring Extensions](https://mapstruct.org/documentation/stable/reference/html/#spring)
- [Spring Boot Documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Spring Data JPA](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)
