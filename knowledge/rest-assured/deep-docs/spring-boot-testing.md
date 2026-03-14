# REST Assured Spring Boot Testing Guide

## Complete Integration Testing Setup

### Maven Dependencies

```xml
<dependencies>
    <!-- REST Assured -->
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>rest-assured</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>json-path</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>xml-path</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- Spring Boot Test -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- Testcontainers (optional) -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>postgresql</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

## REST Assured with Spring Boot

### Base Test Configuration

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
public abstract class BaseIntegrationTest {

    @LocalServerPort
    private int port;

    @Autowired
    protected TestRestTemplate restTemplate;

    @BeforeEach
    void setUp() {
        RestAssured.port = port;
        RestAssured.basePath = "/api";
        RestAssured.enableLoggingOfRequestAndResponseIfValidationFails();
    }

    @AfterEach
    void tearDown() {
        RestAssured.reset();
    }
}
```

### Controller Testing

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Sql(scripts = "/test-data.sql", executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD)
@Sql(scripts = "/cleanup.sql", executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
class UserControllerIntegrationTest extends BaseIntegrationTest {

    @Test
    @DisplayName("Should create user successfully")
    void shouldCreateUser() {
        CreateUserRequest request = CreateUserRequest.builder()
            .email("test@example.com")
            .name("Test User")
            .password("Password123!")
            .build();

        given()
            .contentType(ContentType.JSON)
            .body(request)
        .when()
            .post("/users")
        .then()
            .statusCode(201)
            .header("Location", matchesPattern(".*/api/users/\\d+"))
            .body("id", notNullValue())
            .body("email", equalTo("test@example.com"))
            .body("name", equalTo("Test User"))
            .body("password", nullValue()) // Password should not be returned
            .body("createdAt", notNullValue());
    }

    @Test
    @DisplayName("Should return validation errors for invalid input")
    void shouldReturnValidationErrors() {
        CreateUserRequest request = CreateUserRequest.builder()
            .email("invalid-email")
            .name("")
            .password("123")
            .build();

        given()
            .contentType(ContentType.JSON)
            .body(request)
        .when()
            .post("/users")
        .then()
            .statusCode(400)
            .body("message", equalTo("Validation failed"))
            .body("errors", hasSize(3))
            .body("errors.email", hasItem("Invalid email format"))
            .body("errors.name", hasItem("Name must not be empty"))
            .body("errors.password", hasItem("Password must be at least 8 characters"));
    }

    @Test
    @DisplayName("Should get user by ID")
    void shouldGetUserById() {
        given()
            .pathParam("id", 1)
        .when()
            .get("/users/{id}")
        .then()
            .statusCode(200)
            .body("id", equalTo(1))
            .body("email", equalTo("existing@example.com"))
            .body("name", equalTo("Existing User"));
    }

    @Test
    @DisplayName("Should return 404 for non-existent user")
    void shouldReturn404ForNonExistentUser() {
        given()
            .pathParam("id", 999)
        .when()
            .get("/users/{id}")
        .then()
            .statusCode(404)
            .body("message", equalTo("User not found with id: 999"));
    }

    @Test
    @DisplayName("Should update user successfully")
    void shouldUpdateUser() {
        UpdateUserRequest request = UpdateUserRequest.builder()
            .name("Updated Name")
            .email("updated@example.com")
            .build();

        given()
            .contentType(ContentType.JSON)
            .pathParam("id", 1)
            .body(request)
        .when()
            .put("/users/{id}")
        .then()
            .statusCode(200)
            .body("id", equalTo(1))
            .body("name", equalTo("Updated Name"))
            .body("email", equalTo("updated@example.com"));
    }

    @Test
    @DisplayName("Should delete user successfully")
    void shouldDeleteUser() {
        given()
            .pathParam("id", 1)
        .when()
            .delete("/users/{id}")
        .then()
            .statusCode(204);

        // Verify user is deleted
        given()
            .pathParam("id", 1)
        .when()
            .get("/users/{id}")
        .then()
            .statusCode(404);
    }

    @Test
    @DisplayName("Should list users with pagination")
    void shouldListUsersWithPagination() {
        given()
            .queryParam("page", 0)
            .queryParam("size", 10)
            .queryParam("sort", "name,asc")
        .when()
            .get("/users")
        .then()
            .statusCode(200)
            .body("content", hasSize(lessThanOrEqualTo(10)))
            .body("totalElements", greaterThanOrEqualTo(0))
            .body("totalPages", greaterThanOrEqualTo(0))
            .body("number", equalTo(0))
            .body("size", equalTo(10));
    }
}
```

## Authentication Testing

### JWT Token Testing

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class AuthenticationIntegrationTest extends BaseIntegrationTest {

    @Autowired
    private JwtService jwtService;

    private String validToken;

    @BeforeEach
    void setUpAuth() {
        super.setUp();
        // Generate valid token for tests
        validToken = jwtService.generateToken(
            User.builder()
                .id(1L)
                .email("test@example.com")
                .role(Role.USER)
                .build()
        );
    }

    @Test
    @DisplayName("Should authenticate with valid credentials")
    void shouldAuthenticateWithValidCredentials() {
        LoginRequest request = new LoginRequest("test@example.com", "password123");

        given()
            .contentType(ContentType.JSON)
            .body(request)
        .when()
            .post("/auth/login")
        .then()
            .statusCode(200)
            .body("token", notNullValue())
            .body("type", equalTo("Bearer"))
            .body("expiresIn", greaterThan(0));
    }

    @Test
    @DisplayName("Should reject invalid credentials")
    void shouldRejectInvalidCredentials() {
        LoginRequest request = new LoginRequest("test@example.com", "wrongpassword");

        given()
            .contentType(ContentType.JSON)
            .body(request)
        .when()
            .post("/auth/login")
        .then()
            .statusCode(401)
            .body("message", equalTo("Invalid credentials"));
    }

    @Test
    @DisplayName("Should access protected endpoint with valid token")
    void shouldAccessProtectedEndpointWithValidToken() {
        given()
            .header("Authorization", "Bearer " + validToken)
        .when()
            .get("/users/me")
        .then()
            .statusCode(200)
            .body("id", equalTo(1))
            .body("email", equalTo("test@example.com"));
    }

    @Test
    @DisplayName("Should reject request without token")
    void shouldRejectRequestWithoutToken() {
        given()
        .when()
            .get("/users/me")
        .then()
            .statusCode(401)
            .body("message", equalTo("Missing authentication token"));
    }

    @Test
    @DisplayName("Should reject request with expired token")
    void shouldRejectRequestWithExpiredToken() {
        String expiredToken = jwtService.generateExpiredToken();

        given()
            .header("Authorization", "Bearer " + expiredToken)
        .when()
            .get("/users/me")
        .then()
            .statusCode(401)
            .body("message", equalTo("Token has expired"));
    }

    @Test
    @DisplayName("Should reject access to admin endpoint with user role")
    void shouldRejectAccessToAdminEndpointWithUserRole() {
        given()
            .header("Authorization", "Bearer " + validToken)
        .when()
            .get("/admin/users")
        .then()
            .statusCode(403)
            .body("message", equalTo("Insufficient permissions"));
    }
}
```

## Repository Layer Testing

### @DataJpaTest Integration

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class UserRepositoryIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine")
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
    @DisplayName("Should find user by email")
    void shouldFindUserByEmail() {
        // Given
        User user = User.builder()
            .email("test@example.com")
            .name("Test User")
            .password("hashedPassword")
            .build();
        entityManager.persistAndFlush(user);

        // When
        Optional<User> found = userRepository.findByEmail("test@example.com");

        // Then
        assertThat(found).isPresent();
        assertThat(found.get().getEmail()).isEqualTo("test@example.com");
        assertThat(found.get().getName()).isEqualTo("Test User");
    }

    @Test
    @DisplayName("Should find users by role")
    void shouldFindUsersByRole() {
        // Given
        User admin = User.builder()
            .email("admin@example.com")
            .name("Admin")
            .role(Role.ADMIN)
            .build();
        User user = User.builder()
            .email("user@example.com")
            .name("User")
            .role(Role.USER)
            .build();
        entityManager.persist(admin);
        entityManager.persist(user);
        entityManager.flush();

        // When
        List<User> admins = userRepository.findByRole(Role.ADMIN);

        // Then
        assertThat(admins).hasSize(1);
        assertThat(admins.get(0).getRole()).isEqualTo(Role.ADMIN);
    }

    @Test
    @DisplayName("Should check if email exists")
    void shouldCheckIfEmailExists() {
        // Given
        User user = User.builder()
            .email("existing@example.com")
            .name("Existing User")
            .build();
        entityManager.persistAndFlush(user);

        // When & Then
        assertThat(userRepository.existsByEmail("existing@example.com")).isTrue();
        assertThat(userRepository.existsByEmail("nonexistent@example.com")).isFalse();
    }
}
```

## Service Layer Testing with Mocks

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @Mock
    private PasswordEncoder passwordEncoder;

    @Mock
    private UserMapper userMapper;

    @InjectMocks
    private UserService userService;

    @Test
    @DisplayName("Should create user successfully")
    void shouldCreateUser() {
        // Given
        CreateUserRequest request = CreateUserRequest.builder()
            .email("test@example.com")
            .name("Test User")
            .password("Password123!")
            .build();

        User user = User.builder()
            .email("test@example.com")
            .name("Test User")
            .build();

        User savedUser = User.builder()
            .id(1L)
            .email("test@example.com")
            .name("Test User")
            .build();

        UserResponse response = UserResponse.builder()
            .id(1L)
            .email("test@example.com")
            .name("Test User")
            .build();

        when(userRepository.existsByEmail(request.getEmail())).thenReturn(false);
        when(userMapper.toEntity(request)).thenReturn(user);
        when(passwordEncoder.encode(request.getPassword())).thenReturn("encodedPassword");
        when(userRepository.save(user)).thenReturn(savedUser);
        when(userMapper.toResponse(savedUser)).thenReturn(response);

        // When
        UserResponse result = userService.createUser(request);

        // Then
        assertThat(result).isNotNull();
        assertThat(result.getId()).isEqualTo(1L);
        assertThat(result.getEmail()).isEqualTo("test@example.com");

        verify(userRepository).existsByEmail(request.getEmail());
        verify(userRepository).save(user);
        verify(passwordEncoder).encode(request.getPassword());
    }

    @Test
    @DisplayName("Should throw exception when email already exists")
    void shouldThrowExceptionWhenEmailExists() {
        // Given
        CreateUserRequest request = CreateUserRequest.builder()
            .email("existing@example.com")
            .name("Test User")
            .password("Password123!")
            .build();

        when(userRepository.existsByEmail(request.getEmail())).thenReturn(true);

        // When & Then
        assertThatThrownBy(() -> userService.createUser(request))
            .isInstanceOf(DuplicateEmailException.class)
            .hasMessage("Email already exists: existing@example.com");

        verify(userRepository).existsByEmail(request.getEmail());
        verify(userRepository, never()).save(any());
    }
}
```

