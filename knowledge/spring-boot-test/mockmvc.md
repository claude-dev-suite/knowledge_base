# Spring Boot MockMvc Testing

> Source: https://docs.spring.io/spring-framework/reference/testing/spring-mvc-test-framework.html

## Overview

MockMvc provides a way to test Spring MVC controllers without starting a full HTTP server.

## Setup

### With @WebMvcTest (Recommended for Unit Tests)

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;
}
```

### With @SpringBootTest (Integration Tests)

```java
@SpringBootTest
@AutoConfigureMockMvc
class UserIntegrationTest {

    @Autowired
    private MockMvc mockMvc;
}
```

## Request Building

### GET Requests

```java
// Simple GET
mockMvc.perform(get("/api/users"))
    .andExpect(status().isOk());

// GET with path variable
mockMvc.perform(get("/api/users/{id}", 1))
    .andExpect(status().isOk());

// GET with query parameters
mockMvc.perform(get("/api/users")
        .param("page", "0")
        .param("size", "10")
        .param("sort", "name,asc"))
    .andExpect(status().isOk());

// GET with headers
mockMvc.perform(get("/api/users")
        .header("Authorization", "Bearer token")
        .accept(MediaType.APPLICATION_JSON))
    .andExpect(status().isOk());
```

### POST Requests

```java
// POST with JSON body
mockMvc.perform(post("/api/users")
        .contentType(MediaType.APPLICATION_JSON)
        .content("""
            {
                "name": "John",
                "email": "john@email.com"
            }
            """))
    .andExpect(status().isCreated());

// POST with ObjectMapper
@Autowired
private ObjectMapper objectMapper;

CreateUserRequest request = new CreateUserRequest("John", "john@email.com");
mockMvc.perform(post("/api/users")
        .contentType(MediaType.APPLICATION_JSON)
        .content(objectMapper.writeValueAsString(request)))
    .andExpect(status().isCreated());

// POST form data
mockMvc.perform(post("/api/login")
        .contentType(MediaType.APPLICATION_FORM_URLENCODED)
        .param("username", "john")
        .param("password", "secret"))
    .andExpect(status().isOk());
```

### PUT/PATCH/DELETE

```java
// PUT
mockMvc.perform(put("/api/users/{id}", 1)
        .contentType(MediaType.APPLICATION_JSON)
        .content("""{"name": "Jane"}"""))
    .andExpect(status().isOk());

// PATCH
mockMvc.perform(patch("/api/users/{id}", 1)
        .contentType(MediaType.APPLICATION_JSON)
        .content("""{"name": "Jane"}"""))
    .andExpect(status().isOk());

// DELETE
mockMvc.perform(delete("/api/users/{id}", 1))
    .andExpect(status().isNoContent());
```

## Response Assertions

### Status Codes

```java
.andExpect(status().isOk())           // 200
.andExpect(status().isCreated())      // 201
.andExpect(status().isAccepted())     // 202
.andExpect(status().isNoContent())    // 204
.andExpect(status().isBadRequest())   // 400
.andExpect(status().isUnauthorized()) // 401
.andExpect(status().isForbidden())    // 403
.andExpect(status().isNotFound())     // 404
.andExpect(status().isConflict())     // 409
.andExpect(status().is5xxServerError()) // 5xx
```

### JSON Response Body

```java
// Simple value check
.andExpect(jsonPath("$.name").value("John"))
.andExpect(jsonPath("$.email").value("john@email.com"))

// Check existence
.andExpect(jsonPath("$.id").exists())
.andExpect(jsonPath("$.deletedAt").doesNotExist())

// Type checks
.andExpect(jsonPath("$.id").isNumber())
.andExpect(jsonPath("$.name").isString())
.andExpect(jsonPath("$.active").isBoolean())
.andExpect(jsonPath("$.items").isArray())
.andExpect(jsonPath("$.address").isMap())

// Array assertions
.andExpect(jsonPath("$.items").isArray())
.andExpect(jsonPath("$.items", hasSize(3)))
.andExpect(jsonPath("$.items[0].name").value("Item 1"))
.andExpect(jsonPath("$.items[*].id", containsInAnyOrder(1, 2, 3)))

// Comparison
.andExpect(jsonPath("$.price").value(greaterThan(0)))
.andExpect(jsonPath("$.count").value(lessThanOrEqualTo(100)))

