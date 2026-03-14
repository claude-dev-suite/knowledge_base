# MapStruct Advanced Patterns

## Table of Contents
- [Complex Nested Mapping](#complex-nested-mapping)
- [Circular Dependencies](#circular-dependencies)
- [Custom Qualifiers](#custom-qualifiers)
- [Builder Pattern Integration](#builder-pattern-integration)
- [Performance Optimization](#performance-optimization)
- [Error Handling](#error-handling)
- [Testing Mappers](#testing-mappers)

---

## Complex Nested Mapping

### Multi-Level Nesting

When dealing with deeply nested objects, MapStruct can handle multiple levels automatically:

```java
// Domain model
@Entity
public class Order {
    private Long id;
    private Customer customer;
    private List<OrderItem> items;
    private Address shippingAddress;
}

@Entity
public class Customer {
    private Long id;
    private String name;
    private ContactInfo contactInfo;
}

@Entity
public class ContactInfo {
    private String email;
    private String phone;
    private Address address;
}

// DTO structure
public class OrderDTO {
    private Long id;
    private String customerName;
    private String customerEmail;
    private String customerPhone;
    private List<OrderItemDTO> items;
    private AddressDTO shippingAddress;
}

// Mapper with flattening
@Mapper(componentModel = "spring", uses = {OrderItemMapper.class, AddressMapper.class})
public interface OrderMapper {

    @Mapping(source = "customer.name", target = "customerName")
    @Mapping(source = "customer.contactInfo.email", target = "customerEmail")
    @Mapping(source = "customer.contactInfo.phone", target = "customerPhone")
    OrderDTO toDTO(Order order);

    @Mapping(target = "customer.name", source = "customerName")
    @Mapping(target = "customer.contactInfo.email", source = "customerEmail")
    @Mapping(target = "customer.contactInfo.phone", source = "customerPhone")
    Order toEntity(OrderDTO dto);
}
```

### Conditional Nested Mapping

```java
@Mapper(componentModel = "spring")
public abstract class UserMapper {

    @Mapping(target = "profileDTO", source = "profile")
    public abstract UserDTO toDTO(User user);

    @Condition
    public boolean isProfilePresent(Profile profile) {
        return profile != null && profile.isActive();
    }

    ProfileDTO mapProfile(Profile profile) {
        if (!isProfilePresent(profile)) {
            return null;
        }
        return profileMapper.toDTO(profile);
    }
}
```

---

## Circular Dependencies

### Handling Bidirectional Relationships

When entities reference each other, special handling is needed:

```java
@Entity
public class Department {
    private Long id;
    private String name;
    @OneToMany(mappedBy = "department")
    private List<Employee> employees;
}

@Entity
public class Employee {
    private Long id;
    private String name;
    @ManyToOne
    private Department department;
}

// Solution 1: Context-based mapping
@Mapper(componentModel = "spring")
public abstract class DepartmentMapper {

    @Context
    protected CycleAvoidingMappingContext context;

    public abstract DepartmentDTO toDTO(Department entity, @Context CycleAvoidingMappingContext context);

    @AfterMapping
    protected void handleCircular(Department entity, @MappingTarget DepartmentDTO dto,
                                   @Context CycleAvoidingMappingContext context) {
        context.storeMappedInstance(entity, dto);
    }
}

// CycleAvoidingMappingContext
public class CycleAvoidingMappingContext {
    private Map<Object, Object> knownInstances = new IdentityHashMap<>();

    @BeforeMapping
    public <T> T getMappedInstance(Object source, @TargetType Class<T> targetType) {
        return (T) knownInstances.get(source);
    }

    @BeforeMapping
    public void storeMappedInstance(Object source, @MappingTarget Object target) {
        knownInstances.put(source, target);
    }
}

// Solution 2: Separate mappings with @ToString.Exclude
@Data
@ToString(exclude = "employees") // Lombok
public class DepartmentDTO {
    private Long id;
    private String name;
    private List<EmployeeSummaryDTO> employees; // Summary without department
}

@Data
public class EmployeeSummaryDTO {
    private Long id;
    private String name;
    // No department reference
}

@Mapper(componentModel = "spring", uses = EmployeeMapper.class)
public interface DepartmentMapper {
    @Mapping(target = "employees", qualifiedByName = "toSummary")
    DepartmentDTO toDTO(Department entity);
}

@Mapper(componentModel = "spring")
public interface EmployeeMapper {

    @Named("toSummary")
    EmployeeSummaryDTO toSummary(Employee entity);

    @Mapping(source = "department.id", target = "departmentId")
    @Mapping(target = "department", ignore = true)
    EmployeeDTO toDTO(Employee entity);
}
```

---

## Custom Qualifiers

### Named Qualifiers for Disambiguation

```java
// Define custom qualifier
@Qualifier
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.CLASS)
public @interface ToCurrency {
}

@Qualifier
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.CLASS)
public @interface ToPercentage {
}

// Mapper with qualifiers
@Mapper(componentModel = "spring")
public abstract class FinancialMapper {

    @Mapping(source = "amount", target = "formattedAmount", qualifiedBy = ToCurrency.class)
    @Mapping(source = "discount", target = "discountText", qualifiedBy = ToPercentage.class)
    public abstract InvoiceDTO toDTO(Invoice invoice);

    @ToCurrency
    protected String formatAsCurrency(BigDecimal amount) {
        return NumberFormat.getCurrencyInstance(Locale.US).format(amount);
    }

    @ToPercentage
    protected String formatAsPercentage(BigDecimal value) {
        return NumberFormat.getPercentInstance().format(value);
    }
}
```

### Qualifier with Parameters

```java
@Qualifier
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.CLASS)
public @interface DateFormat {
    String pattern() default "dd/MM/yyyy";
}

@Mapper(componentModel = "spring")
public abstract class EventMapper {

    @Mapping(source = "startDate", target = "startDateFormatted", qualifiedBy = DateFormat.class)
    public abstract EventDTO toDTO(Event event);

    @DateFormat(pattern = "dd/MM/yyyy HH:mm")
    protected String formatDateTime(LocalDateTime date) {
        return date.format(DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm"));
    }
}
```

---

## Builder Pattern Integration

### Working with Immutable DTOs

```java
// Immutable DTO with builder
@Value
@Builder
public class UserDTO {
    Long id;
    String name;
    String email;
    UserStatus status;
    LocalDateTime createdAt;
}

// Mapper automatically uses builder
@Mapper(componentModel = "spring")
public interface UserMapper {

    // MapStruct detects @Builder and uses it
    UserDTO toDTO(User entity);

    // For entity (mutable), normal setters
    User toEntity(CreateUserDTO dto);
}

// Custom builder configuration
@Mapper(componentModel = "spring",
        builder = @Builder(disableBuilder = false))
public interface ProductMapper {
    ProductDTO toDTO(Product entity);
}
```

---

## Performance Optimization

### Lazy Loading Considerations

```java
@Mapper(componentModel = "spring")
public abstract class OrderMapper {

    // Avoid triggering lazy loading
    @Mapping(target = "customerName", expression = "java(order.getCustomer() != null ? order.getCustomer().getName() : null)")
    @Mapping(target = "itemCount", expression = "java(order.getItems() != null ? order.getItems().size() : 0)")
    public abstract OrderSummaryDTO toSummary(Order order);

    // For full DTO, ensure entities are fetched
    @Transactional(readOnly = true)
    public OrderDTO toDTOWithItems(Order order) {
        // Hibernate.initialize(order.getItems()); // If needed
        return toDTO(order);
    }

    protected abstract OrderDTO toDTO(Order order);
}
```

### Batch Mapping Optimization

```java
@Mapper(componentModel = "spring")
public abstract class UserMapper {

    @Autowired
    protected RoleRepository roleRepository;

    // Efficient batch mapping
    public List<UserDTO> toDTOList(List<User> users) {
        // Pre-fetch all roles in one query
        Set<Long> roleIds = users.stream()
            .flatMap(u -> u.getRoleIds().stream())
            .collect(Collectors.toSet());

        Map<Long, Role> roleMap = roleRepository.findAllById(roleIds)
            .stream()
            .collect(Collectors.toMap(Role::getId, Function.identity()));

        return users.stream()
            .map(user -> toDTOWithRoleMap(user, roleMap))
            .collect(Collectors.toList());
    }

    protected UserDTO toDTOWithRoleMap(User user, Map<Long, Role> roleMap) {
        UserDTO dto = toDTO(user);
        dto.setRoles(user.getRoleIds().stream()
            .map(roleMap::get)
            .filter(Objects::nonNull)
            .map(this::toRoleDTO)
            .collect(Collectors.toSet()));
        return dto;
    }

    protected abstract UserDTO toDTO(User user);
    protected abstract RoleDTO toRoleDTO(Role role);
}
```

---

## Error Handling

### Validation in Mappers

```java
@Mapper(componentModel = "spring")
public abstract class UserMapper {

    @Mapping(target = "email", qualifiedByName = "validateEmail")
    public abstract User toEntity(CreateUserDTO dto);

    @Named("validateEmail")
    protected String validateEmail(String email) {
        if (email == null || !email.matches("^[A-Za-z0-9+_.-]+@(.+)$")) {
            throw new ValidationException("Invalid email format: " + email);
        }
        return email.toLowerCase();
    }

    @AfterMapping
    protected void validate(@MappingTarget User user) {
        if (user.getName() == null || user.getName().trim().isEmpty()) {
            throw new ValidationException("Name cannot be empty");
        }
        if (user.getAge() != null && user.getAge() < 0) {
            throw new ValidationException("Age cannot be negative");
        }
    }
}
```

### Null Safety

```java
@Mapper(componentModel = "spring",
        nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE,
        nullValueCheckStrategy = NullValueCheckStrategy.ALWAYS)
public abstract class SafeMapper {

    @Mapping(target = "fullName",
             expression = "java(mapFullName(user.getFirstName(), user.getLastName()))")
    public abstract UserDTO toDTO(User user);

    protected String mapFullName(String firstName, String lastName) {
        StringBuilder fullName = new StringBuilder();
        if (firstName != null && !firstName.trim().isEmpty()) {
            fullName.append(firstName);
        }
        if (lastName != null && !lastName.trim().isEmpty()) {
            if (fullName.length() > 0) fullName.append(" ");
            fullName.append(lastName);
        }
        return fullName.length() > 0 ? fullName.toString() : null;
    }
}
```

---

## Testing Mappers

### Unit Tests

```java
@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = {UserMapperImpl.class, RoleMapperImpl.class})
class UserMapperTest {

    @Autowired
    private UserMapper userMapper;

    @Test
    void shouldMapUserToDTO() {
        // Given
        User user = User.builder()
            .id(1L)
            .name("John Doe")
            .email("john@example.com")
            .role(UserRole.ADMIN)
            .createdAt(LocalDateTime.now())
            .build();

        // When
        UserDTO dto = userMapper.toDTO(user);

        // Then
        assertThat(dto.getId()).isEqualTo(1L);
        assertThat(dto.getName()).isEqualTo("John Doe");
        assertThat(dto.getEmail()).isEqualTo("john@example.com");
        assertThat(dto.getRole()).isEqualTo(UserRole.ADMIN);
        assertThat(dto.getCreatedAt()).isNotNull();
    }

    @Test
    void shouldMapDTOToUser() {
        // Given
        CreateUserDTO dto = CreateUserDTO.builder()
            .name("Jane Doe")
            .email("jane@example.com")
            .password("password123")
            .build();

        // When
        User user = userMapper.toEntity(dto);

        // Then
        assertThat(user.getId()).isNull(); // Not mapped
        assertThat(user.getName()).isEqualTo("Jane Doe");
        assertThat(user.getEmail()).isEqualTo("jane@example.com");
        assertThat(user.getCreatedAt()).isNull(); // Not mapped
    }

    @Test
    void shouldUpdateEntity() {
        // Given
        User existingUser = User.builder()
            .id(1L)
            .name("Old Name")
            .email("old@example.com")
            .createdAt(LocalDateTime.now().minusDays(30))
            .build();

        UpdateUserDTO dto = UpdateUserDTO.builder()
            .name("New Name")
            .email(null) // Should not update
            .build();

        // When
        userMapper.updateEntity(dto, existingUser);

        // Then
        assertThat(existingUser.getName()).isEqualTo("New Name");
        assertThat(existingUser.getEmail()).isEqualTo("old@example.com"); // Unchanged
        assertThat(existingUser.getCreatedAt()).isNotNull(); // Unchanged
    }

    @Test
    void shouldMapList() {
        // Given
        List<User> users = List.of(
            User.builder().id(1L).name("User 1").build(),
            User.builder().id(2L).name("User 2").build()
        );

        // When
        List<UserDTO> dtos = userMapper.toDTOList(users);

        // Then
        assertThat(dtos).hasSize(2);
        assertThat(dtos.get(0).getName()).isEqualTo("User 1");
        assertThat(dtos.get(1).getName()).isEqualTo("User 2");
    }
}
```

### Integration Tests

```java
@SpringBootTest
@Transactional
class UserMapperIntegrationTest {

    @Autowired
    private UserMapper userMapper;

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private EntityManager entityManager;

    @Test
    void shouldMapWithRelationships() {
        // Given
        Department dept = new Department("Engineering");
        entityManager.persist(dept);

        User user = User.builder()
            .name("John Doe")
            .email("john@example.com")
            .department(dept)
            .build();
        userRepository.save(user);
        entityManager.flush();
        entityManager.clear();

        // When
        User foundUser = userRepository.findById(user.getId()).orElseThrow();
        UserDTO dto = userMapper.toDTO(foundUser);

        // Then
        assertThat(dto.getDepartmentName()).isEqualTo("Engineering");
    }
}
```

---

## Best Practices Summary

1. **Order annotation processors correctly**: Lombok → MapStruct → Binding
2. **Use componentModel = "spring"** for dependency injection
3. **Exclude audit fields** (id, createdAt, updatedAt) from request mappings
4. **Use @MappingTarget** for partial updates with IGNORE strategy
5. **Leverage qualifiers** for complex type conversions
6. **Handle circular dependencies** with context or simplified DTOs
7. **Pre-fetch collections** to avoid N+1 queries
8. **Test mappers thoroughly** with unit and integration tests
9. **Use builders** for immutable DTOs
10. **Add validation** in @AfterMapping hooks when needed

## Common Pitfalls

- ❌ Not configuring annotation processor order correctly
- ❌ Forgetting `unmappedTargetPolicy` leading to build warnings
- ❌ Triggering lazy loading during mapping
- ❌ Not handling null values properly
- ❌ Circular dependencies without proper handling
- ❌ Complex logic in mappers (move to services instead)
- ❌ Not testing edge cases (nulls, empty collections)

## References

- [MapStruct Reference Documentation](https://mapstruct.org/documentation/stable/reference/html/)
- [Spring Integration](https://mapstruct.org/documentation/stable/reference/html/#spring)
- [Advanced Mapping](https://mapstruct.org/documentation/stable/reference/html/#mapping-complex-types)
- [Custom Mappers](https://mapstruct.org/documentation/stable/reference/html/#custom-mapper)