## Advanced Request/Response Validation

### JSON Schema Validation

```java
@Test
@DisplayName("Should validate response against JSON schema")
void shouldValidateResponseAgainstJsonSchema() {
    given()
        .pathParam("id", 1)
    .when()
        .get("/users/{id}")
    .then()
        .statusCode(200)
        .body(matchesJsonSchemaInClasspath("schemas/user-response-schema.json"));
}
```

### Complex Response Assertions

```java
@Test
@DisplayName("Should validate complex nested response")
void shouldValidateComplexNestedResponse() {
    given()
        .pathParam("id", 1)
    .when()
        .get("/users/{id}/full")
    .then()
        .statusCode(200)
        .body("user.id", equalTo(1))
        .body("user.email", equalTo("test@example.com"))
        .body("user.profile.firstName", notNullValue())
        .body("user.profile.lastName", notNullValue())
        .body("user.addresses", hasSize(greaterThan(0)))
        .body("user.addresses[0].street", notNullValue())
        .body("user.addresses[0].city", notNullValue())
        .body("user.addresses.findAll { it.type == 'HOME' }", hasSize(1))
        .body("user.roles", hasItem("USER"))
        .body("user.permissions", everyItem(startsWith("READ_")));
}
```

### Extract and Reuse Values

```java
@Test
@DisplayName("Should create and use resource in subsequent requests")
void shouldCreateAndUseResource() {
    // Create user and extract ID
    int userId = given()
        .contentType(ContentType.JSON)
        .body(new CreateUserRequest("test@example.com", "Test User", "password"))
    .when()
        .post("/users")
    .then()
        .statusCode(201)
        .extract()
        .path("id");

    // Use extracted ID in subsequent request
    given()
        .pathParam("id", userId)
    .when()
        .get("/users/{id}")
    .then()
        .statusCode(200)
        .body("id", equalTo(userId));

    // Extract entire response for complex operations
    UserResponse user = given()
        .pathParam("id", userId)
    .when()
        .get("/users/{id}")
    .then()
        .statusCode(200)
        .extract()
        .as(UserResponse.class);

    assertThat(user.getId()).isEqualTo(userId);
    assertThat(user.getEmail()).isEqualTo("test@example.com");
}
```

