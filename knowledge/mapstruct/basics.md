# MapStruct with Lombok

## Basic Mapper

```java
@Mapper(componentModel = "spring", unmappedTargetPolicy = ReportingPolicy.IGNORE)
public interface UserMapper {

    UserResponse toResponse(User user);

    List<UserResponse> toResponseList(List<User> users);

    @Mapping(target = "id", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    User toEntity(CreateUserRequest dto);
}
```

## Update Mapping

```java
@Mapper(componentModel = "spring", unmappedTargetPolicy = ReportingPolicy.IGNORE)
public interface UserMapper {

    // Partial update - ignores null values
    @BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
    void updateEntity(UpdateUserRequest dto, @MappingTarget User user);

    // Full update - sets all fields
    void updateEntityFull(UpdateUserRequest dto, @MappingTarget User user);
}
```

## Field Mapping

```java
@Mapper(componentModel = "spring")
public interface OrderMapper {

    // Different field names
    @Mapping(source = "customer.name", target = "customerName")
    @Mapping(source = "customer.email", target = "customerEmail")
    @Mapping(source = "items", target = "orderItems")
    OrderResponse toResponse(Order order);

    // Constant values
    @Mapping(target = "status", constant = "PENDING")
    Order toEntity(CreateOrderRequest dto);

    // Expression
    @Mapping(target = "fullName", expression = "java(user.getFirstName() + \" \" + user.getLastName())")
    UserResponse toResponse(User user);

    // Date formatting
    @Mapping(source = "createdAt", target = "createdDate", dateFormat = "yyyy-MM-dd")
    OrderResponse toResponse(Order order);
}
```

## Nested Objects

```java
@Mapper(componentModel = "spring", uses = {AddressMapper.class, ItemMapper.class})
public interface OrderMapper {

    // Uses AddressMapper for address field
    // Uses ItemMapper for items collection
    OrderResponse toResponse(Order order);
}

@Mapper(componentModel = "spring")
public interface AddressMapper {
    AddressResponse toResponse(Address address);
}
```

## Enum Mapping

```java
@Mapper(componentModel = "spring")
public interface StatusMapper {

    @ValueMappings({
        @ValueMapping(source = "ACTIVE", target = "ENABLED"),
        @ValueMapping(source = "INACTIVE", target = "DISABLED"),
        @ValueMapping(source = MappingConstants.ANY_REMAINING, target = "UNKNOWN")
    })
    ExternalStatus toExternalStatus(InternalStatus status);
}
```

## Custom Methods

```java
@Mapper(componentModel = "spring")
public abstract class UserMapper {

    @Autowired
    protected RoleRepository roleRepository;

    public abstract UserResponse toResponse(User user);

    @Mapping(target = "roles", source = "roleIds")
    public abstract User toEntity(CreateUserRequest dto);

    // Custom mapping method
    protected Set<Role> mapRoles(Set<Long> roleIds) {
        if (roleIds == null) return new HashSet<>();
        return roleIds.stream()
            .map(id -> roleRepository.findById(id).orElseThrow())
            .collect(Collectors.toSet());
    }
}
```

## Collection Mapping

```java
@Mapper(componentModel = "spring")
public interface ProductMapper {

    ProductResponse toResponse(Product product);

    List<ProductResponse> toResponseList(List<Product> products);

    Set<ProductResponse> toResponseSet(Set<Product> products);

    // Page mapping
    default Page<ProductResponse> toResponsePage(Page<Product> products) {
        return products.map(this::toResponse);
    }
}
```

## After/Before Mapping

```java
@Mapper(componentModel = "spring")
public abstract class UserMapper {

    @AfterMapping
    protected void afterMapping(@MappingTarget UserResponse response, User user) {
        response.setDisplayName(user.getFirstName() + " " + user.getLastName().charAt(0) + ".");
    }

    @BeforeMapping
    protected void beforeMapping(CreateUserRequest dto) {
        dto.setEmail(dto.getEmail().toLowerCase().trim());
    }
}
```

## Maven Configuration

```xml
<properties>
    <mapstruct.version>1.6.2</mapstruct.version>
    <lombok.version>1.18.34</lombok.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct</artifactId>
        <version>${mapstruct.version}</version>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
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
                <compilerArgs>
                    <arg>-Amapstruct.defaultComponentModel=spring</arg>
                </compilerArgs>
            </configuration>
        </plugin>
    </plugins>
</build>
```

## Key Annotations

| Annotation | Purpose |
|------------|---------|
| `@Mapper` | Define mapper interface |
| `@Mapping` | Field mapping |
| `@MappingTarget` | Update existing object |
| `@BeanMapping` | Bean-level settings |
| `@AfterMapping` | Post-processing |
| `@BeforeMapping` | Pre-processing |
