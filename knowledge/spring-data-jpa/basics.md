# Spring Data JPA

> Official Documentation: https://docs.spring.io/spring-data/jpa/reference/

## Overview

Spring Data JPA provides repository support for the Jakarta Persistence API (JPA), simplifying data access layer implementation through repository interfaces, derived query methods, and declarative transaction management.

---

## Repository Interfaces

### Core Repository Hierarchy

```java
// Base repository - provides basic CRUD
public interface CrudRepository<T, ID> extends Repository<T, ID> {
    <S extends T> S save(S entity);
    Optional<T> findById(ID id);
    Iterable<T> findAll();
    long count();
    void deleteById(ID id);
    boolean existsById(ID id);
}

// Adds pagination and sorting
public interface PagingAndSortingRepository<T, ID> extends CrudRepository<T, ID> {
    Iterable<T> findAll(Sort sort);
    Page<T> findAll(Pageable pageable);
}

// JPA-specific operations
public interface JpaRepository<T, ID> extends PagingAndSortingRepository<T, ID> {
    List<T> findAll();
    List<T> findAllById(Iterable<ID> ids);
    <S extends T> List<S> saveAll(Iterable<S> entities);
    void flush();
    <S extends T> S saveAndFlush(S entity);
    void deleteAllInBatch();
    T getReferenceById(ID id);
}
```

### Creating a Repository

```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String email;
    private String firstName;
    private String lastName;
    private LocalDateTime createdAt;

    @Enumerated(EnumType.STRING)
    private UserStatus status;

    @ManyToOne(fetch = FetchType.LAZY)
    private Department department;

    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
    private List<Order> orders = new ArrayList<>();
}

public interface UserRepository extends JpaRepository<User, Long> {
    // Query methods go here
}
```

---

## Query Methods

### Derived Query Methods