## File Upload Testing

```java
@Test
@DisplayName("Should upload user avatar successfully")
void shouldUploadUserAvatar() {
    File file = new File("src/test/resources/test-avatar.jpg");

    given()
        .multiPart("file", file, "image/jpeg")
        .pathParam("id", 1)
    .when()
        .post("/users/{id}/avatar")
    .then()
        .statusCode(200)
        .body("avatarUrl", notNullValue())
        .body("avatarUrl", matchesPattern("https://.*\\.jpg"));
}

@Test
@DisplayName("Should reject invalid file type")
void shouldRejectInvalidFileType() {
    File file = new File("src/test/resources/test-file.pdf");

    given()
        .multiPart("file", file, "application/pdf")
        .pathParam("id", 1)
    .when()
        .post("/users/{id}/avatar")
    .then()
        .statusCode(400)
        .body("message", equalTo("Only image files are allowed"));
}
```

## Error Handling Testing

```java
@Test
@DisplayName("Should handle internal server errors gracefully")
void shouldHandleInternalServerErrors() {
    // Simulate error condition
    given()
        .pathParam("id", -1) // Invalid ID causing server error
    .when()
        .get("/users/{id}")
    .then()
        .statusCode(500)
        .body("message", equalTo("Internal server error"))
        .body("timestamp", notNullValue())
        .body("path", equalTo("/api/users/-1"));
}

@Test
@DisplayName("Should return appropriate error for constraint violation")
void shouldReturnErrorForConstraintViolation() {
    CreateUserRequest request = CreateUserRequest.builder()
        .email("duplicate@example.com") // Already exists
        .name("Test User")
        .password("Password123!")
        .build();

    given()
        .contentType(ContentType.JSON)
        .body(request)
    .when()
        .post("/users")
    .then()
        .statusCode(409)
        .body("message", containsString("already exists"))
        .body("errorCode", equalTo("DUPLICATE_EMAIL"));
}
```

