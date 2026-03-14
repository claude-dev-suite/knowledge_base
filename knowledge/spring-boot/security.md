# Spring Boot Security - Comprehensive Guide

This comprehensive guide covers Spring Security fundamentals, authentication mechanisms, authorization patterns, and security best practices for Spring Boot applications.

---

## Table of Contents

1. [Security Configuration (SecurityFilterChain)](#1-security-configuration-securityfilterchain)
2. [Authentication (AuthenticationManager, AuthenticationProvider)](#2-authentication-authenticationmanager-authenticationprovider)
3. [UserDetailsService and UserDetails](#3-userdetailsservice-and-userdetails)
4. [Password Encoding (BCrypt)](#4-password-encoding-bcrypt)
5. [JWT Authentication](#5-jwt-authentication)
6. [Form-based Login](#6-form-based-login)
7. [HTTP Basic Authentication](#7-http-basic-authentication)
8. [Authorization (hasRole, hasAuthority)](#8-authorization-hasrole-hasauthority)
9. [Method Security (@PreAuthorize, @PostAuthorize, @Secured)](#9-method-security-preauthorize-postauthorize-secured)
10. [CORS Configuration](#10-cors-configuration)
11. [CSRF Protection](#11-csrf-protection)
12. [Session Management](#12-session-management)
13. [OAuth2 Resource Server](#13-oauth2-resource-server)
14. [OAuth2 Client](#14-oauth2-client)
15. [Security Filters](#15-security-filters)
16. [Testing Security (@WithMockUser)](#16-testing-security-withmockuser)
17. [Best Practices](#17-best-practices)

---

## 1. Security Configuration (SecurityFilterChain)

Spring Security 6.x uses `SecurityFilterChain` as the primary way to configure HTTP security. The older `WebSecurityConfigurerAdapter` is deprecated.

### Basic Security Configuration

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**", "/api/health").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/**").authenticated()
                .anyRequest().authenticated()
            )
            .formLogin(Customizer.withDefaults())
            .build();
    }
}
```

### Multiple Security Filter Chains

You can define multiple `SecurityFilterChain` beans for different URL patterns:

```java
@Configuration
@EnableWebSecurity
public class MultipleSecurityConfig {

    @Bean
    @Order(1)
    public SecurityFilterChain apiSecurityFilterChain(HttpSecurity http) throws Exception {
        return http
            .securityMatcher("/api/**")
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .build();
    }

    @Bean
    @Order(2)
    public SecurityFilterChain webSecurityFilterChain(HttpSecurity http) throws Exception {
        return http
            .securityMatcher("/**")
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/home", "/css/**", "/js/**").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard")
                .permitAll()
            )
            .logout(logout -> logout
                .logoutSuccessUrl("/")
                .permitAll()
            )
            .build();
    }
}
```

### Customizing Security Headers

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
        .headers(headers -> headers
            .frameOptions(frame -> frame.sameOrigin())
            .xssProtection(xss -> xss.disable())
            .contentSecurityPolicy(csp ->
                csp.policyDirectives("default-src 'self'; script-src 'self'"))
            .httpStrictTransportSecurity(hsts -> hsts
                .includeSubDomains(true)
                .maxAgeInSeconds(31536000))
        )
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .build();
}
```

### Request Matchers with HTTP Methods

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
            .requestMatchers(HttpMethod.POST, "/api/products/**").hasRole("ADMIN")
            .requestMatchers(HttpMethod.PUT, "/api/products/**").hasAnyRole("ADMIN", "MANAGER")
            .requestMatchers(HttpMethod.DELETE, "/api/products/**").hasRole("ADMIN")
            .requestMatchers("/api/orders/**").authenticated()
            .anyRequest().denyAll()
        )
        .build();
}
```

### Ignoring Static Resources

```java
@Bean
public WebSecurityCustomizer webSecurityCustomizer() {
    return web -> web.ignoring()
        .requestMatchers("/static/**", "/images/**", "/favicon.ico");
}
```

---

## 2. Authentication (AuthenticationManager, AuthenticationProvider)

### AuthenticationManager

The `AuthenticationManager` is the main strategy interface for authentication. It has a single method `authenticate()` that attempts to authenticate an `Authentication` object.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration authConfig) throws Exception {
        return authConfig.getAuthenticationManager();
    }
}
```

### Custom AuthenticationManager

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public AuthenticationManager authenticationManager(
            UserDetailsService userDetailsService,
            PasswordEncoder passwordEncoder) {
        DaoAuthenticationProvider authProvider = new DaoAuthenticationProvider();
        authProvider.setUserDetailsService(userDetailsService);
        authProvider.setPasswordEncoder(passwordEncoder);

        return new ProviderManager(authProvider);
    }
}
```

### AuthenticationProvider

`AuthenticationProvider` is a more granular authentication mechanism. Multiple providers can be configured:

```java
@Component
public class CustomAuthenticationProvider implements AuthenticationProvider {

    private final UserDetailsService userDetailsService;
    private final PasswordEncoder passwordEncoder;

    public CustomAuthenticationProvider(
            UserDetailsService userDetailsService,
            PasswordEncoder passwordEncoder) {
        this.userDetailsService = userDetailsService;
        this.passwordEncoder = passwordEncoder;
    }

    @Override
    public Authentication authenticate(Authentication authentication)
            throws AuthenticationException {
        String username = authentication.getName();
        String password = authentication.getCredentials().toString();

        UserDetails user = userDetailsService.loadUserByUsername(username);

        if (!passwordEncoder.matches(password, user.getPassword())) {
            throw new BadCredentialsException("Invalid password");
        }

        // Additional custom validation
        if (!isUserActive(user)) {
            throw new DisabledException("User account is disabled");
        }

        return new UsernamePasswordAuthenticationToken(
            user, password, user.getAuthorities());
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return UsernamePasswordAuthenticationToken.class
            .isAssignableFrom(authentication);
    }

    private boolean isUserActive(UserDetails user) {
        return user.isEnabled() && user.isAccountNonLocked();
    }
}
```

### Multiple Authentication Providers

```java
@Configuration
@EnableWebSecurity
public class MultiProviderSecurityConfig {

    @Bean
    public AuthenticationManager authenticationManager(
            LdapAuthenticationProvider ldapProvider,
            DaoAuthenticationProvider daoProvider) {
        return new ProviderManager(List.of(ldapProvider, daoProvider));
    }

    @Bean
    public DaoAuthenticationProvider daoAuthenticationProvider(
            UserDetailsService userDetailsService,
            PasswordEncoder passwordEncoder) {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        provider.setUserDetailsService(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder);
        return provider;
    }

    @Bean
    public LdapAuthenticationProvider ldapAuthenticationProvider() {
        LdapAuthenticator authenticator = new BindAuthenticator(contextSource());
        authenticator.setUserDnPatterns(new String[]{"uid={0},ou=users"});
        return new LdapAuthenticationProvider(authenticator);
    }
}
```

### Authentication Events

```java
@Component
public class AuthenticationEventListener {

    private static final Logger log = LoggerFactory.getLogger(AuthenticationEventListener.class);

    @EventListener
    public void onSuccess(AuthenticationSuccessEvent event) {
        String username = event.getAuthentication().getName();
        log.info("Successful authentication for user: {}", username);
    }

    @EventListener
    public void onFailure(AbstractAuthenticationFailureEvent event) {
        String username = event.getAuthentication().getName();
        log.warn("Failed authentication for user: {}", username);
    }
}
```

---

## 3. UserDetailsService and UserDetails

### UserDetails Interface

`UserDetails` provides core user information required by Spring Security:

```java
public class CustomUserDetails implements UserDetails {

    private final User user;

    public CustomUserDetails(User user) {
        this.user = user;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return user.getRoles().stream()
            .map(role -> new SimpleGrantedAuthority("ROLE_" + role.getName()))
            .collect(Collectors.toList());
    }

    @Override
    public String getPassword() {
        return user.getPassword();
    }

    @Override
    public String getUsername() {
        return user.getEmail();
    }

    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return !user.isLocked();
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return user.isActive();
    }

    // Custom methods
    public Long getId() {
        return user.getId();
    }

    public String getFullName() {
        return user.getFirstName() + " " + user.getLastName();
    }
}
```

### UserDetailsService Implementation

```java
@Service
@RequiredArgsConstructor
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    @Override
    @Transactional(readOnly = true)
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByEmail(username)
            .orElseThrow(() -> new UsernameNotFoundException(
                "User not found with email: " + username));

        return new CustomUserDetails(user);
    }
}
```

### Advanced UserDetailsService with Caching

```java
@Service
@RequiredArgsConstructor
public class CachingUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;
    private final RoleRepository roleRepository;

    @Override
    @Transactional(readOnly = true)
    @Cacheable(value = "users", key = "#username")
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByEmailWithRoles(username)
            .orElseThrow(() -> new UsernameNotFoundException(
                "User not found: " + username));

        return buildUserDetails(user);
    }

    private UserDetails buildUserDetails(User user) {
        List<GrantedAuthority> authorities = new ArrayList<>();

        // Add roles
        user.getRoles().forEach(role -> {
            authorities.add(new SimpleGrantedAuthority("ROLE_" + role.getName()));

            // Add permissions from roles
            role.getPermissions().forEach(permission ->
                authorities.add(new SimpleGrantedAuthority(permission.getName()))
            );
        });

        return org.springframework.security.core.userdetails.User.builder()
            .username(user.getEmail())
            .password(user.getPassword())
            .authorities(authorities)
            .accountExpired(user.isAccountExpired())
            .accountLocked(user.isLocked())
            .credentialsExpired(user.isCredentialsExpired())
            .disabled(!user.isEnabled())
            .build();
    }

    @CacheEvict(value = "users", key = "#username")
    public void evictUserCache(String username) {
        // Cache is evicted
    }
}
```

### In-Memory UserDetailsService

```java
@Bean
public UserDetailsService inMemoryUserDetailsService(PasswordEncoder passwordEncoder) {
    UserDetails admin = User.builder()
        .username("admin")
        .password(passwordEncoder.encode("admin123"))
        .roles("ADMIN", "USER")
        .build();

    UserDetails user = User.builder()
        .username("user")
        .password(passwordEncoder.encode("user123"))
        .roles("USER")
        .build();

    return new InMemoryUserDetailsManager(admin, user);
}
```

### JDBC UserDetailsService

```java
@Bean
public UserDetailsService jdbcUserDetailsService(DataSource dataSource) {
    JdbcUserDetailsManager manager = new JdbcUserDetailsManager(dataSource);

    manager.setUsersByUsernameQuery(
        "SELECT username, password, enabled FROM users WHERE username = ?");

    manager.setAuthoritiesByUsernameQuery(
        "SELECT u.username, r.role_name FROM users u " +
        "JOIN user_roles ur ON u.id = ur.user_id " +
        "JOIN roles r ON ur.role_id = r.id " +
        "WHERE u.username = ?");

    return manager;
}
```

---

## 4. Password Encoding (BCrypt)

### BCryptPasswordEncoder

BCrypt is the recommended password encoder for Spring Security:

```java
@Configuration
public class PasswordConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### BCrypt with Custom Strength

```java
@Bean
public PasswordEncoder passwordEncoder() {
    // Strength parameter (4-31), default is 10
    // Higher values = more secure but slower
    return new BCryptPasswordEncoder(12);
}
```

### Delegating Password Encoder

Supports multiple encoding formats for migration scenarios:

```java
@Bean
public PasswordEncoder passwordEncoder() {
    String encodingId = "bcrypt";
    Map<String, PasswordEncoder> encoders = new HashMap<>();
    encoders.put("bcrypt", new BCryptPasswordEncoder());
    encoders.put("noop", NoOpPasswordEncoder.getInstance());
    encoders.put("pbkdf2", Pbkdf2PasswordEncoder.defaultsForSpringSecurity_v5_8());
    encoders.put("scrypt", SCryptPasswordEncoder.defaultsForSpringSecurity_v5_8());
    encoders.put("argon2", Argon2PasswordEncoder.defaultsForSpringSecurity_v5_8());

    return new DelegatingPasswordEncoder(encodingId, encoders);
}
```

### Password Encoding in User Registration

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    @Transactional
    public User registerUser(RegisterRequest request) {
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new UserAlreadyExistsException("Email already registered");
        }

        User user = User.builder()
            .email(request.getEmail())
            .password(passwordEncoder.encode(request.getPassword()))
            .firstName(request.getFirstName())
            .lastName(request.getLastName())
            .enabled(true)
            .build();

        return userRepository.save(user);
    }

    @Transactional
    public void changePassword(Long userId, ChangePasswordRequest request) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException("User not found"));

        if (!passwordEncoder.matches(request.getCurrentPassword(), user.getPassword())) {
            throw new InvalidPasswordException("Current password is incorrect");
        }

        user.setPassword(passwordEncoder.encode(request.getNewPassword()));
        userRepository.save(user);
    }
}
```

### Password Validation

```java
@Component
public class PasswordValidator {

    private static final String PASSWORD_PATTERN =
        "^(?=.*[0-9])(?=.*[a-z])(?=.*[A-Z])(?=.*[@#$%^&+=])(?=\\S+$).{8,}$";

    private final Pattern pattern = Pattern.compile(PASSWORD_PATTERN);

    public boolean isValid(String password) {
        return pattern.matcher(password).matches();
    }

    public List<String> validate(String password) {
        List<String> errors = new ArrayList<>();

        if (password.length() < 8) {
            errors.add("Password must be at least 8 characters long");
        }
        if (!password.matches(".*[0-9].*")) {
            errors.add("Password must contain at least one digit");
        }
        if (!password.matches(".*[a-z].*")) {
            errors.add("Password must contain at least one lowercase letter");
        }
        if (!password.matches(".*[A-Z].*")) {
            errors.add("Password must contain at least one uppercase letter");
        }
        if (!password.matches(".*[@#$%^&+=].*")) {
            errors.add("Password must contain at least one special character");
        }

        return errors;
    }
}
```

---

## 5. JWT Authentication

### JWT Dependencies

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.11.5</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.11.5</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.11.5</version>
    <scope>runtime</scope>
</dependency>
```

### JWT Configuration Properties

```yaml
# application.yml
jwt:
  secret: ${JWT_SECRET:your-256-bit-secret-key-here-must-be-at-least-32-chars}
  access-token-expiration: 900000      # 15 minutes
  refresh-token-expiration: 604800000  # 7 days
  issuer: your-application-name
```

### JWT Service Implementation

```java
@Service
@RequiredArgsConstructor
public class JwtService {

    @Value("${jwt.secret}")
    private String secretKey;

    @Value("${jwt.access-token-expiration}")
    private long accessTokenExpiration;

    @Value("${jwt.refresh-token-expiration}")
    private long refreshTokenExpiration;

    @Value("${jwt.issuer}")
    private String issuer;

    public String generateAccessToken(UserDetails userDetails) {
        return generateToken(new HashMap<>(), userDetails, accessTokenExpiration);
    }

    public String generateRefreshToken(UserDetails userDetails) {
        return generateToken(new HashMap<>(), userDetails, refreshTokenExpiration);
    }

    public String generateToken(
            Map<String, Object> extraClaims,
            UserDetails userDetails,
            long expiration) {

        // Add roles to claims
        extraClaims.put("roles", userDetails.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .collect(Collectors.toList()));

        return Jwts.builder()
            .setClaims(extraClaims)
            .setSubject(userDetails.getUsername())
            .setIssuer(issuer)
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + expiration))
            .signWith(getSignInKey(), SignatureAlgorithm.HS256)
            .compact();
    }

    public boolean isTokenValid(String token, UserDetails userDetails) {
        try {
            final String username = extractUsername(token);
            return username.equals(userDetails.getUsername()) && !isTokenExpired(token);
        } catch (JwtException e) {
            return false;
        }
    }

    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    public Date extractExpiration(String token) {
        return extractClaim(token, Claims::getExpiration);
    }

    @SuppressWarnings("unchecked")
    public List<String> extractRoles(String token) {
        return extractClaim(token, claims -> claims.get("roles", List.class));
    }

    public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = extractAllClaims(token);
        return claimsResolver.apply(claims);
    }

    private Claims extractAllClaims(String token) {
        return Jwts.parserBuilder()
            .setSigningKey(getSignInKey())
            .build()
            .parseClaimsJws(token)
            .getBody();
    }

    private Key getSignInKey() {
        byte[] keyBytes = Decoders.BASE64.decode(secretKey);
        return Keys.hmacShaKeyFor(keyBytes);
    }

    private boolean isTokenExpired(String token) {
        return extractExpiration(token).before(new Date());
    }
}
```

### JWT Authentication Filter

```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        final String authHeader = request.getHeader("Authorization");

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        try {
            final String jwt = authHeader.substring(7);
            final String username = jwtService.extractUsername(jwt);

            if (username != null &&
                    SecurityContextHolder.getContext().getAuthentication() == null) {

                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                if (jwtService.isTokenValid(jwt, userDetails)) {
                    UsernamePasswordAuthenticationToken authToken =
                        new UsernamePasswordAuthenticationToken(
                            userDetails,
                            null,
                            userDetails.getAuthorities());

                    authToken.setDetails(
                        new WebAuthenticationDetailsSource().buildDetails(request));

                    SecurityContextHolder.getContext().setAuthentication(authToken);
                }
            }
        } catch (ExpiredJwtException e) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.setContentType("application/json");
            response.getWriter().write("{\"error\": \"Token expired\"}");
            return;
        } catch (JwtException e) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.setContentType("application/json");
            response.getWriter().write("{\"error\": \"Invalid token\"}");
            return;
        }

        filterChain.doFilter(request, response);
    }

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        String path = request.getServletPath();
        return path.startsWith("/api/v1/auth/") ||
               path.startsWith("/public/") ||
               path.startsWith("/swagger-ui/") ||
               path.startsWith("/v3/api-docs/");
    }
}
```

### JWT Security Configuration

```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthFilter;
    private final UserDetailsService userDetailsService;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)
            .cors(Customizer.withDefaults())
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/v1/public/**").permitAll()
                .anyRequest().authenticated())
            .authenticationProvider(authenticationProvider())
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(jwtAuthenticationEntryPoint())
                .accessDeniedHandler(jwtAccessDeniedHandler()))
            .build();
    }

    @Bean
    public AuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        provider.setUserDetailsService(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder());
        return provider;
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public AuthenticationEntryPoint jwtAuthenticationEntryPoint() {
        return (request, response, authException) -> {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.setContentType("application/json");
            response.getWriter().write(
                "{\"error\": \"Unauthorized\", \"message\": \"" +
                authException.getMessage() + "\"}");
        };
    }

    @Bean
    public AccessDeniedHandler jwtAccessDeniedHandler() {
        return (request, response, accessDeniedException) -> {
            response.setStatus(HttpServletResponse.SC_FORBIDDEN);
            response.setContentType("application/json");
            response.getWriter().write(
                "{\"error\": \"Forbidden\", \"message\": \"Access denied\"}");
        };
    }
}
```

### Authentication Controller

```java
@RestController
@RequestMapping("/api/v1/auth")
@RequiredArgsConstructor
public class AuthController {

    private final AuthService authService;

    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(@Valid @RequestBody LoginRequest request) {
        return ResponseEntity.ok(authService.login(request));
    }

    @PostMapping("/register")
    @ResponseStatus(HttpStatus.CREATED)
    public ResponseEntity<AuthResponse> register(@Valid @RequestBody RegisterRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(authService.register(request));
    }

    @PostMapping("/refresh")
    public ResponseEntity<AuthResponse> refresh(@RequestBody RefreshTokenRequest request) {
        return ResponseEntity.ok(authService.refreshToken(request));
    }

    @PostMapping("/logout")
    public ResponseEntity<Void> logout(@RequestBody LogoutRequest request) {
        authService.logout(request);
        return ResponseEntity.noContent().build();
    }
}
```

### Authentication Service

```java
@Service
@RequiredArgsConstructor
public class AuthService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final JwtService jwtService;
    private final AuthenticationManager authenticationManager;
    private final RefreshTokenRepository refreshTokenRepository;

    @Transactional
    public AuthResponse register(RegisterRequest request) {
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new UserAlreadyExistsException("Email already in use");
        }

        User user = User.builder()
            .email(request.getEmail())
            .password(passwordEncoder.encode(request.getPassword()))
            .firstName(request.getFirstName())
            .lastName(request.getLastName())
            .role(Role.USER)
            .enabled(true)
            .build();

        userRepository.save(user);

        CustomUserDetails userDetails = new CustomUserDetails(user);
        String accessToken = jwtService.generateAccessToken(userDetails);
        String refreshToken = jwtService.generateRefreshToken(userDetails);

        saveRefreshToken(user, refreshToken);

        return AuthResponse.builder()
            .accessToken(accessToken)
            .refreshToken(refreshToken)
            .expiresIn(jwtService.extractExpiration(accessToken).getTime())
            .build();
    }

    public AuthResponse login(LoginRequest request) {
        authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(
                request.getEmail(),
                request.getPassword()
            )
        );

        User user = userRepository.findByEmail(request.getEmail())
            .orElseThrow(() -> new UsernameNotFoundException("User not found"));

        CustomUserDetails userDetails = new CustomUserDetails(user);
        String accessToken = jwtService.generateAccessToken(userDetails);
        String refreshToken = jwtService.generateRefreshToken(userDetails);

        revokeAllUserTokens(user);
        saveRefreshToken(user, refreshToken);

        return AuthResponse.builder()
            .accessToken(accessToken)
            .refreshToken(refreshToken)
            .expiresIn(jwtService.extractExpiration(accessToken).getTime())
            .build();
    }

    @Transactional
    public AuthResponse refreshToken(RefreshTokenRequest request) {
        String refreshToken = request.getRefreshToken();
        String username = jwtService.extractUsername(refreshToken);

        User user = userRepository.findByEmail(username)
            .orElseThrow(() -> new UsernameNotFoundException("User not found"));

        RefreshToken storedToken = refreshTokenRepository
            .findByTokenAndRevokedFalse(refreshToken)
            .orElseThrow(() -> new InvalidTokenException("Invalid refresh token"));

        CustomUserDetails userDetails = new CustomUserDetails(user);

        if (!jwtService.isTokenValid(refreshToken, userDetails)) {
            throw new InvalidTokenException("Refresh token expired");
        }

        String newAccessToken = jwtService.generateAccessToken(userDetails);

        return AuthResponse.builder()
            .accessToken(newAccessToken)
            .refreshToken(refreshToken)
            .expiresIn(jwtService.extractExpiration(newAccessToken).getTime())
            .build();
    }

    @Transactional
    public void logout(LogoutRequest request) {
        refreshTokenRepository.findByToken(request.getRefreshToken())
            .ifPresent(token -> {
                token.setRevoked(true);
                refreshTokenRepository.save(token);
            });
    }

    private void saveRefreshToken(User user, String token) {
        RefreshToken refreshToken = RefreshToken.builder()
            .user(user)
            .token(token)
            .expiresAt(jwtService.extractExpiration(token).toInstant())
            .revoked(false)
            .build();
        refreshTokenRepository.save(refreshToken);
    }

    private void revokeAllUserTokens(User user) {
        refreshTokenRepository.revokeAllByUserId(user.getId());
    }
}
```

---

## 6. Form-based Login

### Basic Form Login Configuration

```java
@Configuration
@EnableWebSecurity
public class FormLoginSecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/home", "/register", "/css/**", "/js/**").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login")
                .loginProcessingUrl("/perform_login")
                .defaultSuccessUrl("/dashboard", true)
                .failureUrl("/login?error=true")
                .usernameParameter("email")
                .passwordParameter("password")
                .permitAll()
            )
            .logout(logout -> logout
                .logoutUrl("/perform_logout")
                .logoutSuccessUrl("/login?logout=true")
                .deleteCookies("JSESSIONID")
                .invalidateHttpSession(true)
                .clearAuthentication(true)
                .permitAll()
            )
            .build();
    }
}
```

### Custom Login Success Handler

```java
@Component
public class CustomAuthenticationSuccessHandler implements AuthenticationSuccessHandler {

    private final RedirectStrategy redirectStrategy = new DefaultRedirectStrategy();

    @Override
    public void onAuthenticationSuccess(
            HttpServletRequest request,
            HttpServletResponse response,
            Authentication authentication) throws IOException {

        String targetUrl = determineTargetUrl(authentication);

        if (response.isCommitted()) {
            return;
        }

        redirectStrategy.sendRedirect(request, response, targetUrl);
    }

    private String determineTargetUrl(Authentication authentication) {
        Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();

        for (GrantedAuthority authority : authorities) {
            if (authority.getAuthority().equals("ROLE_ADMIN")) {
                return "/admin/dashboard";
            } else if (authority.getAuthority().equals("ROLE_USER")) {
                return "/user/dashboard";
            }
        }

        return "/";
    }
}
```

### Custom Login Failure Handler

```java
@Component
public class CustomAuthenticationFailureHandler implements AuthenticationFailureHandler {

    @Override
    public void onAuthenticationFailure(
            HttpServletRequest request,
            HttpServletResponse response,
            AuthenticationException exception) throws IOException, ServletException {

        String errorMessage;

        if (exception instanceof BadCredentialsException) {
            errorMessage = "Invalid username or password";
        } else if (exception instanceof DisabledException) {
            errorMessage = "Account is disabled";
        } else if (exception instanceof LockedException) {
            errorMessage = "Account is locked";
        } else if (exception instanceof AccountExpiredException) {
            errorMessage = "Account has expired";
        } else {
            errorMessage = "Authentication failed";
        }

        request.getSession().setAttribute("errorMessage", errorMessage);
        response.sendRedirect("/login?error=true");
    }
}
```

### Remember-Me Authentication

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .formLogin(form -> form.loginPage("/login").permitAll())
        .rememberMe(remember -> remember
            .key("uniqueAndSecretKey")
            .tokenValiditySeconds(86400 * 30) // 30 days
            .rememberMeParameter("remember-me")
            .userDetailsService(userDetailsService)
        )
        .build();
}
```

### Persistent Remember-Me with Database

```java
@Bean
public SecurityFilterChain securityFilterChain(
        HttpSecurity http,
        PersistentTokenRepository tokenRepository,
        UserDetailsService userDetailsService) throws Exception {
    return http
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .formLogin(form -> form.loginPage("/login").permitAll())
        .rememberMe(remember -> remember
            .tokenRepository(tokenRepository)
            .tokenValiditySeconds(86400 * 30)
            .userDetailsService(userDetailsService)
        )
        .build();
}

@Bean
public PersistentTokenRepository persistentTokenRepository(DataSource dataSource) {
    JdbcTokenRepositoryImpl tokenRepository = new JdbcTokenRepositoryImpl();
    tokenRepository.setDataSource(dataSource);
    // tokenRepository.setCreateTableOnStartup(true); // Only for initial setup
    return tokenRepository;
}
```

---

## 7. HTTP Basic Authentication

### Basic HTTP Authentication Configuration

```java
@Configuration
@EnableWebSecurity
public class BasicAuthSecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .httpBasic(basic -> basic
                .realmName("My Application")
                .authenticationEntryPoint(customEntryPoint())
            )
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .build();
    }

    @Bean
    public AuthenticationEntryPoint customEntryPoint() {
        BasicAuthenticationEntryPoint entryPoint = new BasicAuthenticationEntryPoint();
        entryPoint.setRealmName("My Application API");
        return entryPoint;
    }
}
```

### Custom Basic Auth Entry Point

```java
@Component
public class CustomBasicAuthenticationEntryPoint extends BasicAuthenticationEntryPoint {

    private final ObjectMapper objectMapper;

    public CustomBasicAuthenticationEntryPoint(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @Override
    public void commence(
            HttpServletRequest request,
            HttpServletResponse response,
            AuthenticationException authException) throws IOException {

        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.addHeader("WWW-Authenticate", "Basic realm=\"" + getRealmName() + "\"");

        ErrorResponse error = new ErrorResponse(
            HttpStatus.UNAUTHORIZED.value(),
            "Unauthorized",
            "Authentication required to access this resource"
        );

        objectMapper.writeValue(response.getOutputStream(), error);
    }

    @Override
    public void afterPropertiesSet() {
        setRealmName("My Application");
        super.afterPropertiesSet();
    }
}
```

### Combined Basic and Form Authentication

```java
@Configuration
@EnableWebSecurity
public class CombinedAuthConfig {

    @Bean
    @Order(1)
    public SecurityFilterChain apiSecurityFilterChain(HttpSecurity http) throws Exception {
        return http
            .securityMatcher("/api/**")
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
            .httpBasic(Customizer.withDefaults())
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .csrf(AbstractHttpConfigurer::disable)
            .build();
    }

    @Bean
    @Order(2)
    public SecurityFilterChain webSecurityFilterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/home").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login")
                .permitAll()
            )
            .build();
    }
}
```

---

## 8. Authorization (hasRole, hasAuthority)

### Understanding Roles and Authorities

- **Role**: A high-level permission, prefixed with `ROLE_` (e.g., `ROLE_ADMIN`)
- **Authority**: A fine-grained permission (e.g., `READ_USERS`, `WRITE_POSTS`)

### URL-Based Authorization

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
        .authorizeHttpRequests(auth -> auth
            // Public endpoints
            .requestMatchers("/", "/home", "/public/**").permitAll()

            // Role-based access
            .requestMatchers("/admin/**").hasRole("ADMIN")
            .requestMatchers("/manager/**").hasAnyRole("ADMIN", "MANAGER")

            // Authority-based access
            .requestMatchers(HttpMethod.GET, "/api/users/**").hasAuthority("READ_USERS")
            .requestMatchers(HttpMethod.POST, "/api/users/**").hasAuthority("CREATE_USERS")
            .requestMatchers(HttpMethod.PUT, "/api/users/**").hasAuthority("UPDATE_USERS")
            .requestMatchers(HttpMethod.DELETE, "/api/users/**").hasAuthority("DELETE_USERS")

            // Combined conditions
            .requestMatchers("/reports/**").hasAnyAuthority("VIEW_REPORTS", "ADMIN")

            // Custom access decisions
            .requestMatchers("/api/sensitive/**").access(
                AuthorizationManagers.allOf(
                    AuthorityAuthorizationManager.hasRole("ADMIN"),
                    new IpAddressAuthorizationManager("192.168.1.0/24")
                )
            )

            // All other requests require authentication
            .anyRequest().authenticated()
        )
        .build();
}
```

### Custom Authorization Manager

```java
public class IpAddressAuthorizationManager implements AuthorizationManager<RequestAuthorizationContext> {

    private final String allowedSubnet;

    public IpAddressAuthorizationManager(String allowedSubnet) {
        this.allowedSubnet = allowedSubnet;
    }

    @Override
    public AuthorizationDecision check(
            Supplier<Authentication> authentication,
            RequestAuthorizationContext context) {

        String clientIp = context.getRequest().getRemoteAddr();
        boolean isAllowed = isIpInSubnet(clientIp, allowedSubnet);

        return new AuthorizationDecision(isAllowed);
    }

    private boolean isIpInSubnet(String ip, String subnet) {
        // Implementation to check if IP is in subnet
        return true; // Simplified
    }
}
```

### SpEL-Based Authorization

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/**").access(new WebExpressionAuthorizationManager(
                "hasRole('ADMIN') or (hasRole('USER') and hasIpAddress('192.168.1.0/24'))"
            ))
            .anyRequest().authenticated()
        )
        .build();
}
```

### Hierarchical Roles

```java
@Bean
public RoleHierarchy roleHierarchy() {
    return RoleHierarchyImpl.withDefaultRolePrefix()
        .role("ADMIN").implies("MANAGER")
        .role("MANAGER").implies("USER")
        .role("USER").implies("GUEST")
        .build();
}

@Bean
public MethodSecurityExpressionHandler methodSecurityExpressionHandler(
        RoleHierarchy roleHierarchy) {
    DefaultMethodSecurityExpressionHandler handler = new DefaultMethodSecurityExpressionHandler();
    handler.setRoleHierarchy(roleHierarchy);
    return handler;
}
```

---

## 9. Method Security (@PreAuthorize, @PostAuthorize, @Secured)

### Enabling Method Security

```java
@Configuration
@EnableMethodSecurity(
    prePostEnabled = true,  // Enables @PreAuthorize, @PostAuthorize
    securedEnabled = true,  // Enables @Secured
    jsr250Enabled = true    // Enables @RolesAllowed
)
public class MethodSecurityConfig {
}
```

### @PreAuthorize Annotations

```java
@Service
public class UserService {

    // Simple role check
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteUser(Long userId) {
        // Only admins can delete users
    }

    // Multiple roles
    @PreAuthorize("hasAnyRole('ADMIN', 'MANAGER')")
    public void updateUser(Long userId, UserDto userDto) {
        // Admins and managers can update users
    }

    // Authority check
    @PreAuthorize("hasAuthority('DELETE_USERS')")
    public void permanentlyDeleteUser(Long userId) {
        // Requires specific authority
    }

    // SpEL with method parameters
    @PreAuthorize("#userId == authentication.principal.id or hasRole('ADMIN')")
    public UserDto getUserProfile(Long userId) {
        // User can view their own profile, or admin can view any
    }

    // Complex SpEL expression
    @PreAuthorize("@securityService.canAccessUser(#userId)")
    public UserDto getUser(Long userId) {
        // Delegates to a custom security service
    }

    // Check object properties
    @PreAuthorize("#user.department == authentication.principal.department or hasRole('ADMIN')")
    public void updateUserProfile(User user) {
        // User must be in same department or be admin
    }
}
```

### @PostAuthorize Annotations

```java
@Service
public class DocumentService {

    // Filter result after method execution
    @PostAuthorize("returnObject.owner == authentication.principal.username or hasRole('ADMIN')")
    public Document getDocument(Long documentId) {
        return documentRepository.findById(documentId)
            .orElseThrow(() -> new DocumentNotFoundException(documentId));
    }

    // Check collection elements
    @PostAuthorize("returnObject.?[owner != authentication.principal.username].empty or hasRole('ADMIN')")
    public List<Document> getDocumentsByProject(Long projectId) {
        return documentRepository.findByProjectId(projectId);
    }
}
```

### @PreFilter and @PostFilter

```java
@Service
public class DataService {

    // Filter input collection
    @PreFilter("filterObject.department == authentication.principal.department")
    public void processUsers(List<User> users) {
        // Only processes users from same department
        users.forEach(this::process);
    }

    // Filter output collection
    @PostFilter("filterObject.publiclyVisible or filterObject.owner == authentication.principal.username")
    public List<Document> getAllDocuments() {
        return documentRepository.findAll();
    }

    // Filter with target collection specified
    @PreFilter(value = "filterObject.status == 'ACTIVE'", filterTarget = "users")
    public void deactivateUsers(List<User> users, String reason) {
        // Only active users are passed
    }
}
```

### @Secured Annotation

```java
@Service
public class ReportService {

    @Secured("ROLE_ADMIN")
    public Report generateAdminReport() {
        // Only ROLE_ADMIN can access
    }

    @Secured({"ROLE_ADMIN", "ROLE_MANAGER"})
    public Report generateManagementReport() {
        // ROLE_ADMIN or ROLE_MANAGER can access
    }
}
```

### @RolesAllowed (JSR-250)

```java
@Service
public class OrderService {

    @RolesAllowed("ADMIN")
    public void cancelOrder(Long orderId) {
        // Only ADMIN role
    }

    @RolesAllowed({"ADMIN", "CUSTOMER_SERVICE"})
    public void refundOrder(Long orderId) {
        // ADMIN or CUSTOMER_SERVICE role
    }

    @PermitAll
    public Order getOrderDetails(Long orderId) {
        // Anyone can access
    }

    @DenyAll
    public void deprecatedMethod() {
        // No one can access
    }
}
```

### Custom Security Expression Handler

```java
@Component("securityService")
public class SecurityService {

    private final UserRepository userRepository;

    public SecurityService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public boolean canAccessUser(Long userId) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth == null) return false;

        CustomUserDetails principal = (CustomUserDetails) auth.getPrincipal();

        // Admin can access anyone
        if (principal.getAuthorities().stream()
                .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"))) {
            return true;
        }

        // Users can access themselves
        return principal.getId().equals(userId);
    }

    public boolean isOwner(Object entity) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth == null) return false;

        String currentUser = auth.getName();

        if (entity instanceof OwnedEntity owned) {
            return currentUser.equals(owned.getOwner());
        }

        return false;
    }

    public boolean belongsToSameOrganization(Long targetUserId) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        CustomUserDetails principal = (CustomUserDetails) auth.getPrincipal();

        User targetUser = userRepository.findById(targetUserId).orElse(null);
        if (targetUser == null) return false;

        return principal.getOrganizationId().equals(targetUser.getOrganizationId());
    }
}
```

---

## 10. CORS Configuration

### Basic CORS Configuration

```java
@Configuration
public class CorsConfig {

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();

        // Allowed origins
        configuration.setAllowedOrigins(List.of(
            "http://localhost:3000",
            "http://localhost:5173",
            "https://myapp.com"
        ));

        // Allowed HTTP methods
        configuration.setAllowedMethods(List.of(
            "GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"
        ));

        // Allowed headers
        configuration.setAllowedHeaders(List.of(
            "Authorization",
            "Content-Type",
            "X-Requested-With",
            "Accept",
            "Origin",
            "Access-Control-Request-Method",
            "Access-Control-Request-Headers"
        ));

        // Exposed headers (accessible to JavaScript)
        configuration.setExposedHeaders(List.of(
            "Access-Control-Allow-Origin",
            "Access-Control-Allow-Credentials",
            "X-Custom-Header"
        ));

        // Allow credentials (cookies, authorization headers)
        configuration.setAllowCredentials(true);

        // Preflight cache duration
        configuration.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration);
        return source;
    }
}
```

### CORS with Security Filter Chain

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
        .cors(cors -> cors.configurationSource(corsConfigurationSource()))
        .csrf(AbstractHttpConfigurer::disable)
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .build();
}
```

### Per-Endpoint CORS with @CrossOrigin

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {

    // Controller-level CORS
    @CrossOrigin(origins = "http://localhost:3000")
    @GetMapping
    public List<Product> getAllProducts() {
        return productService.findAll();
    }

    // Method-level CORS with more options
    @CrossOrigin(
        origins = {"http://localhost:3000", "https://myapp.com"},
        methods = {RequestMethod.GET, RequestMethod.POST},
        allowedHeaders = {"Authorization", "Content-Type"},
        exposedHeaders = {"X-Custom-Header"},
        allowCredentials = "true",
        maxAge = 3600
    )
    @PostMapping
    public Product createProduct(@RequestBody Product product) {
        return productService.create(product);
    }
}
```

### Dynamic CORS Configuration

```java
@Configuration
public class DynamicCorsConfig {

    @Value("${cors.allowed-origins}")
    private List<String> allowedOrigins;

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOriginPatterns(List.of(
            "https://*.myapp.com",
            "http://localhost:*"
        ));
        configuration.setAllowedMethods(List.of("*"));
        configuration.setAllowedHeaders(List.of("*"));
        configuration.setAllowCredentials(true);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", configuration);
        return source;
    }
}
```

### CORS Configuration for WebMvc

```java
@Configuration
public class WebMvcCorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("http://localhost:3000")
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowedHeaders("*")
            .allowCredentials(true)
            .maxAge(3600);

        registry.addMapping("/public/**")
            .allowedOrigins("*")
            .allowedMethods("GET")
            .maxAge(3600);
    }
}
```

---

## 11. CSRF Protection

### Default CSRF Protection

CSRF protection is enabled by default in Spring Security:

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
        .csrf(Customizer.withDefaults())  // CSRF enabled by default
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .formLogin(Customizer.withDefaults())
        .build();
}
```

### Custom CSRF Token Repository

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
        .csrf(csrf -> csrf
            .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
            .csrfTokenRequestHandler(new SpaCsrfTokenRequestHandler())
        )
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .build();
}
```

### SPA CSRF Token Handler

```java
public class SpaCsrfTokenRequestHandler extends CsrfTokenRequestAttributeHandler {

    private final CsrfTokenRequestHandler delegate = new XorCsrfTokenRequestAttributeHandler();

    @Override
    public void handle(
            HttpServletRequest request,
            HttpServletResponse response,
            Supplier<CsrfToken> csrfToken) {

        // Always use XorCsrfTokenRequestAttributeHandler to provide BREACH protection
        this.delegate.handle(request, response, csrfToken);
    }

    @Override
    public String resolveCsrfTokenValue(HttpServletRequest request, CsrfToken csrfToken) {
        // Check header first (for SPA), then fall back to parameter
        String headerValue = request.getHeader(csrfToken.getHeaderName());
        if (StringUtils.hasText(headerValue)) {
            return super.resolveCsrfTokenValue(request, csrfToken);
        }
        return this.delegate.resolveCsrfTokenValue(request, csrfToken);
    }
}
```

### CSRF with Cookie Repository for SPA

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    CookieCsrfTokenRepository tokenRepository = CookieCsrfTokenRepository.withHttpOnlyFalse();
    tokenRepository.setCookiePath("/");
    tokenRepository.setCookieName("XSRF-TOKEN");
    tokenRepository.setHeaderName("X-XSRF-TOKEN");

    return http
        .csrf(csrf -> csrf
            .csrfTokenRepository(tokenRepository)
            .csrfTokenRequestHandler(new SpaCsrfTokenRequestHandler())
        )
        .addFilterAfter(new CsrfCookieFilter(), BasicAuthenticationFilter.class)
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .build();
}
```

### CSRF Cookie Filter for SPA

```java
public class CsrfCookieFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        CsrfToken csrfToken = (CsrfToken) request.getAttribute("_csrf");

        // Render the token to ensure cookie is set
        if (csrfToken != null) {
            csrfToken.getToken();
        }

        filterChain.doFilter(request, response);
    }
}
```

### Ignoring CSRF for Specific Endpoints

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
        .csrf(csrf -> csrf
            .ignoringRequestMatchers(
                "/api/webhooks/**",
                "/api/public/**"
            )
            .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
        )
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .build();
}
```

### Disabling CSRF for Stateless APIs

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
        .csrf(AbstractHttpConfigurer::disable)  // Disable for stateless JWT APIs
        .sessionManagement(session ->
            session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .build();
}
```

---

## 12. Session Management

### Session Creation Policy

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
        .sessionManagement(session -> session
            // STATELESS - No session created (for REST APIs with JWT)
            // .sessionCreationPolicy(SessionCreationPolicy.STATELESS)

            // ALWAYS - Always create session
            // .sessionCreationPolicy(SessionCreationPolicy.ALWAYS)

            // IF_REQUIRED - Create session only when required (default)
            .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)

            // NEVER - Never create session, but use if exists
            // .sessionCreationPolicy(SessionCreationPolicy.NEVER)
        )
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .build();
}
```

### Session Fixation Protection

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
        .sessionManagement(session -> session
            // Migrate session - creates new session, copies attributes (default)
            .sessionFixation().migrateSession()

            // New session - creates new session, doesn't copy attributes
            // .sessionFixation().newSession()

            // Change session ID - changes session ID without creating new session
            // .sessionFixation().changeSessionId()

            // None - no protection (not recommended)
            // .sessionFixation().none()
        )
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .build();
}
```

### Concurrent Session Control

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
        .sessionManagement(session -> session
            .maximumSessions(1)  // Only one session per user
            .maxSessionsPreventsLogin(false)  // New login expires old session
            // .maxSessionsPreventsLogin(true)  // Prevent new login if max reached
            .expiredUrl("/session-expired")
            .sessionRegistry(sessionRegistry())
        )
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .build();
}

@Bean
public SessionRegistry sessionRegistry() {
    return new SessionRegistryImpl();
}

@Bean
public HttpSessionEventPublisher httpSessionEventPublisher() {
    return new HttpSessionEventPublisher();
}
```

### Session Timeout Configuration

```yaml
# application.yml
server:
  servlet:
    session:
      timeout: 30m  # 30 minutes
      cookie:
        name: JSESSIONID
        http-only: true
        secure: true  # Only send over HTTPS
        same-site: strict
```

### Custom Session Expired Strategy

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
        .sessionManagement(session -> session
            .maximumSessions(1)
            .expiredSessionStrategy(event -> {
                HttpServletResponse response = event.getResponse();
                response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                response.setContentType("application/json");
                response.getWriter().write(
                    "{\"error\": \"Session expired\", \"message\": \"Your session has expired\"}");
            })
        )
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .build();
}
```

### Invalid Session Strategy

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
        .sessionManagement(session -> session
            .invalidSessionUrl("/invalid-session")
            .invalidSessionStrategy((request, response) -> {
                response.sendRedirect("/login?session=invalid");
            })
        )
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .build();
}
```

### Programmatic Session Management

```java
@Service
@RequiredArgsConstructor
public class SessionService {

    private final SessionRegistry sessionRegistry;

    public List<String> getActiveUsers() {
        return sessionRegistry.getAllPrincipals().stream()
            .map(principal -> ((UserDetails) principal).getUsername())
            .collect(Collectors.toList());
    }

    public List<SessionInformation> getUserSessions(String username) {
        return sessionRegistry.getAllPrincipals().stream()
            .filter(p -> ((UserDetails) p).getUsername().equals(username))
            .flatMap(p -> sessionRegistry.getAllSessions(p, false).stream())
            .collect(Collectors.toList());
    }

    public void expireUserSessions(String username) {
        sessionRegistry.getAllPrincipals().stream()
            .filter(p -> ((UserDetails) p).getUsername().equals(username))
            .flatMap(p -> sessionRegistry.getAllSessions(p, false).stream())
            .forEach(SessionInformation::expireNow);
    }

    public void expireSession(String sessionId) {
        SessionInformation sessionInfo = sessionRegistry.getSessionInformation(sessionId);
        if (sessionInfo != null) {
            sessionInfo.expireNow();
        }
    }
}
```

---

## 13. OAuth2 Resource Server

### Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

### JWT Resource Server Configuration

```yaml
# application.yml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com
          # Or use JWK set URI directly
          # jwk-set-uri: https://auth.example.com/.well-known/jwks.json
```

### Security Configuration for Resource Server

```java
@Configuration
@EnableWebSecurity
public class ResourceServerConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasAuthority("SCOPE_admin")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            )
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .build();
    }

    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter grantedAuthoritiesConverter =
            new JwtGrantedAuthoritiesConverter();
        grantedAuthoritiesConverter.setAuthoritiesClaimName("roles");
        grantedAuthoritiesConverter.setAuthorityPrefix("ROLE_");

        JwtAuthenticationConverter jwtAuthenticationConverter =
            new JwtAuthenticationConverter();
        jwtAuthenticationConverter.setJwtGrantedAuthoritiesConverter(
            grantedAuthoritiesConverter);

        return jwtAuthenticationConverter;
    }
}
```

### Custom JWT Decoder

```java
@Bean
public JwtDecoder jwtDecoder() {
    NimbusJwtDecoder jwtDecoder = NimbusJwtDecoder
        .withJwkSetUri("https://auth.example.com/.well-known/jwks.json")
        .build();

    OAuth2TokenValidator<Jwt> withIssuer = JwtValidators
        .createDefaultWithIssuer("https://auth.example.com");

    OAuth2TokenValidator<Jwt> withAudience = new JwtClaimValidator<List<String>>(
        "aud",
        aud -> aud != null && aud.contains("my-api")
    );

    OAuth2TokenValidator<Jwt> validator = new DelegatingOAuth2TokenValidator<>(
        withIssuer, withAudience);

    jwtDecoder.setJwtValidator(validator);
    return jwtDecoder;
}
```

### Custom JWT Authentication Converter

```java
@Component
public class CustomJwtAuthenticationConverter implements Converter<Jwt, AbstractAuthenticationToken> {

    @Override
    public AbstractAuthenticationToken convert(Jwt jwt) {
        Collection<GrantedAuthority> authorities = extractAuthorities(jwt);

        String principalName = jwt.getClaimAsString("preferred_username");
        if (principalName == null) {
            principalName = jwt.getSubject();
        }

        return new JwtAuthenticationToken(jwt, authorities, principalName);
    }

    private Collection<GrantedAuthority> extractAuthorities(Jwt jwt) {
        List<GrantedAuthority> authorities = new ArrayList<>();

        // Extract realm roles
        Map<String, Object> realmAccess = jwt.getClaim("realm_access");
        if (realmAccess != null) {
            List<String> roles = (List<String>) realmAccess.get("roles");
            if (roles != null) {
                roles.forEach(role ->
                    authorities.add(new SimpleGrantedAuthority("ROLE_" + role.toUpperCase()))
                );
            }
        }

        // Extract resource roles
        Map<String, Object> resourceAccess = jwt.getClaim("resource_access");
        if (resourceAccess != null) {
            Map<String, Object> myApi = (Map<String, Object>) resourceAccess.get("my-api");
            if (myApi != null) {
                List<String> roles = (List<String>) myApi.get("roles");
                if (roles != null) {
                    roles.forEach(role ->
                        authorities.add(new SimpleGrantedAuthority("ROLE_" + role.toUpperCase()))
                    );
                }
            }
        }

        // Extract scopes
        String scopeClaim = jwt.getClaimAsString("scope");
        if (scopeClaim != null) {
            Arrays.stream(scopeClaim.split(" "))
                .map(scope -> new SimpleGrantedAuthority("SCOPE_" + scope))
                .forEach(authorities::add);
        }

        return authorities;
    }
}
```

### Opaque Token Resource Server

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .oauth2ResourceServer(oauth2 -> oauth2
            .opaqueToken(opaque -> opaque
                .introspectionUri("https://auth.example.com/introspect")
                .introspectionClientCredentials("client-id", "client-secret")
            )
        )
        .build();
}
```

### Accessing JWT Claims

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/me")
    public Map<String, Object> getCurrentUser(@AuthenticationPrincipal Jwt jwt) {
        return Map.of(
            "subject", jwt.getSubject(),
            "email", jwt.getClaimAsString("email"),
            "name", jwt.getClaimAsString("name"),
            "roles", jwt.getClaimAsStringList("roles")
        );
    }

    @GetMapping("/profile")
    public ResponseEntity<UserProfile> getProfile(
            @AuthenticationPrincipal JwtAuthenticationToken authentication) {
        Jwt jwt = authentication.getToken();
        String userId = jwt.getSubject();

        // Fetch user profile using the subject
        return ResponseEntity.ok(userService.getProfile(userId));
    }
}
```

---

## 14. OAuth2 Client

### Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

### OAuth2 Client Configuration

```yaml
# application.yml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope:
              - email
              - profile
          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
            scope:
              - user:email
              - read:user
          custom-provider:
            client-id: ${CUSTOM_CLIENT_ID}
            client-secret: ${CUSTOM_CLIENT_SECRET}
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
            scope:
              - openid
              - profile
              - email
        provider:
          custom-provider:
            authorization-uri: https://auth.example.com/authorize
            token-uri: https://auth.example.com/oauth/token
            user-info-uri: https://auth.example.com/userinfo
            user-name-attribute: sub
            jwk-set-uri: https://auth.example.com/.well-known/jwks.json
```

### OAuth2 Login Security Configuration

```java
@Configuration
@EnableWebSecurity
public class OAuth2LoginSecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/error", "/login/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard")
                .failureUrl("/login?error=true")
                .userInfoEndpoint(userInfo -> userInfo
                    .userService(customOAuth2UserService())
                    .oidcUserService(customOidcUserService())
                )
                .authorizationEndpoint(auth -> auth
                    .authorizationRequestRepository(
                        new HttpSessionOAuth2AuthorizationRequestRepository())
                )
            )
            .build();
    }

    @Bean
    public OAuth2UserService<OAuth2UserRequest, OAuth2User> customOAuth2UserService() {
        return new CustomOAuth2UserService();
    }

    @Bean
    public OAuth2UserService<OidcUserRequest, OidcUser> customOidcUserService() {
        return new CustomOidcUserService();
    }
}
```

### Custom OAuth2 User Service

```java
@Service
@RequiredArgsConstructor
public class CustomOAuth2UserService extends DefaultOAuth2UserService {

    private final UserRepository userRepository;

    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
        OAuth2User oauth2User = super.loadUser(userRequest);

        String registrationId = userRequest.getClientRegistration().getRegistrationId();
        String userNameAttribute = userRequest.getClientRegistration()
            .getProviderDetails().getUserInfoEndpoint().getUserNameAttributeName();

        OAuthUserInfo userInfo = OAuthUserInfoFactory.getOAuthUserInfo(
            registrationId, oauth2User.getAttributes());

        User user = processOAuthUser(userInfo, registrationId);

        Set<GrantedAuthority> authorities = user.getRoles().stream()
            .map(role -> new SimpleGrantedAuthority("ROLE_" + role.getName()))
            .collect(Collectors.toSet());

        return new CustomOAuth2User(
            authorities,
            oauth2User.getAttributes(),
            userNameAttribute,
            user
        );
    }

    @Transactional
    protected User processOAuthUser(OAuthUserInfo userInfo, String provider) {
        Optional<User> existingUser = userRepository.findByEmail(userInfo.getEmail());

        if (existingUser.isPresent()) {
            User user = existingUser.get();
            if (!user.getProvider().equals(provider)) {
                throw new OAuth2AuthenticationException(
                    "Account already exists with different provider");
            }
            return updateExistingUser(user, userInfo);
        }

        return createNewUser(userInfo, provider);
    }

    private User createNewUser(OAuthUserInfo userInfo, String provider) {
        User user = User.builder()
            .email(userInfo.getEmail())
            .firstName(userInfo.getFirstName())
            .lastName(userInfo.getLastName())
            .imageUrl(userInfo.getImageUrl())
            .provider(provider)
            .providerId(userInfo.getId())
            .enabled(true)
            .build();

        return userRepository.save(user);
    }

    private User updateExistingUser(User user, OAuthUserInfo userInfo) {
        user.setFirstName(userInfo.getFirstName());
        user.setLastName(userInfo.getLastName());
        user.setImageUrl(userInfo.getImageUrl());
        return userRepository.save(user);
    }
}
```

### OAuth2 Success Handler

```java
@Component
@RequiredArgsConstructor
public class OAuth2AuthenticationSuccessHandler
        extends SimpleUrlAuthenticationSuccessHandler {

    private final JwtService jwtService;
    private final UserRepository userRepository;

    @Override
    public void onAuthenticationSuccess(
            HttpServletRequest request,
            HttpServletResponse response,
            Authentication authentication) throws IOException {

        OAuth2User oauth2User = (OAuth2User) authentication.getPrincipal();
        String email = oauth2User.getAttribute("email");

        User user = userRepository.findByEmail(email)
            .orElseThrow(() -> new UsernameNotFoundException("User not found"));

        CustomUserDetails userDetails = new CustomUserDetails(user);
        String accessToken = jwtService.generateAccessToken(userDetails);
        String refreshToken = jwtService.generateRefreshToken(userDetails);

        // Redirect to frontend with tokens
        String targetUrl = UriComponentsBuilder.fromUriString("/oauth2/redirect")
            .queryParam("access_token", accessToken)
            .queryParam("refresh_token", refreshToken)
            .build().toUriString();

        getRedirectStrategy().sendRedirect(request, response, targetUrl);
    }
}
```

### Client Credentials Grant for Service-to-Service

```java
@Configuration
public class OAuth2ClientConfig {

    @Bean
    public OAuth2AuthorizedClientManager authorizedClientManager(
            ClientRegistrationRepository clientRegistrationRepository,
            OAuth2AuthorizedClientRepository authorizedClientRepository) {

        OAuth2AuthorizedClientProvider authorizedClientProvider =
            OAuth2AuthorizedClientProviderBuilder.builder()
                .clientCredentials()
                .refreshToken()
                .build();

        DefaultOAuth2AuthorizedClientManager authorizedClientManager =
            new DefaultOAuth2AuthorizedClientManager(
                clientRegistrationRepository, authorizedClientRepository);

        authorizedClientManager.setAuthorizedClientProvider(authorizedClientProvider);

        return authorizedClientManager;
    }
}
```

### WebClient with OAuth2

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient webClient(OAuth2AuthorizedClientManager authorizedClientManager) {
        ServletOAuth2AuthorizedClientExchangeFilterFunction oauth2Client =
            new ServletOAuth2AuthorizedClientExchangeFilterFunction(authorizedClientManager);

        oauth2Client.setDefaultClientRegistrationId("custom-provider");

        return WebClient.builder()
            .apply(oauth2Client.oauth2Configuration())
            .build();
    }
}

@Service
@RequiredArgsConstructor
public class ExternalApiService {

    private final WebClient webClient;

    public Mono<ApiResponse> callExternalApi() {
        return webClient.get()
            .uri("https://api.example.com/data")
            .attributes(ServletOAuth2AuthorizedClientExchangeFilterFunction
                .clientRegistrationId("custom-provider"))
            .retrieve()
            .bodyToMono(ApiResponse.class);
    }
}
```

---

## 15. Security Filters

### Filter Chain Architecture

Spring Security uses a chain of filters. Understanding the order is crucial:

```
SecurityContextPersistenceFilter
CsrfFilter
LogoutFilter
UsernamePasswordAuthenticationFilter
DefaultLoginPageGeneratingFilter
BasicAuthenticationFilter
RequestCacheAwareFilter
SecurityContextHolderAwareRequestFilter
AnonymousAuthenticationFilter
SessionManagementFilter
ExceptionTranslationFilter
FilterSecurityInterceptor (AuthorizationFilter in Spring Security 6)
```

### Custom Security Filter

```java
@Component
public class RequestLoggingFilter extends OncePerRequestFilter {

    private static final Logger log = LoggerFactory.getLogger(RequestLoggingFilter.class);

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        long startTime = System.currentTimeMillis();
        String requestId = UUID.randomUUID().toString();

        // Add request ID to MDC for logging
        MDC.put("requestId", requestId);
        response.setHeader("X-Request-Id", requestId);

        try {
            log.info("Incoming request: {} {} from {}",
                request.getMethod(),
                request.getRequestURI(),
                request.getRemoteAddr());

            filterChain.doFilter(request, response);

        } finally {
            long duration = System.currentTimeMillis() - startTime;
            log.info("Request completed: {} {} - Status: {} - Duration: {}ms",
                request.getMethod(),
                request.getRequestURI(),
                response.getStatus(),
                duration);
            MDC.clear();
        }
    }
}
```

### Adding Custom Filter to Chain

```java
@Bean
public SecurityFilterChain securityFilterChain(
        HttpSecurity http,
        RequestLoggingFilter loggingFilter,
        JwtAuthenticationFilter jwtFilter) throws Exception {
    return http
        .addFilterBefore(loggingFilter, SecurityContextHolderFilter.class)
        .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .build();
}
```

### Rate Limiting Filter

```java
@Component
public class RateLimitingFilter extends OncePerRequestFilter {

    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();
    private final int requestsPerMinute = 60;

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        String clientIp = getClientIp(request);
        Bucket bucket = buckets.computeIfAbsent(clientIp, this::createBucket);

        if (bucket.tryConsume(1)) {
            filterChain.doFilter(request, response);
        } else {
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.setContentType("application/json");
            response.getWriter().write(
                "{\"error\": \"Rate limit exceeded\", \"retryAfter\": 60}");
        }
    }

    private Bucket createBucket(String key) {
        Bandwidth limit = Bandwidth.classic(requestsPerMinute,
            Refill.greedy(requestsPerMinute, Duration.ofMinutes(1)));
        return Bucket.builder().addLimit(limit).build();
    }

    private String getClientIp(HttpServletRequest request) {
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        if (xForwardedFor != null && !xForwardedFor.isEmpty()) {
            return xForwardedFor.split(",")[0].trim();
        }
        return request.getRemoteAddr();
    }
}
```

### API Key Authentication Filter

```java
@Component
public class ApiKeyAuthenticationFilter extends OncePerRequestFilter {

    private static final String API_KEY_HEADER = "X-API-Key";
    private final ApiKeyService apiKeyService;

    public ApiKeyAuthenticationFilter(ApiKeyService apiKeyService) {
        this.apiKeyService = apiKeyService;
    }

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        String apiKey = request.getHeader(API_KEY_HEADER);

        if (apiKey != null) {
            Optional<ApiKeyDetails> keyDetails = apiKeyService.validateApiKey(apiKey);

            if (keyDetails.isPresent()) {
                ApiKeyDetails details = keyDetails.get();
                ApiKeyAuthenticationToken authentication = new ApiKeyAuthenticationToken(
                    details,
                    details.getAuthorities()
                );
                authentication.setDetails(
                    new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        }

        filterChain.doFilter(request, response);
    }

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        return !request.getRequestURI().startsWith("/api/");
    }
}
```

### Audit Logging Filter

```java
@Component
@RequiredArgsConstructor
public class AuditLoggingFilter extends OncePerRequestFilter {

    private final AuditService auditService;

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        ContentCachingRequestWrapper wrappedRequest =
            new ContentCachingRequestWrapper(request);
        ContentCachingResponseWrapper wrappedResponse =
            new ContentCachingResponseWrapper(response);

        try {
            filterChain.doFilter(wrappedRequest, wrappedResponse);
        } finally {
            AuditEvent event = buildAuditEvent(wrappedRequest, wrappedResponse);
            auditService.log(event);
            wrappedResponse.copyBodyToResponse();
        }
    }

    private AuditEvent buildAuditEvent(
            ContentCachingRequestWrapper request,
            ContentCachingResponseWrapper response) {

        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        String username = auth != null ? auth.getName() : "anonymous";

        return AuditEvent.builder()
            .timestamp(Instant.now())
            .username(username)
            .method(request.getMethod())
            .uri(request.getRequestURI())
            .queryString(request.getQueryString())
            .remoteAddress(request.getRemoteAddr())
            .statusCode(response.getStatus())
            .build();
    }

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        String path = request.getRequestURI();
        return path.startsWith("/actuator/") ||
               path.startsWith("/swagger-ui/") ||
               path.startsWith("/static/");
    }
}
```

---

## 16. Testing Security (@WithMockUser)

### Dependencies

```xml
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-test</artifactId>
    <scope>test</scope>
</dependency>
```

### @WithMockUser Annotation

```java
@SpringBootTest
@AutoConfigureMockMvc
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    @WithMockUser(username = "user", roles = {"USER"})
    void getProfile_WithUser_ShouldSucceed() throws Exception {
        mockMvc.perform(get("/api/users/profile"))
            .andExpect(status().isOk());
    }

    @Test
    @WithMockUser(username = "admin", roles = {"ADMIN"})
    void deleteUser_WithAdmin_ShouldSucceed() throws Exception {
        mockMvc.perform(delete("/api/users/123"))
            .andExpect(status().isNoContent());
    }

    @Test
    @WithMockUser(username = "user", roles = {"USER"})
    void deleteUser_WithUser_ShouldBeForbidden() throws Exception {
        mockMvc.perform(delete("/api/users/123"))
            .andExpect(status().isForbidden());
    }

    @Test
    void getProfile_WithoutAuth_ShouldBeUnauthorized() throws Exception {
        mockMvc.perform(get("/api/users/profile"))
            .andExpect(status().isUnauthorized());
    }
}
```

### @WithMockUser with Authorities

```java
@Test
@WithMockUser(username = "user", authorities = {"READ_USERS", "WRITE_USERS"})
void updateUser_WithAuthorities_ShouldSucceed() throws Exception {
    mockMvc.perform(put("/api/users/123")
            .contentType(MediaType.APPLICATION_JSON)
            .content("{\"name\": \"Updated Name\"}"))
        .andExpect(status().isOk());
}
```

### @WithUserDetails

```java
@Test
@WithUserDetails(value = "admin@example.com", userDetailsServiceBeanName = "customUserDetailsService")
void adminEndpoint_WithUserDetails_ShouldSucceed() throws Exception {
    mockMvc.perform(get("/api/admin/dashboard"))
        .andExpect(status().isOk());
}
```

### Custom Security Context with @WithSecurityContext

```java
@Retention(RetentionPolicy.RUNTIME)
@WithSecurityContext(factory = WithCustomUserSecurityContextFactory.class)
public @interface WithCustomUser {
    String email() default "test@example.com";
    String[] roles() default {"USER"};
    long id() default 1L;
}

public class WithCustomUserSecurityContextFactory
        implements WithSecurityContextFactory<WithCustomUser> {

    @Override
    public SecurityContext createSecurityContext(WithCustomUser annotation) {
        SecurityContext context = SecurityContextHolder.createEmptyContext();

        User user = User.builder()
            .id(annotation.id())
            .email(annotation.email())
            .firstName("Test")
            .lastName("User")
            .build();

        CustomUserDetails principal = new CustomUserDetails(user);

        List<GrantedAuthority> authorities = Arrays.stream(annotation.roles())
            .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
            .collect(Collectors.toList());

        Authentication auth = new UsernamePasswordAuthenticationToken(
            principal, null, authorities);

        context.setAuthentication(auth);
        return context;
    }
}

// Usage
@Test
@WithCustomUser(email = "john@example.com", roles = {"ADMIN", "USER"}, id = 42L)
void customUserTest() throws Exception {
    mockMvc.perform(get("/api/users/me"))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.email").value("john@example.com"));
}
```

### Testing with JWT

```java
@SpringBootTest
@AutoConfigureMockMvc
class JwtSecurityTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private JwtService jwtService;

    @Test
    void protectedEndpoint_WithValidJwt_ShouldSucceed() throws Exception {
        UserDetails userDetails = User.builder()
            .username("test@example.com")
            .password("password")
            .roles("USER")
            .build();

        String token = jwtService.generateAccessToken(userDetails);

        mockMvc.perform(get("/api/protected")
                .header("Authorization", "Bearer " + token))
            .andExpect(status().isOk());
    }

    @Test
    void protectedEndpoint_WithExpiredJwt_ShouldBeUnauthorized() throws Exception {
        // Create an expired token for testing
        String expiredToken = createExpiredToken();

        mockMvc.perform(get("/api/protected")
                .header("Authorization", "Bearer " + expiredToken))
            .andExpect(status().isUnauthorized());
    }

    @Test
    void protectedEndpoint_WithInvalidJwt_ShouldBeUnauthorized() throws Exception {
        mockMvc.perform(get("/api/protected")
                .header("Authorization", "Bearer invalid-token"))
            .andExpect(status().isUnauthorized());
    }
}
```

### Testing OAuth2 Resource Server

```java
@SpringBootTest
@AutoConfigureMockMvc
class OAuth2ResourceServerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void protectedEndpoint_WithValidJwt_ShouldSucceed() throws Exception {
        mockMvc.perform(get("/api/resource")
                .with(jwt().authorities(new SimpleGrantedAuthority("SCOPE_read"))))
            .andExpect(status().isOk());
    }

    @Test
    void adminEndpoint_WithAdminRole_ShouldSucceed() throws Exception {
        mockMvc.perform(get("/api/admin")
                .with(jwt()
                    .jwt(jwt -> jwt
                        .claim("sub", "admin@example.com")
                        .claim("roles", List.of("ADMIN")))
                    .authorities(new SimpleGrantedAuthority("ROLE_ADMIN"))))
            .andExpect(status().isOk());
    }
}
```

### Testing CSRF

```java
@Test
void postRequest_WithCsrf_ShouldSucceed() throws Exception {
    mockMvc.perform(post("/api/data")
            .with(csrf())
            .with(user("user").roles("USER"))
            .contentType(MediaType.APPLICATION_JSON)
            .content("{\"value\": \"test\"}"))
        .andExpect(status().isOk());
}

@Test
void postRequest_WithoutCsrf_ShouldBeForbidden() throws Exception {
    mockMvc.perform(post("/api/data")
            .with(user("user").roles("USER"))
            .contentType(MediaType.APPLICATION_JSON)
            .content("{\"value\": \"test\"}"))
        .andExpect(status().isForbidden());
}
```

### Testing Method Security

```java
@SpringBootTest
class MethodSecurityTest {

    @Autowired
    private UserService userService;

    @Test
    @WithMockUser(roles = "ADMIN")
    void deleteUser_AsAdmin_ShouldSucceed() {
        assertDoesNotThrow(() -> userService.deleteUser(1L));
    }

    @Test
    @WithMockUser(roles = "USER")
    void deleteUser_AsUser_ShouldThrowAccessDenied() {
        assertThrows(AccessDeniedException.class,
            () -> userService.deleteUser(1L));
    }
}
```

---

## 17. Best Practices

### 1. Secure Password Storage

```java
// Always use BCrypt or Argon2 for password hashing
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12);
}

// Never store plain text passwords
// Never use MD5 or SHA-1 for passwords
// Never implement your own password hashing
```

### 2. JWT Security Best Practices

```java
// Use strong secret keys (256+ bits)
@Value("${jwt.secret}")
private String secretKey; // Must be at least 32 characters for HS256

// Set reasonable expiration times
private static final long ACCESS_TOKEN_EXPIRATION = 15 * 60 * 1000; // 15 minutes
private static final long REFRESH_TOKEN_EXPIRATION = 7 * 24 * 60 * 60 * 1000; // 7 days

// Store refresh tokens securely and implement token rotation
// Implement token blacklisting for logout
// Use HTTPS only for token transmission
```

### 3. Input Validation

```java
@RestController
@Validated
public class UserController {

    @PostMapping("/users")
    public ResponseEntity<User> createUser(
            @Valid @RequestBody CreateUserRequest request) {
        // Input is validated before processing
        return ResponseEntity.ok(userService.create(request));
    }
}

public class CreateUserRequest {
    @NotBlank
    @Email
    private String email;

    @NotBlank
    @Size(min = 8, max = 100)
    @Pattern(regexp = "^(?=.*[0-9])(?=.*[a-z])(?=.*[A-Z])(?=.*[@#$%^&+=]).*$")
    private String password;
}
```

### 4. Secure Headers Configuration

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
        .headers(headers -> headers
            .contentTypeOptions(Customizer.withDefaults())
            .xssProtection(xss -> xss.headerValue(XXssProtectionHeaderWriter.HeaderValue.ENABLED_MODE_BLOCK))
            .cacheControl(Customizer.withDefaults())
            .httpStrictTransportSecurity(hsts -> hsts
                .includeSubDomains(true)
                .maxAgeInSeconds(31536000))
            .frameOptions(frame -> frame.deny())
            .contentSecurityPolicy(csp -> csp
                .policyDirectives("default-src 'self'; script-src 'self'; object-src 'none'"))
        )
        .build();
}
```

### 5. Exception Handling

```java
@RestControllerAdvice
public class SecurityExceptionHandler {

    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ErrorResponse> handleAccessDenied(AccessDeniedException ex) {
        return ResponseEntity.status(HttpStatus.FORBIDDEN)
            .body(new ErrorResponse("ACCESS_DENIED", "You don't have permission to access this resource"));
    }

    @ExceptionHandler(AuthenticationException.class)
    public ResponseEntity<ErrorResponse> handleAuthentication(AuthenticationException ex) {
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
            .body(new ErrorResponse("AUTHENTICATION_FAILED", "Authentication failed"));
    }

    // Don't expose internal details in error messages
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        log.error("Unexpected error", ex);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred"));
    }
}
```

### 6. Logging Security Events

```java
@Component
@RequiredArgsConstructor
public class SecurityEventLogger {

    private static final Logger securityLog = LoggerFactory.getLogger("SECURITY");

    @EventListener
    public void onAuthenticationSuccess(AuthenticationSuccessEvent event) {
        String username = event.getAuthentication().getName();
        securityLog.info("Authentication success for user: {}", username);
    }

    @EventListener
    public void onAuthenticationFailure(AbstractAuthenticationFailureEvent event) {
        String username = event.getAuthentication().getName();
        String reason = event.getException().getMessage();
        securityLog.warn("Authentication failed for user: {} - Reason: {}", username, reason);
    }

    @EventListener
    public void onAuthorizationDenied(AuthorizationDeniedEvent event) {
        Authentication auth = event.getAuthentication();
        securityLog.warn("Access denied for user: {} to resource: {}",
            auth.getName(), event.getSource());
    }
}
```

### 7. Rate Limiting

```java
@Component
public class RateLimitingFilter extends OncePerRequestFilter {

    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        String key = getClientIdentifier(request);
        Bucket bucket = buckets.computeIfAbsent(key, this::createBucket);

        if (bucket.tryConsume(1)) {
            filterChain.doFilter(request, response);
        } else {
            response.setStatus(429);
            response.getWriter().write("{\"error\": \"Too many requests\"}");
        }
    }

    private String getClientIdentifier(HttpServletRequest request) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth != null && auth.isAuthenticated()) {
            return "user:" + auth.getName();
        }
        return "ip:" + request.getRemoteAddr();
    }
}
```

### 8. Secure Configuration

```yaml
# application.yml - Production settings
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${JWT_ISSUER_URI}

# Never commit secrets to version control
jwt:
  secret: ${JWT_SECRET}

# Use environment variables for sensitive data
server:
  ssl:
    enabled: true
    key-store: ${SSL_KEYSTORE_PATH}
    key-store-password: ${SSL_KEYSTORE_PASSWORD}
```

### 9. Defense in Depth

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            // URL-level security (first layer)
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())

            // Session management
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))

            // Secure headers
            .headers(headers -> headers
                .frameOptions(frame -> frame.deny())
                .contentSecurityPolicy(csp ->
                    csp.policyDirectives("default-src 'self'")))

            // CORS and CSRF
            .cors(Customizer.withDefaults())
            .csrf(AbstractHttpConfigurer::disable) // Only for stateless APIs

            .build();
    }
}

// Method-level security (second layer)
@Service
public class AdminService {

    @PreAuthorize("hasRole('ADMIN') and #request.targetUserId != authentication.principal.id")
    public void deleteUser(DeleteUserRequest request) {
        // Additional business logic checks (third layer)
        validateDeletionRules(request);
        // ...
    }
}
```

### 10. Regular Security Audits

```java
// Use Spring Security's built-in debugging (development only)
@EnableWebSecurity(debug = true) // NEVER in production!
public class SecurityConfig {
    // ...
}

// Implement security health checks
@Component
public class SecurityHealthIndicator implements HealthIndicator {

    @Override
    public Health health() {
        try {
            // Check security configuration
            validateSecurityConfiguration();
            return Health.up()
                .withDetail("https", isHttpsEnabled())
                .withDetail("csrf", isCsrfEnabled())
                .build();
        } catch (Exception e) {
            return Health.down()
                .withException(e)
                .build();
        }
    }
}
```

---

## Additional Resources

- [Spring Security Reference Documentation](https://docs.spring.io/spring-security/reference/)
- [Spring Security Architecture](https://docs.spring.io/spring-security/reference/servlet/architecture.html)
- [OAuth 2.0 Authorization Framework](https://oauth.net/2/)
- [JSON Web Tokens (JWT)](https://jwt.io/)
- [OWASP Security Cheat Sheet](https://cheatsheetseries.owasp.org/)

---

## Quick Reference

### Common Annotations

| Annotation | Description |
|------------|-------------|
| `@EnableWebSecurity` | Enables Spring Security configuration |
| `@EnableMethodSecurity` | Enables method-level security |
| `@PreAuthorize` | Pre-authorization check before method execution |
| `@PostAuthorize` | Post-authorization check after method execution |
| `@Secured` | Role-based method security |
| `@RolesAllowed` | JSR-250 role-based method security |
| `@WithMockUser` | Mock authenticated user in tests |
| `@WithUserDetails` | Use real UserDetailsService in tests |

### Common Configuration Patterns

```java
// Permit all
.requestMatchers("/public/**").permitAll()

// Require authentication
.anyRequest().authenticated()

// Role-based
.requestMatchers("/admin/**").hasRole("ADMIN")

// Authority-based
.requestMatchers("/api/**").hasAuthority("API_ACCESS")

// Multiple roles
.requestMatchers("/manager/**").hasAnyRole("ADMIN", "MANAGER")

// HTTP method specific
.requestMatchers(HttpMethod.DELETE, "/api/**").hasRole("ADMIN")

// Deny all (secure by default)
.anyRequest().denyAll()
```

### HTTP Status Codes for Security

| Status | Meaning | When to Use |
|--------|---------|-------------|
| 401 | Unauthorized | Authentication required but not provided |
| 403 | Forbidden | Authenticated but lacks permission |
| 419 | Authentication Timeout | Session expired (non-standard) |
| 429 | Too Many Requests | Rate limit exceeded |