Spring Data JPA creates queries from method names automatically:

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // Simple property queries
    Optional<User> findByEmail(String email);
    List<User> findByLastName(String lastName);

    // Multiple conditions
    List<User> findByFirstNameAndLastName(String firstName, String lastName);
    List<User> findByFirstNameOrLastName(String firstName, String lastName);

    // Comparison operators
    List<User> findByCreatedAtAfter(LocalDateTime date);
    List<User> findByCreatedAtBefore(LocalDateTime date);
    List<User> findByCreatedAtBetween(LocalDateTime start, LocalDateTime end);
    List<User> findByAgeLessThan(int age);
    List<User> findByAgeGreaterThanEqual(int age);

    // String matching
    List<User> findByEmailContaining(String text);
    List<User> findByEmailStartingWith(String prefix);
    List<User> findByEmailEndingWith(String suffix);
    List<User> findByEmailLike(String pattern);  // Use % for wildcards
    List<User> findByEmailIgnoreCase(String email);

    // Null checks
    List<User> findByMiddleNameIsNull();
    List<User> findByMiddleNameIsNotNull();

    // Boolean properties
    List<User> findByActiveTrue();
    List<User> findByActiveFalse();

    // Collection membership
    List<User> findByStatusIn(Collection<UserStatus> statuses);
    List<User> findByStatusNotIn(Collection<UserStatus> statuses);

    // Nested properties (traverses relationships)
    List<User> findByDepartmentName(String deptName);
    List<User> findByOrdersProductName(String productName);

    // Ordering
    List<User> findByStatusOrderByCreatedAtDesc(UserStatus status);
    List<User> findByStatusOrderByLastNameAscFirstNameAsc(UserStatus status);

    // Limiting results
    User findFirstByOrderByCreatedAtDesc();
    List<User> findTop10ByStatusOrderByCreatedAtDesc(UserStatus status);
    List<User> findFirst5ByDepartmentId(Long deptId);

    // Distinct
    List<User> findDistinctByLastName(String lastName);

    // Count and existence
    long countByStatus(UserStatus status);
    boolean existsByEmail(String email);

    // Delete
    void deleteByStatus(UserStatus status);
    long deleteByCreatedAtBefore(LocalDateTime date);
}
```

### Query Method Keywords Reference

| Keyword | Sample | JPQL Equivalent |
|---------|--------|-----------------|
| `And` | `findByFirstNameAndLastName` | `WHERE x.firstName = ?1 AND x.lastName = ?2` |
| `Or` | `findByFirstNameOrLastName` | `WHERE x.firstName = ?1 OR x.lastName = ?2` |
| `Is`, `Equals` | `findByFirstName`, `findByFirstNameIs` | `WHERE x.firstName = ?1` |
| `Between` | `findByStartDateBetween` | `WHERE x.startDate BETWEEN ?1 AND ?2` |
| `LessThan` | `findByAgeLessThan` | `WHERE x.age < ?1` |
| `LessThanEqual` | `findByAgeLessThanEqual` | `WHERE x.age <= ?1` |
| `GreaterThan` | `findByAgeGreaterThan` | `WHERE x.age > ?1` |
| `GreaterThanEqual` | `findByAgeGreaterThanEqual` | `WHERE x.age >= ?1` |
| `After` | `findByStartDateAfter` | `WHERE x.startDate > ?1` |
| `Before` | `findByStartDateBefore` | `WHERE x.startDate < ?1` |
| `IsNull`, `Null` | `findByAgeIsNull` | `WHERE x.age IS NULL` |
| `IsNotNull`, `NotNull` | `findByAgeNotNull` | `WHERE x.age IS NOT NULL` |
| `Like` | `findByFirstNameLike` | `WHERE x.firstName LIKE ?1` |
| `NotLike` | `findByFirstNameNotLike` | `WHERE x.firstName NOT LIKE ?1` |
| `StartingWith` | `findByFirstNameStartingWith` | `WHERE x.firstName LIKE ?1%` |
| `EndingWith` | `findByFirstNameEndingWith` | `WHERE x.firstName LIKE %?1` |
| `Containing` | `findByFirstNameContaining` | `WHERE x.firstName LIKE %?1%` |
| `OrderBy` | `findByAgeOrderByLastNameDesc` | `WHERE x.age = ?1 ORDER BY x.lastName DESC` |
| `Not` | `findByLastNameNot` | `WHERE x.lastName <> ?1` |
| `In` | `findByAgeIn(Collection ages)` | `WHERE x.age IN ?1` |
| `NotIn` | `findByAgeNotIn(Collection ages)` | `WHERE x.age NOT IN ?1` |
| `True` | `findByActiveTrue()` | `WHERE x.active = true` |
| `False` | `findByActiveFalse()` | `WHERE x.active = false` |
| `IgnoreCase` | `findByFirstNameIgnoreCase` | `WHERE UPPER(x.firstName) = UPPER(?1)` |

---

## @Query Annotation

### JPQL Queries

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // Basic JPQL
    @Query("SELECT u FROM User u WHERE u.status = :status")
    List<User> findByStatusCustom(@Param("status") UserStatus status);

    // With multiple parameters
    @Query("SELECT u FROM User u WHERE u.firstName = :firstName AND u.lastName = :lastName")
    List<User> findByFullName(@Param("firstName") String firstName,
                              @Param("lastName") String lastName);

    // Positional parameters
    @Query("SELECT u FROM User u WHERE u.email = ?1")
    Optional<User> findByEmailPositional(String email);

    // JOIN queries
    @Query("SELECT u FROM User u JOIN u.department d WHERE d.name = :deptName")
    List<User> findByDepartmentNameQuery(@Param("deptName") String deptName);

    // LEFT JOIN FETCH (eager loading)
    @Query("SELECT u FROM User u LEFT JOIN FETCH u.orders WHERE u.id = :id")
    Optional<User> findByIdWithOrders(@Param("id") Long id);

    // Aggregation
    @Query("SELECT COUNT(u) FROM User u WHERE u.status = :status")
    long countByStatusQuery(@Param("status") UserStatus status);

    @Query("SELECT u.status, COUNT(u) FROM User u GROUP BY u.status")
    List<Object[]> countByStatusGrouped();

    // LIKE with CONCAT
    @Query("SELECT u FROM User u WHERE u.email LIKE CONCAT('%', :domain)")
    List<User> findByEmailDomain(@Param("domain") String domain);

    // IN clause
    @Query("SELECT u FROM User u WHERE u.id IN :ids")
    List<User> findByIdIn(@Param("ids") Collection<Long> ids);

    // Subqueries
    @Query("SELECT u FROM User u WHERE u.department.id IN " +
           "(SELECT d.id FROM Department d WHERE d.active = true)")
    List<User> findUsersInActiveDepartments();

    // Modifying queries
    @Modifying
    @Query("UPDATE User u SET u.status = :status WHERE u.id = :id")
    int updateStatus(@Param("id") Long id, @Param("status") UserStatus status);

    @Modifying
    @Query("DELETE FROM User u WHERE u.status = :status")
    int deleteByStatusQuery(@Param("status") UserStatus status);

    @Modifying(clearAutomatically = true)  // Clears persistence context after
    @Query("UPDATE User u SET u.lastLogin = :date WHERE u.id = :id")
    void updateLastLogin(@Param("id") Long id, @Param("date") LocalDateTime date);
}
```