## Performance Testing

```java
@Test
@DisplayName("Should respond within acceptable time limit")
void shouldRespondWithinTimeLimit() {
    given()
        .pathParam("id", 1)
    .when()
        .get("/users/{id}")
    .then()
        .statusCode(200)
        .time(lessThan(500L), TimeUnit.MILLISECONDS);
}

@Test
@DisplayName("Should handle bulk operations efficiently")
void shouldHandleBulkOperationsEfficiently() {
    List<CreateUserRequest> users = IntStream.range(0, 100)
        .mapToObj(i -> CreateUserRequest.builder()
            .email("user" + i + "@example.com")
            .name("User " + i)
            .password("password")
            .build())
        .collect(Collectors.toList());

    long startTime = System.currentTimeMillis();

    given()
        .contentType(ContentType.JSON)
        .body(users)
    .when()
        .post("/users/bulk")
    .then()
        .statusCode(201)
        .body("created", equalTo(100));

    long duration = System.currentTimeMillis() - startTime;
    assertThat(duration).isLessThan(5000); // Should complete within 5 seconds
}
```

## Best Practices

1. ✅ Use `@SpringBootTest` with `RANDOM_PORT` for full integration tests
2. ✅ Use `@DataJpaTest` for repository layer tests
3. ✅ Use Testcontainers for real database testing
4. ✅ Always test authentication and authorization
5. ✅ Validate both success and error scenarios
6. ✅ Test edge cases (empty data, max values, special characters)
7. ✅ Use descriptive test names with `@DisplayName`
8. ✅ Extract common setup to base test classes
9. ✅ Clean up test data after each test
10. ✅ Test response time for performance-critical endpoints

## References

- [REST Assured Documentation](https://rest-assured.io/)
- [Spring Boot Testing](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.testing)
- [Testcontainers](https://testcontainers.com/)
- [Hamcrest Matchers](http://hamcrest.org/JavaHamcrest/)
