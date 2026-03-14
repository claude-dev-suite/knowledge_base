# JUnit 5 Testing Strategies

## Table of Contents
- [Test Pyramid](#test-pyramid)
- [Unit Testing Best Practices](#unit-testing-best-practices)
- [Integration Testing](#integration-testing)
- [Mocking Strategies](#mocking-strategies)
- [Test Data Builders](#test-data-builders)
- [Testing Spring Boot Applications](#testing-spring-boot-applications)
- [Performance Testing](#performance-testing)

---

## Test Pyramid

### Concept

```
        /\
       /E2E\      10% - End-to-End Tests (Slow, Brittle)
      /------\
     /Integr.\   20% - Integration Tests (Medium Speed)
    /----------\
   /   Unit     \ 70% - Unit Tests (Fast, Reliable)
  /--------------\
```

### Implementation Strategy

1. **Unit Tests (70%)**: Fast, isolated, test single units
2. **Integration Tests (20%)**: Test component interactions
3. **E2E Tests (10%)**: Test complete user flows

---

## Unit Testing Best Practices

### 1. Test Naming Convention

```java
// Pattern: should[ExpectedBehavior]When[StateUnderTest]
@Test
void shouldReturnUserWhenValidIdProvided() { }

@Test
void shouldThrowExceptionWhenUserNotFound() { }

@Test
void shouldUpdateEmailWhenValidEmailProvided() { }

// Alternative: [MethodName]_[StateUnderTest]_[ExpectedBehavior]
@Test
void findById_UserExists_ReturnsUser() { }

@Test
void create_DuplicateEmail_ThrowsException() { }
```

### 2. Arrange-Act-Assert (AAA) Pattern

```java
@Test
void shouldCalculateTotalPrice() {
    // Arrange (Given)
    Order order = Order.builder()
        .items(List.of(
            new Item("Product A", 10.00, 2),
            new Item("Product B", 15.00, 1)
        ))
        .build();

    // Act (When)
    BigDecimal total = orderService.calculateTotal(order);

    // Assert (Then)
    assertEquals(new BigDecimal("35.00"), total);
}
```

### 3. Test One Thing Per Test

```java
// ❌ Bad: Testing multiple things
@Test
void shouldCreateAndUpdateUser() {
    User user = service.create(dto);
    assertNotNull(user.getId());

    user.setName("Updated");
    User updated = service.update(user);
    assertEquals("Updated", updated.getName());
}

// ✅ Good: Separate tests
@Test
void shouldCreateUser() {
    User user = service.create(dto);
    assertNotNull(user.getId());
    assertEquals("John", user.getName());
}

@Test
void shouldUpdateUserName() {
    User user = existingUser();
    UpdateUserDTO dto = new UpdateUserDTO("Updated");

    User updated = service.update(user.getId(), dto);

    assertEquals("Updated", updated.getName());
}
```

### 4. Test Edge Cases

```java
@Test
void shouldHandleNullInput() {
    assertThrows(IllegalArgumentException.class,
        () -> service.process(null));
}

@Test
void shouldHandleEmptyList() {
    List<User> result = service.findAll(Collections.emptyList());
    assertTrue(result.isEmpty());
}

@Test
void shouldHandleMaxValue() {
    BigDecimal max = new BigDecimal(Long.MAX_VALUE);
    assertDoesNotThrow(() -> service.calculate(max));
}
```

---

## Integration Testing

### Repository Integration Tests

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class UserRepositoryIntegrationTest {

    @Autowired
    private TestEntityManager entityManager;

    @Autowired
    private UserRepository userRepository;

    @Test
    void shouldFindUserByEmail() {
        // Given
        User user = User.builder()
            .name("John Doe")
            .email("john@test.com")
            .password("encoded")
            .build();
        entityManager.persist(user);
        entityManager.flush();

        // When
        Optional<User> found = userRepository.findByEmail("john@test.com");

        // Then
        assertTrue(found.isPresent());
        assertEquals("John Doe", found.get().getName());
    }

    @Test
    void shouldNotFindUserWithNonExistentEmail() {
        Optional<User> found = userRepository.findByEmail("nonexistent@test.com");
        assertFalse(found.isPresent());
    }

    @Test
    @Sql("/test-data/users.sql")
    void shouldFindAllActiveUsers() {
        List<User> users = userRepository.findAllByStatus(UserStatus.ACTIVE);

        assertThat(users)
            .hasSize(3)
            .extracting(User::getStatus)
            .containsOnly(UserStatus.ACTIVE);
    }
}
```

### Service Integration Tests

```java
@SpringBootTest
@Transactional
@ActiveProfiles("test")
class UserServiceIntegrationTest {

    @Autowired
    private UserService userService;

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Test
    void shouldCreateUserWithEncodedPassword() {
        // Given
        CreateUserDTO dto = CreateUserDTO.builder()
            .name("Jane Doe")
            .email("jane@test.com")
            .password("plainPassword123")
            .build();

        // When
        UserDTO created = userService.create(dto);

        // Then
        assertNotNull(created.getId());

        User savedUser = userRepository.findById(created.getId()).orElseThrow();
        assertTrue(passwordEncoder.matches("plainPassword123", savedUser.getPassword()));
    }

    @Test
    void shouldThrowExceptionWhenCreatingDuplicateEmail() {
        // Given
        User existing = User.builder()
            .name("John")
            .email("duplicate@test.com")
            .password("pass")
            .build();
        userRepository.save(existing);

        CreateUserDTO dto = CreateUserDTO.builder()
            .name("Jane")
            .email("duplicate@test.com")
            .password("pass")
            .build();

        // When/Then
        assertThrows(DuplicateResourceException.class,
            () -> userService.create(dto));
    }
}
```

### Controller Integration Tests

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Transactional
class UserControllerIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private UserRepository userRepository;

    private String baseUrl;

    @BeforeEach
    void setUp(@LocalServerPort int port) {
        baseUrl = "http://localhost:" + port + "/api/v1/users";
    }

    @Test
    void shouldCreateUser() {
        CreateUserDTO dto = CreateUserDTO.builder()
            .name("John Doe")
            .email("john@test.com")
            .password("password123")
            .build();

        ResponseEntity<UserDTO> response = restTemplate.postForEntity(
            baseUrl, dto, UserDTO.class);

        assertEquals(HttpStatus.CREATED, response.getStatusCode());
        assertNotNull(response.getBody());
        assertEquals("John Doe", response.getBody().getName());
    }

    @Test
    void shouldGetUserById() {
        User user = userRepository.save(
            User.builder().name("Test").email("test@test.com").build());

        ResponseEntity<UserDTO> response = restTemplate.getForEntity(
            baseUrl + "/" + user.getId(), UserDTO.class);

        assertEquals(HttpStatus.OK, response.getStatusCode());
        assertEquals("Test", response.getBody().getName());
    }
}
```

---

## Mocking Strategies

### When to Mock

```java
// ✅ Mock external dependencies
@Mock
private EmailService emailService;

@Mock
private PaymentGateway paymentGateway;

// ✅ Mock repositories in service tests
@Mock
private UserRepository userRepository;

// ❌ Don't mock the class under test
// ❌ Don't mock value objects/DTOs
// ❌ Don't mock entities
```

### Strict vs Lenient Mocks

```java
// Strict (default) - verifies all interactions
@Mock
private UserRepository repository;

// Lenient - allows unstubbed calls
@Mock(lenient = true)
private LoggingService logger;

// Or in test
@Test
void test() {
    lenient().when(logger.log(anyString())).thenReturn(true);
}
```

### Partial Mocking with Spy

```java
@Spy
private UserService userService = new UserServiceImpl();

@Test
void shouldUseRealMethodWithSpy() {
    // Use real method
    doCallRealMethod().when(userService).validateEmail(anyString());

    // Mock only specific method
    doReturn(true).when(userService).checkEmailExists(anyString());

    boolean result = userService.isValidAndUnique("test@test.com");
    assertTrue(result);
}
```

### Argument Captors

```java
@Test
void shouldSaveUserWithCorrectData() {
    ArgumentCaptor<User> userCaptor = ArgumentCaptor.forClass(User.class);

    service.create(dto);

    verify(repository).save(userCaptor.capture());
    User saved = userCaptor.getValue();
    assertEquals("John", saved.getName());
    assertNotNull(saved.getPassword());
}
```

---

## Test Data Builders

### Builder Pattern for Test Data

```java
public class UserTestBuilder {
    private Long id = 1L;
    private String name = "Test User";
    private String email = "test@test.com";
    private String password = "password";
    private UserRole role = UserRole.USER;
    private UserStatus status = UserStatus.ACTIVE;
    private LocalDateTime createdAt = LocalDateTime.now();

    public static UserTestBuilder aUser() {
        return new UserTestBuilder();
    }

    public UserTestBuilder withId(Long id) {
        this.id = id;
        return this;
    }

    public UserTestBuilder withName(String name) {
        this.name = name;
        return this;
    }

    public UserTestBuilder withEmail(String email) {
        this.email = email;
        return this;
    }

    public UserTestBuilder withRole(UserRole role) {
        this.role = role;
        return this;
    }

    public UserTestBuilder admin() {
        this.role = UserRole.ADMIN;
        return this;
    }

    public UserTestBuilder inactive() {
        this.status = UserStatus.INACTIVE;
        return this;
    }

    public User build() {
        return User.builder()
            .id(id)
            .name(name)
            .email(email)
            .password(password)
            .role(role)
            .status(status)
            .createdAt(createdAt)
            .build();
    }
}

// Usage in tests
@Test
void testAdminUser() {
    User admin = aUser().admin().withName("Admin").build();
    assertTrue(admin.isAdmin());
}

@Test
void testInactiveUser() {
    User user = aUser().inactive().build();
    assertFalse(user.isActive());
}
```

### Object Mother Pattern

```java
public class UserMother {
    public static User createDefault() {
        return User.builder()
            .id(1L)
            .name("Default User")
            .email("default@test.com")
            .build();
    }

    public static User createAdmin() {
        return User.builder()
            .id(2L)
            .name("Admin User")
            .email("admin@test.com")
            .role(UserRole.ADMIN)
            .build();
    }

    public static User createInactive() {
        return User.builder()
            .id(3L)
            .name("Inactive User")
            .email("inactive@test.com")
            .status(UserStatus.INACTIVE)
            .build();
    }
}
```

---

## Testing Spring Boot Applications

### Test Configuration

```java
@TestConfiguration
public class TestConfig {

    @Bean
    @Primary
    public PasswordEncoder passwordEncoder() {
        // Use fast encoder for tests
        return NoOpPasswordEncoder.getInstance();
    }

    @Bean
    public Clock fixedClock() {
        return Clock.fixed(
            Instant.parse("2024-01-01T00:00:00Z"),
            ZoneId.of("UTC")
        );
    }
}
```

### Test Profiles

```yaml
# src/test/resources/application-test.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: false
  mail:
    host: localhost
    port: 25

logging:
  level:
    root: WARN
    com.company: DEBUG
```

### Test Slices

```java
// Test only web layer
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;
}

// Test only JPA layer
@DataJpaTest
class UserRepositoryTest {
    @Autowired
    private UserRepository repository;
}

// Test only JSON serialization
@JsonTest
class UserDTOJsonTest {
    @Autowired
    private JacksonTester<UserDTO> json;
}
```

---

## Performance Testing

### Testing Performance Constraints

```java
@Test
@Timeout(value = 1, unit = TimeUnit.SECONDS)
void shouldCompleteWithinOneSecond() {
    service.processLargeDataset(data);
}

@Test
void shouldHandleLargeDataset() {
    List<User> users = IntStream.range(0, 10000)
        .mapToObj(i -> new User("User" + i, "user" + i + "@test.com"))
        .collect(Collectors.toList());

    assertTimeout(Duration.ofSeconds(5), () -> {
        service.processBatch(users);
    });
}
```

### Memory Testing

```java
@Test
void shouldNotCauseMemoryLeak() {
    Runtime runtime = Runtime.getRuntime();
    long before = runtime.totalMemory() - runtime.freeMemory();

    // Process large dataset
    for (int i = 0; i < 1000; i++) {
        service.process(createLargeObject());
    }

    System.gc();
    Thread.sleep(100);

    long after = runtime.totalMemory() - runtime.freeMemory();
    long growth = after - before;

    assertTrue(growth < 10_000_000,
        "Memory growth exceeds 10MB: " + growth);
}
```

---

## Best Practices Summary

1. ✅ Follow the test pyramid (70% unit, 20% integration, 10% E2E)
2. ✅ Use descriptive test names
3. ✅ Follow AAA pattern
4. ✅ Test one thing per test
5. ✅ Test edge cases and error paths
6. ✅ Use test builders for complex objects
7. ✅ Mock external dependencies only
8. ✅ Use appropriate test slices (@WebMvcTest, @DataJpaTest)
9. ✅ Keep tests fast and independent
10. ✅ Use @Transactional for test data cleanup

## Anti-Patterns

1. ❌ Testing private methods directly
2. ❌ Mocking the class under test
3. ❌ Tests depending on execution order
4. ❌ Tests depending on external state
5. ❌ Ignoring test failures
6. ❌ Not cleaning up test data
7. ❌ Too many mocks (test becomes brittle)
8. ❌ Testing framework code
9. ❌ Sleeping in tests (use proper waits)
10. ❌ Copy-paste test code

---

## References

- [JUnit 5 User Guide](https://junit.org/junit5/docs/current/user-guide/)
- [Mockito Documentation](https://javadoc.io/doc/org.mockito/mockito-core/latest/org/mockito/Mockito.html)
- [Spring Boot Testing](https://docs.spring.io/spring-boot/reference/testing/index.html)
- [Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html)