### Native SQL Queries

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // Native SQL query
    @Query(value = "SELECT * FROM users WHERE email = :email", nativeQuery = true)
    Optional<User> findByEmailNative(@Param("email") String email);

    // Native with pagination
    @Query(value = "SELECT * FROM users WHERE status = :status",
           countQuery = "SELECT COUNT(*) FROM users WHERE status = :status",
           nativeQuery = true)
    Page<User> findByStatusNative(@Param("status") String status, Pageable pageable);

    // Native with complex joins
    @Query(value = """
           SELECT u.* FROM users u
           INNER JOIN departments d ON u.department_id = d.id
           WHERE d.name = :deptName
           """, nativeQuery = true)
    List<User> findByDepartmentNative(@Param("deptName") String deptName);
}
```

---

## Pagination and Sorting

### Using Pageable

```java
public interface UserRepository extends JpaRepository<User, Long> {
    Page<User> findByStatus(UserStatus status, Pageable pageable);
    Slice<User> findByDepartmentId(Long deptId, Pageable pageable);
    List<User> findByLastName(String lastName, Pageable pageable);
    List<User> findByFirstName(String firstName, Sort sort);
}

// Service usage
@Service
public class UserService {

    public Page<User> getUsers(int page, int size, String sortBy, String direction) {
        Sort sort = Sort.by(Sort.Direction.fromString(direction), sortBy);
        Pageable pageable = PageRequest.of(page, size, sort);
        return userRepository.findAll(pageable);
    }

    // Multiple sort properties
    public Page<User> getUsersSorted() {
        Sort sort = Sort.by(
            Sort.Order.desc("createdAt"),
            Sort.Order.asc("lastName"),
            Sort.Order.asc("firstName")
        );
        return userRepository.findAll(PageRequest.of(0, 20, sort));
    }
}
```

### Page vs Slice vs List

```java
// Page - includes total count (extra COUNT query)
Page<User> page = repository.findByStatus(status, pageable);
page.getContent();        // List<User>
page.getTotalElements();  // Total count
page.getTotalPages();     // Total pages
page.getNumber();         // Current page number
page.getSize();           // Page size
page.hasNext();           // Has next page

// Slice - no total count (more efficient for infinite scroll)
Slice<User> slice = repository.findByDepartmentId(deptId, pageable);
slice.getContent();       // List<User>
slice.hasNext();          // Has next page (fetches n+1 to check)
// No getTotalElements() or getTotalPages()

// List - just returns results with limit
List<User> list = repository.findByLastName(lastName, pageable);
```

---

## Specifications (Dynamic Queries)

### Setup

```java
public interface UserRepository extends JpaRepository<User, Long>,
                                        JpaSpecificationExecutor<User> {
}
```

### Creating Specifications

```java
public class UserSpecifications {

    public static Specification<User> hasStatus(UserStatus status) {
        return (root, query, cb) -> cb.equal(root.get("status"), status);
    }

    public static Specification<User> emailContains(String text) {
        return (root, query, cb) ->
            cb.like(cb.lower(root.get("email")), "%" + text.toLowerCase() + "%");
    }

    public static Specification<User> createdAfter(LocalDateTime date) {
        return (root, query, cb) -> cb.greaterThan(root.get("createdAt"), date);
    }

    public static Specification<User> createdBetween(LocalDateTime start, LocalDateTime end) {
        return (root, query, cb) -> cb.between(root.get("createdAt"), start, end);
    }

    public static Specification<User> inDepartment(Long departmentId) {
        return (root, query, cb) ->
            cb.equal(root.get("department").get("id"), departmentId);
    }

    // JOIN example
    public static Specification<User> hasDepartmentName(String deptName) {
        return (root, query, cb) -> {
            Join<User, Department> dept = root.join("department");
            return cb.equal(dept.get("name"), deptName);
        };
    }
}
```

### Using Specifications

```java
@Service
public class UserService {

