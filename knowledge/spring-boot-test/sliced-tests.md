# Spring Boot Sliced Tests

> Source: https://docs.spring.io/spring-boot/reference/testing/spring-boot-applications.html

## Overview

Sliced tests load only the components needed for testing a specific layer of your application, making tests faster and more focused.

## Available Slices

### @WebMvcTest - Controller Layer

Tests Spring MVC controllers without starting a full application context.

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    @Test
    void shouldReturnUser() throws Exception {
        when(userService.findById(1L)).thenReturn(new User(1L, "John"));

        mockMvc.perform(get("/api/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("John"));
    }
}
```

**What's auto-configured:**
- `@Controller`, `@ControllerAdvice`, `@JsonComponent`
- `Converter`, `GenericConverter`, `Filter`
- `HandlerInterceptor`, `WebMvcConfigurer`
- MockMvc

**What's NOT included:**
- `@Component`, `@Service`, `@Repository` beans

### @DataJpaTest - Repository Layer

Tests JPA repositories with an embedded database.

```java
@DataJpaTest
class UserRepositoryTest {

    @Autowired
    private TestEntityManager entityManager;

    @Autowired
    private UserRepository repository;

    @Test
    void shouldFindByEmail() {
        entityManager.persist(new User("john@email.com", "John"));

        Optional<User> found = repository.findByEmail("john@email.com");

        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("John");
    }
}
```

**What's auto-configured:**
- JPA repositories
- `TestEntityManager`
- DataSource (embedded H2 by default)
- Flyway/Liquibase migrations (can be disabled)

**Using real database:**
```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = Replace.NONE)
@Testcontainers
class UserRepositoryTest {
    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");
}
```

### @DataMongoTest - MongoDB Layer

```java
@DataMongoTest
class ProductRepositoryTest {

    @Autowired
    private MongoTemplate mongoTemplate;

    @Autowired
    private ProductRepository repository;

    @Test
    void shouldFindByCategory() {
        mongoTemplate.save(new Product("Phone", "electronics"));

        List<Product> found = repository.findByCategory("electronics");

        assertThat(found).hasSize(1);
    }
}
```

### @JsonTest - JSON Serialization

```java
@JsonTest
class UserJsonTest {

    @Autowired
    private JacksonTester<User> json;

    @Test
    void shouldSerialize() throws Exception {
        User user = new User(1L, "John", "john@email.com");

        assertThat(json.write(user))
            .isEqualToJson("expected.json");

        assertThat(json.write(user))
            .hasJsonPathStringValue("@.name")
            .extractingJsonPathStringValue("@.email")
            .isEqualTo("john@email.com");
    }

    @Test
    void shouldDeserialize() throws Exception {
        String content = """
            {"id": 1, "name": "John", "email": "john@email.com"}
            """;

        assertThat(json.parse(content))
            .isEqualTo(new User(1L, "John", "john@email.com"));
    }
}
```

### @WebFluxTest - WebFlux Controllers

```java
@WebFluxTest(UserController.class)
class UserControllerTest {

    @Autowired
    private WebTestClient webClient;

    @MockBean
    private UserService userService;

    @Test
    void shouldReturnUser() {
        when(userService.findById(1L)).thenReturn(Mono.just(new User(1L, "John")));

        webClient.get().uri("/api/users/1")
            .exchange()
            .expectStatus().isOk()
            .expectBody(User.class)
            .value(user -> assertThat(user.getName()).isEqualTo("John"));
    }
}
```

### @RestClientTest - REST Clients

```java
@RestClientTest(UserClient.class)
class UserClientTest {

    @Autowired
    private UserClient client;

    @Autowired
    private MockRestServiceServer server;

    @Test
    void shouldGetUser() {
        server.expect(requestTo("/users/1"))
            .andRespond(withSuccess("""
                {"id": 1, "name": "John"}
                """, MediaType.APPLICATION_JSON));

        User user = client.getUser(1L);

        assertThat(user.getName()).isEqualTo("John");
    }
}
```

## Comparison Table

| Annotation | Layer | Auto-configured | Use Case |
|------------|-------|-----------------|----------|
| `@WebMvcTest` | Controller | MockMvc, JSON | REST endpoints |
| `@DataJpaTest` | Repository | JPA, DataSource | Database queries |
| `@DataMongoTest` | Repository | MongoTemplate | MongoDB operations |
| `@JsonTest` | Serialization | Jackson/Gson | JSON mapping |
| `@WebFluxTest` | Controller | WebTestClient | Reactive endpoints |
| `@RestClientTest` | Client | MockRestServiceServer | HTTP clients |

## Best Practices

1. **Use sliced tests for unit testing** - They're faster than `@SpringBootTest`
2. **Mock dependencies with `@MockBean`** - Keep tests isolated
3. **One slice per test class** - Don't mix annotations
4. **Use `@Import` for additional config** - When you need extra beans
5. **Prefer sliced tests over full context** - Only use `@SpringBootTest` for integration tests

## Common Patterns

### Adding Security Context
```java
@WebMvcTest(UserController.class)
@Import(SecurityConfig.class)
class SecuredControllerTest {

    @Test
    @WithMockUser(roles = "ADMIN")
    void adminCanAccess() throws Exception {
        mockMvc.perform(get("/admin/users"))
            .andExpect(status().isOk());
    }
}
```

### Custom Configuration
```java
@WebMvcTest(UserController.class)
@Import(TestConfig.class)
class UserControllerTest {

    @TestConfiguration
    static class TestConfig {
        @Bean
        UserService testUserService() {
            return new StubUserService();
        }
    }
}
```
