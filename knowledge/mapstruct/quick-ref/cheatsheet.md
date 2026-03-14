# MapStruct Quick Reference

## Basic Mapper

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
    UserDTO toDTO(User entity);
    User toEntity(UserDTO dto);
    List<UserDTO> toDTOList(List<User> entities);
}
```

## Field Mapping

```java
// Ignore field
@Mapping(target = "id", ignore = true)

// Different names
@Mapping(source = "fullName", target = "name")

// Nested
@Mapping(source = "address.city", target = "cityName")

// Constant
@Mapping(target = "status", constant = "ACTIVE")

// Expression
@Mapping(target = "fullName", expression = "java(entity.getFirstName() + \" \" + entity.getLastName())")

// Date format
@Mapping(source = "createdAt", target = "createdDate", dateFormat = "dd/MM/yyyy")
```

## Update Mapping

```java
// Partial update (ignore nulls)
@BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
void updateEntity(UpdateDTO dto, @MappingTarget User entity);
```

## Enum Mapping

```java
@ValueMappings({
    @ValueMapping(source = "ACTIVE", target = "ENABLED"),
    @ValueMapping(source = "INACTIVE", target = "DISABLED"),
    @ValueMapping(source = MappingConstants.ANY_REMAINING, target = "UNKNOWN")
})
ExternalStatus toExternalStatus(InternalStatus status);
```

## Nested Mappers

```java
@Mapper(componentModel = "spring", uses = {AddressMapper.class})
public interface UserMapper {
    UserDTO toDTO(User entity); // Uses AddressMapper for address field
}
```

## Custom Logic

```java
@Mapper(componentModel = "spring")
public abstract class UserMapper {

    @Autowired
    protected RoleRepository roleRepository;

    public abstract UserDTO toDTO(User entity);

    @Mapping(target = "roles", source = "roleIds")
    public abstract User toEntity(CreateUserDTO dto);

    protected Set<Role> mapRoles(Set<Long> roleIds) {
        return roleIds.stream()
            .map(roleRepository::findById)
            .filter(Optional::isPresent)
            .map(Optional::get)
            .collect(Collectors.toSet());
    }
}
```

## After/Before Mapping

```java
@AfterMapping
protected void afterMapping(@MappingTarget UserDTO dto, User entity) {
    dto.setDisplayName(entity.getFirstName() + " " + entity.getLastName());
}

@BeforeMapping
protected void beforeMapping(CreateUserDTO dto) {
    dto.setEmail(dto.getEmail().toLowerCase());
}
```

## Collections

```java
List<UserDTO> toDTOList(List<User> entities);
Set<UserDTO> toDTOSet(Set<User> entities);

default Page<UserDTO> toDTOPage(Page<User> entities) {
    return entities.map(this::toDTO);
}
```

## Maven Config

```xml
<properties>
    <mapstruct.version>1.6.2</mapstruct.version>
    <lombok.version>1.18.30</lombok.version>
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
| `@BeanMapping` | Bean-level config |
| `@ValueMapping` | Enum mapping |
| `@AfterMapping` | Post-process |
| `@BeforeMapping` | Pre-process |
| `@Condition` | Conditional mapping |

## Common Patterns

```java
// Entity to DTO
@Mapping(target = "id", ignore = true)
@Mapping(target = "createdAt", ignore = true)
@Mapping(target = "updatedAt", ignore = true)
User toEntity(CreateUserDTO dto);

// DTO to Entity (response)
UserDTO toDTO(User entity);

// Partial update
@BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
void updateEntity(UpdateUserDTO dto, @MappingTarget User entity);

// List mapping
List<UserDTO> toDTOList(List<User> entities);
```

## Tips

- Order matters: Lombok → MapStruct → Binding
- Use `componentModel = "spring"` for DI
- Use `unmappedTargetPolicy = ReportingPolicy.IGNORE` for DTOs
- Always exclude `id`, `createdAt`, `updatedAt` from request DTOs
- Use abstract classes for custom logic
- Use `@MappingTarget` for updates
- Use `uses` parameter for nested mappers