    public List<User> searchUsers(UserSearchCriteria criteria) {
        Specification<User> spec = Specification.where(null);

        if (criteria.getStatus() != null) {
            spec = spec.and(UserSpecifications.hasStatus(criteria.getStatus()));
        }

        if (criteria.getEmail() != null) {
            spec = spec.and(UserSpecifications.emailContains(criteria.getEmail()));
        }

        if (criteria.getStartDate() != null && criteria.getEndDate() != null) {
            spec = spec.and(UserSpecifications.createdBetween(
                criteria.getStartDate(), criteria.getEndDate()));
        }

        return userRepository.findAll(spec);
    }

    // With pagination
    public Page<User> searchUsersPaged(UserSearchCriteria criteria, Pageable pageable) {
        Specification<User> spec = buildSpecification(criteria);
        return userRepository.findAll(spec, pageable);
    }
}
```

---

## Projections

### Interface-Based Projections

```java
// Closed projection (only specified fields)
public interface UserSummary {
    String getFirstName();
    String getLastName();
    String getEmail();
}

// Open projection (with SpEL expressions)
public interface UserFullName {
    @Value("#{target.firstName + ' ' + target.lastName}")
    String getFullName();
    String getEmail();
}

// Nested projection
public interface UserWithDepartment {
    String getFirstName();
    String getLastName();
    DepartmentSummary getDepartment();

    interface DepartmentSummary {
        String getName();
    }
}

// Usage in repository
public interface UserRepository extends JpaRepository<User, Long> {
    List<UserSummary> findByStatus(UserStatus status);
    <T> List<T> findByStatus(UserStatus status, Class<T> type);  // Dynamic
}
```

### Class-Based Projections (DTOs)

```java
public class UserDTO {
    private final String firstName;
    private final String lastName;
    private final String email;

    public UserDTO(String firstName, String lastName, String email) {
        this.firstName = firstName;
        this.lastName = lastName;
        this.email = email;
    }
}

// Using @Query with constructor expression
@Query("SELECT new com.example.dto.UserDTO(u.firstName, u.lastName, u.email) " +
       "FROM User u WHERE u.status = :status")
List<UserDTO> findUserDTOsByStatus(@Param("status") UserStatus status);
```

---

## Auditing

### Setup

```java
@Configuration
@EnableJpaAuditing
public class JpaConfig {

    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext())
            .map(SecurityContext::getAuthentication)
            .filter(Authentication::isAuthenticated)
            .map(Authentication::getName);
    }
}
```

### Auditable Entity

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class AuditableEntity {

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    @CreatedBy
    @Column(updatable = false)
    private String createdBy;

    @LastModifiedBy
    private String updatedBy;

    @Version
    private Long version;
}

@Entity
public class User extends AuditableEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String email;
}
```

---

## Custom Repository Implementation

```java
// Custom interface
public interface UserRepositoryCustom {
    List<User> findUsersWithComplexCriteria(UserSearchCriteria criteria);
}

// Implementation (must be named {RepositoryInterface}Impl)
@Repository
public class UserRepositoryCustomImpl implements UserRepositoryCustom {

    @PersistenceContext
    private EntityManager entityManager;

    @Override
    public List<User> findUsersWithComplexCriteria(UserSearchCriteria criteria) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<User> query = cb.createQuery(User.class);
        Root<User> root = query.from(User.class);

        List<Predicate> predicates = new ArrayList<>();
        // Build predicates...

        query.where(predicates.toArray(new Predicate[0]));
        return entityManager.createQuery(query).getResultList();
    }
}

// Extend both interfaces
public interface UserRepository extends JpaRepository<User, Long>,
                                        UserRepositoryCustom {
}
```

---

## Best Practices

1. **Use derived queries for simple cases** - Type-safe and easy to refactor
2. **Use @Query for complex queries** - More readable than long method names
3. **Use Specifications for dynamic queries** - Combine conditions at runtime
4. **Use projections to reduce data transfer** - Don't fetch entire entities when you only need a few fields
5. **Always use pagination for list queries** - Prevents memory issues
6. **Prefer Slice over Page** - Unless you need total count
7. **Use FETCH JOIN for N+1 prevention** - Or configure entity graphs
8. **Enable auditing** - Track who and when entities were modified

## Common Pitfalls

1. **N+1 queries** - Use `JOIN FETCH` or `@EntityGraph`
2. **Lazy loading exceptions** - Keep transactions open or use DTOs
3. **Missing @Modifying** - Required for UPDATE/DELETE queries
4. **Case sensitivity** - Use `IgnoreCase` in method names
5. **Null parameters** - Handle nulls in Specifications
6. **Large IN clauses** - Consider batch queries instead
