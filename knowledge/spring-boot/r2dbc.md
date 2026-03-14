# Spring Data R2DBC Reference

## Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-r2dbc</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>r2dbc-postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

## Configuration

```yaml
spring:
  r2dbc:
    url: r2dbc:postgresql://localhost:5432/mydb
    username: ${DB_USER}
    password: ${DB_PASSWORD}
    pool:
      initial-size: 5
      max-size: 20
      max-idle-time: 30m
```

## Entity

```java
@Table("users")
public class User {

    @Id
    private Long id;

    @Column("user_name")
    private String name;

    private String email;

    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    @Version
    private Long version;
}
```

## Repository

```java
public interface UserRepository extends ReactiveCrudRepository<User, Long> {

    Mono<User> findByEmail(String email);

    Flux<User> findByNameContaining(String name);

    @Query("SELECT * FROM users WHERE status = :status ORDER BY created_at DESC")
    Flux<User> findByStatus(String status);

    @Query("SELECT * FROM users WHERE created_at > :date")
    Flux<User> findRecentUsers(LocalDateTime date);

    @Modifying
    @Query("UPDATE users SET status = :status WHERE id = :id")
    Mono<Integer> updateStatus(Long id, String status);

    @Query("SELECT COUNT(*) FROM users WHERE status = :status")
    Mono<Long> countByStatus(String status);

    Mono<Boolean> existsByEmail(String email);
}
```

## Service

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;

    public Flux<User> findAll() {
        return userRepository.findAll();
    }

    public Mono<User> findById(Long id) {
        return userRepository.findById(id)
            .switchIfEmpty(Mono.error(new UserNotFoundException(id)));
    }

    public Mono<User> create(CreateUserRequest request) {
        return Mono.just(request)
            .map(this::toUser)
            .flatMap(userRepository::save);
    }

    public Mono<User> update(Long id, UpdateUserRequest request) {
        return userRepository.findById(id)
            .switchIfEmpty(Mono.error(new UserNotFoundException(id)))
            .map(user -> {
                user.setName(request.getName());
                user.setEmail(request.getEmail());
                return user;
            })
            .flatMap(userRepository::save);
    }

    public Mono<Void> delete(Long id) {
        return userRepository.deleteById(id);
    }
}
```

## DatabaseClient

```java
@Repository
@RequiredArgsConstructor
public class UserCustomRepository {

    private final DatabaseClient databaseClient;

    public Flux<User> findWithFilters(UserFilter filter) {
        StringBuilder sql = new StringBuilder("SELECT * FROM users WHERE 1=1");
        Map<String, Object> params = new HashMap<>();

        if (filter.getName() != null) {
            sql.append(" AND name LIKE :name");
            params.put("name", "%" + filter.getName() + "%");
        }

        if (filter.getStatus() != null) {
            sql.append(" AND status = :status");
            params.put("status", filter.getStatus());
        }

        sql.append(" ORDER BY created_at DESC");

        DatabaseClient.GenericExecuteSpec spec = databaseClient.sql(sql.toString());
        for (Map.Entry<String, Object> entry : params.entrySet()) {
            spec = spec.bind(entry.getKey(), entry.getValue());
        }

        return spec.map((row, metadata) -> User.builder()
                .id(row.get("id", Long.class))
                .name(row.get("name", String.class))
                .email(row.get("email", String.class))
                .build())
            .all();
    }

    public Mono<Void> batchInsert(List<User> users) {
        return Flux.fromIterable(users)
            .flatMap(user -> databaseClient.sql(
                    "INSERT INTO users (name, email) VALUES (:name, :email)")
                .bind("name", user.getName())
                .bind("email", user.getEmail())
                .fetch()
                .rowsUpdated())
            .then();
    }
}
```

## Transactions

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final InventoryRepository inventoryRepository;
    private final TransactionalOperator transactionalOperator;

    // Declarative
    @Transactional
    public Mono<Order> createOrder(CreateOrderRequest request) {
        return inventoryRepository.decreaseStock(request.getProductId(), request.getQuantity())
            .then(orderRepository.save(toOrder(request)));
    }

    // Programmatic
    public Mono<Order> createOrderProgrammatic(CreateOrderRequest request) {
        return inventoryRepository.decreaseStock(request.getProductId(), request.getQuantity())
            .then(orderRepository.save(toOrder(request)))
            .as(transactionalOperator::transactional);
    }
}
```

## Pagination

```java
public interface UserRepository extends ReactiveCrudRepository<User, Long> {

    Flux<User> findAllBy(Pageable pageable);

    @Query("SELECT * FROM users WHERE status = :status LIMIT :limit OFFSET :offset")
    Flux<User> findByStatusPaged(String status, int limit, int offset);
}

@Service
public class UserService {

    public Mono<Page<User>> findAll(Pageable pageable) {
        return userRepository.findAllBy(pageable)
            .collectList()
            .zipWith(userRepository.count())
            .map(tuple -> new PageImpl<>(tuple.getT1(), pageable, tuple.getT2()));
    }
}
```

## Relations (Manual)

```java
@Service
@RequiredArgsConstructor
public class UserWithOrdersService {

    private final UserRepository userRepository;
    private final OrderRepository orderRepository;

    public Mono<UserWithOrders> findUserWithOrders(Long userId) {
        return userRepository.findById(userId)
            .zipWith(orderRepository.findByUserId(userId).collectList())
            .map(tuple -> new UserWithOrders(tuple.getT1(), tuple.getT2()));
    }

    public Flux<UserWithOrders> findAllWithOrders() {
        return userRepository.findAll()
            .flatMap(user ->
                orderRepository.findByUserId(user.getId())
                    .collectList()
                    .map(orders -> new UserWithOrders(user, orders))
            );
    }
}
```

## Auditing

```java
@Configuration
@EnableR2dbcAuditing
public class R2dbcConfig {

    @Bean
    public ReactiveAuditorAware<String> auditorAware() {
        return () -> ReactiveSecurityContextHolder.getContext()
            .map(SecurityContext::getAuthentication)
            .filter(Authentication::isAuthenticated)
            .map(Authentication::getName)
            .switchIfEmpty(Mono.just("system"));
    }
}

@Table("users")
public class User {
    @Id
    private Long id;

    @CreatedBy
    private String createdBy;

    @LastModifiedBy
    private String updatedBy;

    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;
}
```

## Testing

```java
@DataR2dbcTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class UserRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @Autowired
    private UserRepository userRepository;

    @Test
    void shouldSaveAndFind() {
        User user = new User();
        user.setName("John");
        user.setEmail("john@example.com");

        StepVerifier.create(userRepository.save(user))
            .assertNext(saved -> {
                assertNotNull(saved.getId());
                assertEquals("John", saved.getName());
            })
            .verifyComplete();
    }

    @Test
    void shouldFindByEmail() {
        StepVerifier.create(userRepository.findByEmail("john@example.com"))
            .assertNext(user -> assertEquals("John", user.getName()))
            .verifyComplete();
    }
}
```
