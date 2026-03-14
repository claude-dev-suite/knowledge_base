# Spring Boot 2 → Basics Delta

## Not Available in Spring Boot 2

- Native compilation with GraalVM (experimental only)
- Jakarta EE 10 APIs
- Built-in observability with Micrometer Tracing
- Problem Details (RFC 7807) built-in
- HTTP interfaces (declarative HTTP clients)
- Virtual threads support (Java 21)

## Major Changes: javax → jakarta

```java
// Spring Boot 3 - Jakarta EE 10
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.validation.constraints.NotNull;
import jakarta.servlet.http.HttpServletRequest;

// Spring Boot 2 - Java EE 8
import javax.persistence.Entity;
import javax.persistence.Id;
import javax.validation.constraints.NotNull;
import javax.servlet.http.HttpServletRequest;
```

### Package Migration Map

| Spring Boot 2 (javax) | Spring Boot 3 (jakarta) |
|----------------------|------------------------|
| `javax.persistence.*` | `jakarta.persistence.*` |
| `javax.validation.*` | `jakarta.validation.*` |
| `javax.servlet.*` | `jakarta.servlet.*` |
| `javax.annotation.*` | `jakarta.annotation.*` |
| `javax.transaction.*` | `jakarta.transaction.*` |

## Syntax Differences

### Security Configuration

```java
// Spring Boot 3 / Spring Security 6
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2Login(Customizer.withDefaults())
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
            );

        return http.build();
    }
}

// Spring Boot 2 / Spring Security 5
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests()
                .antMatchers("/api/public/**").permitAll()
                .antMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            .and()
            .oauth2Login()
            .and()
            .csrf()
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse());
    }
}
```

### URL Matching

```java
// Spring Boot 3 - requestMatchers (PathPatternParser)
.requestMatchers("/api/**")
.requestMatchers(HttpMethod.GET, "/users/**")

// Spring Boot 2 - antMatchers (AntPathMatcher)
.antMatchers("/api/**")
.antMatchers(HttpMethod.GET, "/users/**")
```

### Trailing Slash

```java
// Spring Boot 3 - Trailing slash NOT matched by default
@GetMapping("/users")  // Matches /users but NOT /users/

// To enable trailing slash matching:
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void configurePathMatch(PathMatchConfigurer configurer) {
        configurer.setUseTrailingSlashMatch(true);
    }
}

// Spring Boot 2 - Trailing slash matched by default
@GetMapping("/users")  // Matches /users AND /users/
```

### HTTP Interfaces (Spring Boot 3 only)

```java
// Spring Boot 3 - Declarative HTTP clients
public interface UserClient {
    @GetExchange("/users/{id}")
    User getUser(@PathVariable Long id);

    @PostExchange("/users")
    User createUser(@RequestBody User user);
}

@Bean
UserClient userClient(WebClient.Builder builder) {
    WebClient client = builder.baseUrl("http://api.example.com").build();
    HttpServiceProxyFactory factory = HttpServiceProxyFactory
        .builderFor(WebClientAdapter.create(client))
        .build();
    return factory.createClient(UserClient.class);
}

// Spring Boot 2 - Use WebClient or RestTemplate manually
@Service
public class UserService {
    private final WebClient webClient;

    public User getUser(Long id) {
        return webClient.get()
            .uri("/users/{id}", id)
            .retrieve()
            .bodyToMono(User.class)
            .block();
    }
}
```

### Observability

```java
// Spring Boot 3 - Built-in with Micrometer Tracing
@RestController
public class UserController {
    // Tracing automatic with spring-boot-starter-actuator
    // + micrometer-tracing-bridge-* dependency

    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }
}

// Spring Boot 2 - Spring Cloud Sleuth required
// Add: spring-cloud-starter-sleuth dependency
```

### Problem Details (RFC 7807)

```java
// Spring Boot 3 - Built-in support
@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {
    // ProblemDetail is built-in
    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND,
            ex.getMessage()
        );
        problem.setTitle("Resource Not Found");
        problem.setProperty("resourceId", ex.getResourceId());
        return problem;
    }
}

// Spring Boot 2 - Manual implementation or library
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.NOT_FOUND.value(),
            "Resource Not Found",
            ex.getMessage()
        );
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
}
```

## Requirements

| Aspect | Spring Boot 2 | Spring Boot 3 |
|--------|--------------|---------------|
| Java | 8, 11, 17 | 17+ (21 for virtual threads) |
| Spring Framework | 5.x | 6.x |
| Hibernate | 5.x | 6.x |
| Jakarta EE | N/A (uses javax) | 10 |
| Gradle | 6.8+ | 7.5+ |
| Maven | 3.5+ | 3.6.3+ |

## Still Current in Spring Boot 2

- Auto-configuration
- Starters (with javax packages)
- Spring MVC / WebFlux
- Spring Data JPA
- Actuator endpoints
- Profile-based configuration
- Application properties/yaml

## Recommendations for Spring Boot 2 Users

1. **Upgrade Java first** - Minimum Java 17 for Spring Boot 3
2. **Use migration tool**: `spring-boot-migrator` or IDE refactoring
3. **Update dependencies** - Check for Jakarta-compatible versions
4. **Test thoroughly** - Especially security configuration
5. **Plan for javax→jakarta** - This is the biggest change

## Migration Checklist

1. [ ] Upgrade to Java 17+
2. [ ] Replace all `javax.*` imports with `jakarta.*`
3. [ ] Update Spring Security config (remove WebSecurityConfigurerAdapter)
4. [ ] Replace antMatchers with requestMatchers
5. [ ] Update Hibernate 5 → 6 queries if needed
6. [ ] Update test dependencies
7. [ ] Check third-party library compatibility

