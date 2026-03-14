# JUnit 5 Quick Reference

## Basic Test

```java
@Test
void shouldAddNumbers() {
    assertEquals(5, calculator.add(2, 3));
}

@Test
void shouldThrowException() {
    assertThrows(IllegalArgumentException.class, () -> service.process(null));
}
```

## Assertions

```java
// Equality
assertEquals(expected, actual);
assertNotEquals(unexpected, actual);
assertSame(expected, actual);        // Identity
assertNotSame(expected, actual);

// Boolean
assertTrue(condition);
assertFalse(condition);

// Null
assertNull(object);
assertNotNull(object);

// Collections
assertArrayEquals(expectedArray, actualArray);

// Exceptions
assertThrows(Exception.class, () -> code());
assertDoesNotThrow(() -> code());

// Timeout
assertTimeout(Duration.ofSeconds(1), () -> code());

// Multiple
assertAll("user",
    () -> assertEquals("John", user.getName()),
    () -> assertTrue(user.isActive())
);
```

## Lifecycle

```java
@BeforeAll
static void setupAll() { }    // Once before all

@BeforeEach
void setUp() { }              // Before each test

@AfterEach
void tearDown() { }           // After each test

@AfterAll
static void tearDownAll() { } // Once after all
```

## Parameterized Tests

```java
@ParameterizedTest
@ValueSource(strings = {"apple", "banana"})
void testStrings(String fruit) { }

@ParameterizedTest
@CsvSource({"1,2,3", "10,20,30"})
void testWithCsv(int a, int b, int sum) { }

@ParameterizedTest
@MethodSource("provideUsers")
void testWithMethod(User user) { }

static Stream<User> provideUsers() {
    return Stream.of(new User("Alice"), new User("Bob"));
}

@ParameterizedTest
@EnumSource(UserRole.class)
void testEnum(UserRole role) { }
```

## Mockito

```java
@ExtendWith(MockitoExtension.class)
class ServiceTest {

    @Mock
    private Repository repo;

    @InjectMocks
    private ServiceImpl service;

    @Test
    void testWithMock() {
        when(repo.findById(1L)).thenReturn(Optional.of(user));

        User result = service.findById(1L);

        assertNotNull(result);
        verify(repo, times(1)).findById(1L);
    }
}
```

## Spring Boot Tests

```java
// Full context
@SpringBootTest
class IntegrationTest {
    @Autowired
    private Service service;
}

// Controller test
@WebMvcTest(UserController.class)
class ControllerTest {
    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService service;
}

// Repository test
@DataJpaTest
class RepositoryTest {
    @Autowired
    private UserRepository repo;
}
```

## MockMvc

```java
mockMvc.perform(get("/api/users/{id}", 1))
    .andExpect(status().isOk())
    .andExpect(jsonPath("$.name").value("John"));

mockMvc.perform(post("/api/users")
        .contentType(MediaType.APPLICATION_JSON)
        .content(jsonString))
    .andExpect(status().isCreated());
```

## Mockito Matchers

```java
when(repo.save(any(User.class))).thenReturn(user);
when(repo.findById(anyLong())).thenReturn(Optional.of(user));
when(repo.findByEmail(eq("test@test.com"))).thenReturn(user);
when(repo.findAll()).thenReturn(List.of(user1, user2));
```

## Verify

```java
verify(repo).save(any(User.class));
verify(repo, times(2)).findById(anyLong());
verify(repo, never()).delete(any());
verify(repo, atLeast(1)).findAll();
verify(repo, atMost(3)).save(any());
verifyNoMoreInteractions(repo);
```

## Annotations

| Annotation | Purpose |
|------------|---------|
| `@Test` | Test method |
| `@BeforeEach` | Run before each test |
| `@AfterEach` | Run after each test |
| `@BeforeAll` | Run once before all |
| `@AfterAll` | Run once after all |
| `@ParameterizedTest` | Parameterized test |
| `@Mock` | Create mock |
| `@InjectMocks` | Inject mocks |
| `@MockBean` | Spring Boot mock |
| `@SpringBootTest` | Integration test |
| `@WebMvcTest` | Controller test |
| `@DataJpaTest` | Repository test |

## Test Properties

```properties
# src/test/resources/application-test.properties
spring.datasource.url=jdbc:h2:mem:testdb
spring.jpa.hibernate.ddl-auto=create-drop
logging.level.root=WARN
```

## Common Patterns

```java
// Given-When-Then
@Test
void shouldCreateUser() {
    // Given
    CreateUserDTO dto = new CreateUserDTO("John", "john@test.com");
    when(repo.save(any())).thenReturn(user);

    // When
    UserDTO result = service.create(dto);

    // Then
    assertNotNull(result);
    assertEquals("John", result.getName());
    verify(repo).save(any());
}

// Testing exceptions
@Test
void shouldThrowWhenNotFound() {
    when(repo.findById(999L)).thenReturn(Optional.empty());

    assertThrows(ResourceNotFoundException.class,
        () -> service.findById(999L));
}

// Testing void methods
@Test
void shouldDeleteUser() {
    when(repo.existsById(1L)).thenReturn(true);

    assertDoesNotThrow(() -> service.delete(1L));

    verify(repo).deleteById(1L);
}
```
