# Spring WebFlux Reference

## Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

## Mono and Flux Basics

```java
// Mono: 0 or 1 element
Mono<User> user = Mono.just(new User("John"));
Mono<User> empty = Mono.empty();
Mono<User> error = Mono.error(new RuntimeException("Error"));

// Flux: 0 to N elements
Flux<Integer> numbers = Flux.just(1, 2, 3, 4, 5);
Flux<Integer> range = Flux.range(1, 10);
Flux<Long> interval = Flux.interval(Duration.ofSeconds(1));
```

## Reactive Controller

```java
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    @GetMapping
    public Flux<User> findAll() {
        return userService.findAll();
    }

    @GetMapping("/{id}")
    public Mono<ResponseEntity<User>> findById(@PathVariable String id) {
        return userService.findById(id)
            .map(ResponseEntity::ok)
            .defaultIfEmpty(ResponseEntity.notFound().build());
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono<User> create(@RequestBody @Valid Mono<CreateUserRequest> request) {
        return request.flatMap(userService::create);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public Mono<Void> delete(@PathVariable String id) {
        return userService.delete(id);
    }
}
```

## Reactive Service

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;

    public Flux<User> findAll() {
        return userRepository.findAll();
    }

    public Mono<User> findById(String id) {
        return userRepository.findById(id)
            .switchIfEmpty(Mono.error(new UserNotFoundException(id)));
    }

    public Mono<User> create(CreateUserRequest request) {
        return Mono.just(request)
            .map(this::toUser)
            .flatMap(userRepository::save);
    }

    public Mono<User> update(String id, UpdateUserRequest request) {
        return userRepository.findById(id)
            .flatMap(user -> {
                user.setName(request.getName());
                return userRepository.save(user);
            });
    }
}
```

## WebClient

```java
@Service
public class ExternalApiService {

    private final WebClient webClient;

    public ExternalApiService(WebClient.Builder builder) {
        this.webClient = builder
            .baseUrl("https://api.example.com")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .build();
    }

    public Mono<User> getUser(String id) {
        return webClient.get()
            .uri("/users/{id}", id)
            .retrieve()
            .bodyToMono(User.class);
    }

    public Flux<User> getAllUsers() {
        return webClient.get()
            .uri("/users")
            .retrieve()
            .bodyToFlux(User.class);
    }

    public Mono<User> createUser(CreateUserRequest request) {
        return webClient.post()
            .uri("/users")
            .bodyValue(request)
            .retrieve()
            .bodyToMono(User.class);
    }

    // With error handling
    public Mono<User> getUserSafe(String id) {
        return webClient.get()
            .uri("/users/{id}", id)
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError,
                response -> Mono.error(new UserNotFoundException(id)))
            .onStatus(HttpStatusCode::is5xxServerError,
                response -> Mono.error(new ServiceException("Server error")))
            .bodyToMono(User.class)
            .timeout(Duration.ofSeconds(5))
            .retryWhen(Retry.backoff(3, Duration.ofSeconds(1)));
    }
}
```

## Operators

```java
// Transform
Flux<String> names = users.map(User::getName);

// Filter
Flux<User> adults = users.filter(u -> u.getAge() >= 18);

// FlatMap (async transformation)
Flux<Order> orders = users.flatMap(user -> orderService.findByUserId(user.getId()));

// Combine
Mono<UserWithOrders> combined = Mono.zip(
    userService.findById(id),
    orderService.findByUserId(id).collectList(),
    (user, orders) -> new UserWithOrders(user, orders)
);

// Error handling
Mono<User> userWithFallback = userService.findById(id)
    .onErrorReturn(new User("default"))
    .onErrorResume(e -> Mono.just(new User("fallback")));

// Timeout
Mono<User> withTimeout = userService.findById(id)
    .timeout(Duration.ofSeconds(5));

// Retry
Mono<User> withRetry = userService.findById(id)
    .retryWhen(Retry.backoff(3, Duration.ofMillis(100)));
```

## Server-Sent Events

```java
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<Notification>> stream() {
    return notificationService.getNotifications()
        .map(notification -> ServerSentEvent.<Notification>builder()
            .id(notification.getId())
            .event("notification")
            .data(notification)
            .build());
}
```

## Functional Endpoints

```java
@Configuration
public class RouterConfig {

    @Bean
    public RouterFunction<ServerResponse> routes(UserHandler handler) {
        return RouterFunctions.route()
            .GET("/api/users", handler::findAll)
            .GET("/api/users/{id}", handler::findById)
            .POST("/api/users", handler::create)
            .PUT("/api/users/{id}", handler::update)
            .DELETE("/api/users/{id}", handler::delete)
            .build();
    }
}

@Component
@RequiredArgsConstructor
public class UserHandler {

    private final UserService userService;

    public Mono<ServerResponse> findAll(ServerRequest request) {
        return ServerResponse.ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(userService.findAll(), User.class);
    }

    public Mono<ServerResponse> findById(ServerRequest request) {
        String id = request.pathVariable("id");
        return userService.findById(id)
            .flatMap(user -> ServerResponse.ok().bodyValue(user))
            .switchIfEmpty(ServerResponse.notFound().build());
    }

    public Mono<ServerResponse> create(ServerRequest request) {
        return request.bodyToMono(CreateUserRequest.class)
            .flatMap(userService::create)
            .flatMap(user -> ServerResponse
                .created(URI.create("/api/users/" + user.getId()))
                .bodyValue(user));
    }
}
```

## Exception Handling

```java
@RestControllerAdvice
public class GlobalErrorHandler {

    @ExceptionHandler(UserNotFoundException.class)
    public Mono<ResponseEntity<ProblemDetail>> handleNotFound(UserNotFoundException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND, ex.getMessage()
        );
        return Mono.just(ResponseEntity.status(HttpStatus.NOT_FOUND).body(problem));
    }
}
```

## Testing

```java
@WebFluxTest(UserController.class)
class UserControllerTest {

    @Autowired
    private WebTestClient webClient;

    @MockBean
    private UserService userService;

    @Test
    void shouldReturnUser() {
        when(userService.findById("1"))
            .thenReturn(Mono.just(new User("1", "John")));

        webClient.get()
            .uri("/api/users/1")
            .exchange()
            .expectStatus().isOk()
            .expectBody(User.class)
            .value(user -> assertEquals("John", user.getName()));
    }

    @Test
    void shouldReturnNotFound() {
        when(userService.findById("999"))
            .thenReturn(Mono.empty());

        webClient.get()
            .uri("/api/users/999")
            .exchange()
            .expectStatus().isNotFound();
    }
}
```