// Regex
.andExpect(jsonPath("$.email").value(matchesRegex(".*@.*\\..*")))

// Null check
.andExpect(jsonPath("$.middleName").value(nullValue()))
.andExpect(jsonPath("$.name").value(notNullValue()))
```

### Headers

```java
.andExpect(header().string("Content-Type", "application/json"))
.andExpect(header().exists("X-Request-Id"))
.andExpect(header().doesNotExist("X-Debug"))
.andExpect(header().string("Location", containsString("/api/users/")))
```

### Content

```java
// Plain text
.andExpect(content().string("Hello World"))
.andExpect(content().string(containsString("Hello")))

// Content type
.andExpect(content().contentType(MediaType.APPLICATION_JSON))
.andExpect(content().contentTypeCompatibleWith(MediaType.APPLICATION_JSON))

// JSON comparison
.andExpect(content().json("""
    {"name": "John", "email": "john@email.com"}
    """))

// JSON with strict mode
.andExpect(content().json(expectedJson, true)) // strict=true
```

## Security Testing

### @WithMockUser

```java
@Test
@WithMockUser(username = "john", roles = {"USER"})
void userCanAccessOwnProfile() throws Exception {
    mockMvc.perform(get("/api/profile"))
        .andExpect(status().isOk());
}

@Test
@WithMockUser(roles = "ADMIN")
void adminCanDeleteUser() throws Exception {
    mockMvc.perform(delete("/api/users/1"))
        .andExpect(status().isNoContent());
}
```

### RequestPostProcessor

```java
mockMvc.perform(get("/api/profile")
        .with(user("john").roles("USER")))
    .andExpect(status().isOk());

mockMvc.perform(post("/api/admin/users")
        .with(user("admin").roles("ADMIN"))
        .contentType(MediaType.APPLICATION_JSON)
        .content("""{"name": "New User"}"""))
    .andExpect(status().isCreated());
```

### JWT Token

```java
mockMvc.perform(get("/api/users")
        .header("Authorization", "Bearer " + jwtToken))
    .andExpect(status().isOk());
```

### CSRF

```java
mockMvc.perform(post("/api/users")
        .with(csrf())
        .contentType(MediaType.APPLICATION_JSON)
        .content("""{"name": "John"}"""))
    .andExpect(status().isCreated());
```

## Result Actions

### Print Response

```java
mockMvc.perform(get("/api/users"))
    .andDo(print())  // Print to console
    .andExpect(status().isOk());
```

### Extract Response

```java
MvcResult result = mockMvc.perform(post("/api/users")
        .contentType(MediaType.APPLICATION_JSON)
        .content("""{"name": "John"}"""))
    .andExpect(status().isCreated())
    .andReturn();

String content = result.getResponse().getContentAsString();
UserResponse response = objectMapper.readValue(content, UserResponse.class);
```

## Error Handling

```java
@Test
void shouldReturn400ForInvalidRequest() throws Exception {
    mockMvc.perform(post("/api/users")
            .contentType(MediaType.APPLICATION_JSON)
            .content("""{"name": ""}"""))  // Invalid: empty name
        .andExpect(status().isBadRequest())
        .andExpect(jsonPath("$.errors[0].field").value("name"))
        .andExpect(jsonPath("$.errors[0].message").value("must not be blank"));
}

@Test
void shouldReturn404WhenNotFound() throws Exception {
    when(userService.findById(999L))
        .thenThrow(new ResourceNotFoundException("User not found"));

    mockMvc.perform(get("/api/users/999"))
        .andExpect(status().isNotFound())
        .andExpect(jsonPath("$.message").value("User not found"));
}
```

## MockMvcTester (Spring Boot 3.4+)

```java
@Autowired
private MockMvcTester mvc;

@Test
void shouldReturnUser() {
    assertThat(mvc.get().uri("/api/users/1"))
        .hasStatusOk()
        .hasContentType(MediaType.APPLICATION_JSON)
        .bodyJson()
        .hasPath("$.name").isEqualTo("John");
}
```

## Best Practices

1. **Use `@WebMvcTest` for controller unit tests** - Faster than `@SpringBootTest`
2. **Mock service dependencies with `@MockBean`**
3. **Test both happy path and error cases**
4. **Use `andDo(print())` when debugging**
5. **Extract and verify Location header for POST**
6. **Test security annotations with `@WithMockUser`**
