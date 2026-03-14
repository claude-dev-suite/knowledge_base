# Spring GraphQL - Basics

## Overview

Spring for GraphQL provides support for building GraphQL applications with Spring. It integrates with Spring MVC, Spring WebFlux, and supports both query and subscription operations.

## Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-graphql</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

## GraphQL Schema

### src/main/resources/graphql/schema.graphqls
```graphql
type Query {
    userById(id: ID!): User
    users: [User!]!
    searchUsers(filter: UserFilter!): [User!]!
}

type Mutation {
    createUser(input: CreateUserInput!): User!
    updateUser(id: ID!, input: UpdateUserInput!): User
    deleteUser(id: ID!): Boolean!
}

type Subscription {
    userCreated: User!
}

type User {
    id: ID!
    name: String!
    email: String!
    orders: [Order!]!
    createdAt: DateTime!
}

type Order {
    id: ID!
    total: Float!
    status: OrderStatus!
    items: [OrderItem!]!
}

type OrderItem {
    id: ID!
    productId: ID!
    productName: String!
    quantity: Int!
    price: Float!
}

enum OrderStatus {
    PENDING
    PROCESSING
    SHIPPED
    DELIVERED
    CANCELLED
}

input UserFilter {
    name: String
    email: String
    status: String
}

input CreateUserInput {
    name: String!
    email: String!
}

input UpdateUserInput {
    name: String
    email: String
}

scalar DateTime
```

## Configuration

### application.yml
```yaml
spring:
  graphql:
    graphiql:
      enabled: true  # Enable GraphiQL UI at /graphiql
    schema:
      printer:
        enabled: true
    cors:
      allowed-origins: "*"
      allowed-methods: GET, POST

logging:
  level:
    org.springframework.graphql: DEBUG
```

## Controller (Annotated Handler)

```java
@Controller
public class UserController {

    private final UserService userService;

    @QueryMapping
    public User userById(@Argument Long id) {
        return userService.findById(id);
    }

    @QueryMapping
    public List<User> users() {
        return userService.findAll();
    }

    @QueryMapping
    public List<User> searchUsers(@Argument UserFilter filter) {
        return userService.search(filter);
    }

    @MutationMapping
    public User createUser(@Argument CreateUserInput input) {
        return userService.create(input);
    }

    @MutationMapping
    public User updateUser(@Argument Long id, @Argument UpdateUserInput input) {
        return userService.update(id, input);
    }

    @MutationMapping
    public boolean deleteUser(@Argument Long id) {
        return userService.delete(id);
    }
}
```

## Data Fetchers

### Nested Fields
```java
@Controller
public class UserController {

    @QueryMapping
    public User userById(@Argument Long id) {
        return userService.findById(id);
    }

    // Resolves User.orders field
    @SchemaMapping(typeName = "User", field = "orders")
    public List<Order> orders(User user) {
        return orderService.findByUserId(user.getId());
    }
}
```

### Batch Loading
```java
@Controller
public class OrderController {

    // Resolves OrderItem.productName efficiently with batching
    @BatchMapping
    public Mono<Map<OrderItem, String>> productName(List<OrderItem> items) {
        List<Long> productIds = items.stream()
            .map(OrderItem::getProductId)
            .distinct()
            .toList();

        return productService.findByIds(productIds)
            .collectMap(Product::getId, Product::getName)
            .map(products -> items.stream()
                .collect(Collectors.toMap(
                    item -> item,
                    item -> products.get(item.getProductId())
                )));
    }
}
```

## Subscriptions

### WebSocket Configuration
```java
@Configuration
public class GraphQLConfig {

    @Bean
    public GraphQlWebSocketHandler graphQlWebSocketHandler(
            WebGraphQlHandler webGraphQlHandler) {
        return new GraphQlWebSocketHandler(webGraphQlHandler, Duration.ofSeconds(60));
    }
}
```

### Subscription Controller
```java
@Controller
public class UserSubscriptionController {

    private final Sinks.Many<User> userSink = Sinks.many().multicast().onBackpressureBuffer();

    @SubscriptionMapping
    public Flux<User> userCreated() {
        return userSink.asFlux();
    }

    // Called when user is created
    public void publishUserCreated(User user) {
        userSink.tryEmitNext(user);
    }
}
```

## Error Handling

```java
@ControllerAdvice
public class GraphQLExceptionHandler {

    @GraphQlExceptionHandler
    public GraphQLError handleNotFound(UserNotFoundException ex) {
        return GraphQLError.newError()
            .errorType(ErrorType.NOT_FOUND)
            .message(ex.getMessage())
            .build();
    }

    @GraphQlExceptionHandler
    public GraphQLError handleValidation(ConstraintViolationException ex) {
        return GraphQLError.newError()
            .errorType(ErrorType.BAD_REQUEST)
            .message("Validation error")
            .extensions(Map.of(
                "violations", ex.getConstraintViolations().stream()
                    .map(v -> Map.of(
                        "field", v.getPropertyPath().toString(),
                        "message", v.getMessage()
                    ))
                    .toList()
            ))
            .build();
    }
}
```

## Testing

```java
@SpringBootTest
@AutoConfigureGraphQlTester
class UserControllerTest {

    @Autowired
    private GraphQlTester graphQlTester;

    @Test
    void shouldGetUserById() {
        this.graphQlTester.document("""
            query {
                userById(id: "1") {
                    id
                    name
                    email
                }
            }
            """)
            .execute()
            .path("userById")
            .entity(User.class)
            .satisfies(user -> {
                assertThat(user.getId()).isEqualTo(1L);
                assertThat(user.getName()).isNotBlank();
            });
    }

    @Test
    void shouldCreateUser() {
        this.graphQlTester.document("""
            mutation CreateUser($input: CreateUserInput!) {
                createUser(input: $input) {
                    id
                    name
                    email
                }
            }
            """)
            .variable("input", Map.of(
                "name", "John Doe",
                "email", "john@example.com"
            ))
            .execute()
            .path("createUser.name")
            .entity(String.class)
            .isEqualTo("John Doe");
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use DataLoader for N+1 problems | Fetch nested data eagerly |
| Define clear schema contracts | Expose internal models directly |
| Implement proper error handling | Return generic errors |
| Use pagination for lists | Return unbounded collections |
| Validate inputs | Trust client input |
| Enable query complexity limits | Allow unlimited query depth |

## Production Checklist

- [ ] GraphiQL disabled in production
- [ ] Query complexity limits configured
- [ ] Authentication/authorization in place
- [ ] Error handling implemented
- [ ] DataLoader for batch loading
- [ ] Tracing and metrics enabled
- [ ] CORS properly configured
