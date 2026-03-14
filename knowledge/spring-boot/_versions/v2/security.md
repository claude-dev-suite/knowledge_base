# Spring Boot 2 → Security Delta

## Not Available in Spring Boot 2 / Spring Security 5

- Lambda DSL as primary configuration style
- `requestMatchers()` method
- Built-in OAuth2 Authorization Server
- SecurityFilterChain as the only configuration approach

## Major Breaking Changes

### WebSecurityConfigurerAdapter Removed

```java
// Spring Boot 3 / Spring Security 6
// WebSecurityConfigurerAdapter is REMOVED
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            );
        return http.build();
    }

    @Bean
    public UserDetailsService userDetailsService() {
        UserDetails user = User.withDefaultPasswordEncoder()
            .username("user")
            .password("password")
            .roles("USER")
            .build();
        return new InMemoryUserDetailsManager(user);
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
                .antMatchers("/public/**").permitAll()
                .anyRequest().authenticated();
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.inMemoryAuthentication()
            .withUser("user")
            .password(passwordEncoder().encode("password"))
            .roles("USER");
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### Method Security

```java
// Spring Boot 3 - @EnableMethodSecurity
@Configuration
@EnableMethodSecurity  // New annotation
public class MethodSecurityConfig {
}

// Supports:
// @PreAuthorize, @PostAuthorize
// @PreFilter, @PostFilter
// @Secured (if securedEnabled = true)

// Spring Boot 2 - @EnableGlobalMethodSecurity
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true, securedEnabled = true)
public class MethodSecurityConfig extends GlobalMethodSecurityConfiguration {
}
```

### URL Matching Changes

```java
// Spring Boot 3 - requestMatchers (uses PathPatternParser)
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/**").authenticated()
    .requestMatchers(HttpMethod.POST, "/users").hasRole("ADMIN")
    .requestMatchers(new AntPathRequestMatcher("/legacy/**")).permitAll()
);

// Spring Boot 2 - antMatchers, mvcMatchers, regexMatchers
http.authorizeRequests()
    .antMatchers("/api/**").authenticated()
    .mvcMatchers(HttpMethod.POST, "/users").hasRole("ADMIN")
    .regexMatchers("/legacy/.*").permitAll();
```

### CSRF Configuration

```java
// Spring Boot 3 - Lambda DSL
http.csrf(csrf -> csrf
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
    .csrfTokenRequestHandler(new CsrfTokenRequestAttributeHandler())
);

// Disable CSRF
http.csrf(AbstractHttpConfigurer::disable);

// Spring Boot 2 - Chained methods
http.csrf()
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse());

// Disable CSRF
http.csrf().disable();
```

### CORS Configuration

```java
// Spring Boot 3
http.cors(cors -> cors
    .configurationSource(request -> {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://example.com"));
        config.setAllowedMethods(List.of("GET", "POST"));
        return config;
    })
);

// Spring Boot 2
http.cors().configurationSource(request -> {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(Arrays.asList("https://example.com"));
    config.setAllowedMethods(Arrays.asList("GET", "POST"));
    return config;
});
```

### OAuth2 Login

```java
// Spring Boot 3
http.oauth2Login(oauth2 -> oauth2
    .loginPage("/login")
    .defaultSuccessUrl("/home")
    .failureUrl("/login?error")
    .userInfoEndpoint(userInfo -> userInfo
        .userService(customOAuth2UserService)
    )
);

// Spring Boot 2
http.oauth2Login()
    .loginPage("/login")
    .defaultSuccessUrl("/home")
    .failureUrl("/login?error")
    .userInfoEndpoint()
        .userService(customOAuth2UserService);
```

### JWT Resource Server

```java
// Spring Boot 3
http.oauth2ResourceServer(oauth2 -> oauth2
    .jwt(jwt -> jwt
        .decoder(jwtDecoder())
        .jwtAuthenticationConverter(jwtAuthConverter())
    )
);

// Spring Boot 2
http.oauth2ResourceServer()
    .jwt()
        .decoder(jwtDecoder())
        .jwtAuthenticationConverter(jwtAuthConverter());
```

### Session Management

```java
// Spring Boot 3
http.sessionManagement(session -> session
    .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
    .maximumSessions(1)
    .maxSessionsPreventsLogin(true)
);

// Spring Boot 2
http.sessionManagement()
    .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
    .maximumSessions(1)
    .maxSessionsPreventsLogin(true);
```

## Still Current in Spring Boot 2

- @PreAuthorize, @PostAuthorize annotations
- UserDetailsService interface
- PasswordEncoder interface
- OAuth2 client support
- Basic authentication
- Form login
- Remember-me authentication

## Migration Patterns

### From WebSecurityConfigurerAdapter

```java
// Step 1: Extract SecurityFilterChain bean
// Step 2: Extract UserDetailsService bean
// Step 3: Extract AuthenticationManager bean if needed
// Step 4: Replace antMatchers with requestMatchers
// Step 5: Use Lambda DSL for all configurations

@Bean
public AuthenticationManager authenticationManager(
        AuthenticationConfiguration config) throws Exception {
    return config.getAuthenticationManager();
}
```

## Recommendations for Spring Boot 2 Users

1. **Start with SecurityFilterChain** - Even in Boot 2.7+
2. **Use Lambda DSL** - Available in Security 5.7+
3. **Avoid deprecated methods** - `antMatchers` etc.
4. **Test authorization rules** - URL matching may differ
5. **Plan OAuth2 migration** - If using custom implementations

