# Spring Authorization Server - Configuration

## Overview

Spring Authorization Server configuration defines registered clients, authorization grants, token settings, and security policies for OAuth 2.0 and OpenID Connect.

## Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-authorization-server</artifactId>
</dependency>
```

## Basic Configuration

### Authorization Server Config
```java
@Configuration
@EnableWebSecurity
public class AuthorizationServerConfig {

    @Bean
    @Order(1)
    public SecurityFilterChain authorizationServerSecurityFilterChain(HttpSecurity http)
            throws Exception {
        OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http);

        http.getConfigurer(OAuth2AuthorizationServerConfigurer.class)
            .oidc(Customizer.withDefaults());

        http
            .exceptionHandling(exceptions -> exceptions
                .defaultAuthenticationEntryPointFor(
                    new LoginUrlAuthenticationEntryPoint("/login"),
                    new MediaTypeRequestMatcher(MediaType.TEXT_HTML)
                )
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));

        return http.build();
    }

    @Bean
    @Order(2)
    public SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authorize -> authorize
                .anyRequest().authenticated()
            )
            .formLogin(Customizer.withDefaults());

        return http.build();
    }
}
```

## Client Registration

### In-Memory Client Repository
```java
@Bean
public RegisteredClientRepository registeredClientRepository() {
    RegisteredClient webClient = RegisteredClient.withId(UUID.randomUUID().toString())
        .clientId("web-client")
        .clientSecret("{noop}secret")
        .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
        .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
        .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
        .redirectUri("http://localhost:8080/login/oauth2/code/web-client")
        .postLogoutRedirectUri("http://localhost:8080/")
        .scope(OidcScopes.OPENID)
        .scope(OidcScopes.PROFILE)
        .scope("read")
        .scope("write")
        .clientSettings(ClientSettings.builder()
            .requireAuthorizationConsent(true)
            .build())
        .tokenSettings(TokenSettings.builder()
            .accessTokenTimeToLive(Duration.ofHours(1))
            .refreshTokenTimeToLive(Duration.ofDays(7))
            .build())
        .build();

    RegisteredClient machineClient = RegisteredClient.withId(UUID.randomUUID().toString())
        .clientId("machine-client")
        .clientSecret("{noop}machine-secret")
        .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
        .authorizationGrantType(AuthorizationGrantType.CLIENT_CREDENTIALS)
        .scope("api.read")
        .scope("api.write")
        .tokenSettings(TokenSettings.builder()
            .accessTokenTimeToLive(Duration.ofMinutes(30))
            .build())
        .build();

    return new InMemoryRegisteredClientRepository(webClient, machineClient);
}
```

### JDBC Client Repository
```java
@Bean
public RegisteredClientRepository registeredClientRepository(JdbcTemplate jdbcTemplate) {
    return new JdbcRegisteredClientRepository(jdbcTemplate);
}

// Required schema - oauth2_registered_client table
// Run: classpath:org/springframework/security/oauth2/server/authorization/
//      client/oauth2-registered-client-schema.sql
```

### Client Authentication Methods
```java
RegisteredClient pkceClient = RegisteredClient.withId(UUID.randomUUID().toString())
    .clientId("public-client")
    // No client secret for public clients
    .clientAuthenticationMethod(ClientAuthenticationMethod.NONE)
    .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
    .redirectUri("http://localhost:3000/callback")
    .scope(OidcScopes.OPENID)
    .clientSettings(ClientSettings.builder()
        .requireProofKey(true)  // PKCE required
        .build())
    .build();

RegisteredClient privateKeyClient = RegisteredClient.withId(UUID.randomUUID().toString())
    .clientId("private-key-client")
    .clientAuthenticationMethod(ClientAuthenticationMethod.PRIVATE_KEY_JWT)
    .authorizationGrantType(AuthorizationGrantType.CLIENT_CREDENTIALS)
    .scope("api.read")
    .clientSettings(ClientSettings.builder()
        .jwkSetUrl("https://client.example.com/.well-known/jwks.json")
        .tokenEndpointAuthenticationSigningAlgorithm(SignatureAlgorithm.RS256)
        .build())
    .build();
```

## Token Settings

### Access Token Configuration
```java
@Bean
public RegisteredClientRepository registeredClientRepository() {
    RegisteredClient client = RegisteredClient.withId(UUID.randomUUID().toString())
        .clientId("my-client")
        .clientSecret("{bcrypt}$2a$10$...")
        .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
        .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
        .redirectUri("https://example.com/callback")
        .scope(OidcScopes.OPENID)
        .tokenSettings(TokenSettings.builder()
            // Access token settings
            .accessTokenTimeToLive(Duration.ofMinutes(30))
            .accessTokenFormat(OAuth2TokenFormat.SELF_CONTAINED)  // JWT
            // .accessTokenFormat(OAuth2TokenFormat.REFERENCE)    // Opaque

            // Refresh token settings
            .refreshTokenTimeToLive(Duration.ofDays(30))
            .reuseRefreshTokens(false)  // Rotate refresh tokens

            // ID token settings
            .idTokenSignatureAlgorithm(SignatureAlgorithm.RS256)

            // Authorization code settings
            .authorizationCodeTimeToLive(Duration.ofMinutes(5))

            // Device code settings (for device flow)
            .deviceCodeTimeToLive(Duration.ofMinutes(5))

            .build())
        .build();

    return new InMemoryRegisteredClientRepository(client);
}
```

## JWK Source Configuration

### RSA Key Pair
```java
@Bean
public JWKSource<SecurityContext> jwkSource() {
    KeyPair keyPair = generateRsaKey();
    RSAPublicKey publicKey = (RSAPublicKey) keyPair.getPublic();
    RSAPrivateKey privateKey = (RSAPrivateKey) keyPair.getPrivate();

    RSAKey rsaKey = new RSAKey.Builder(publicKey)
        .privateKey(privateKey)
        .keyID(UUID.randomUUID().toString())
        .build();

    JWKSet jwkSet = new JWKSet(rsaKey);
    return new ImmutableJWKSet<>(jwkSet);
}

private static KeyPair generateRsaKey() {
    try {
        KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("RSA");
        keyPairGenerator.initialize(2048);
        return keyPairGenerator.generateKeyPair();
    } catch (Exception ex) {
        throw new IllegalStateException(ex);
    }
}

@Bean
public JwtDecoder jwtDecoder(JWKSource<SecurityContext> jwkSource) {
    return OAuth2AuthorizationServerConfiguration.jwtDecoder(jwkSource);
}
```

### Multiple Keys (Key Rotation)
```java
@Bean
public JWKSource<SecurityContext> jwkSource() {
    RSAKey currentKey = generateRsaKey("current-key");
    RSAKey previousKey = generateRsaKey("previous-key");

    JWKSet jwkSet = new JWKSet(List.of(currentKey, previousKey));
    return (jwkSelector, context) -> jwkSelector.select(jwkSet);
}

// Periodically rotate keys and update JWKSet
```

## Authorization Server Settings

```java
@Bean
public AuthorizationServerSettings authorizationServerSettings() {
    return AuthorizationServerSettings.builder()
        .issuer("https://auth.example.com")
        .authorizationEndpoint("/oauth2/authorize")
        .deviceAuthorizationEndpoint("/oauth2/device_authorization")
        .deviceVerificationEndpoint("/oauth2/device_verification")
        .tokenEndpoint("/oauth2/token")
        .jwkSetEndpoint("/oauth2/jwks")
        .tokenRevocationEndpoint("/oauth2/revoke")
        .tokenIntrospectionEndpoint("/oauth2/introspect")
        .oidcClientRegistrationEndpoint("/connect/register")
        .oidcUserInfoEndpoint("/userinfo")
        .oidcLogoutEndpoint("/connect/logout")
        .build();
}
```

## User Authentication

### User Details Service
```java
@Bean
public UserDetailsService userDetailsService() {
    UserDetails user = User.builder()
        .username("user")
        .password("{noop}password")
        .roles("USER")
        .build();

    UserDetails admin = User.builder()
        .username("admin")
        .password("{bcrypt}$2a$10$...")
        .roles("USER", "ADMIN")
        .build();

    return new InMemoryUserDetailsManager(user, admin);
}

@Bean
public PasswordEncoder passwordEncoder() {
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
}
```

### Database User Store
```java
@Service
public class JpaUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        return userRepository.findByUsername(username)
            .map(user -> User.builder()
                .username(user.getUsername())
                .password(user.getPassword())
                .authorities(user.getRoles().stream()
                    .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
                    .toList())
                .accountLocked(!user.isActive())
                .build())
            .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));
    }
}
```

## JDBC Token Store

```java
@Bean
public OAuth2AuthorizationService authorizationService(
        JdbcTemplate jdbcTemplate,
        RegisteredClientRepository registeredClientRepository) {
    return new JdbcOAuth2AuthorizationService(jdbcTemplate, registeredClientRepository);
}

@Bean
public OAuth2AuthorizationConsentService authorizationConsentService(
        JdbcTemplate jdbcTemplate,
        RegisteredClientRepository registeredClientRepository) {
    return new JdbcOAuth2AuthorizationConsentService(jdbcTemplate, registeredClientRepository);
}

// Required schema files from spring-authorization-server
// oauth2-authorization-schema.sql
// oauth2-authorization-consent-schema.sql
```

## Application Properties

```yaml
spring:
  security:
    oauth2:
      authorizationserver:
        issuer: https://auth.example.com

server:
  port: 9000

logging:
  level:
    org.springframework.security: DEBUG
    org.springframework.security.oauth2: TRACE
```

## Best Practices

| Do | Don't |
|----|-------|
| Use PKCE for public clients | Allow public clients without PKCE |
| Rotate signing keys | Use single key forever |
| Use JDBC for production | In-memory for production |
| Set appropriate token TTLs | Long-lived access tokens |
| Require consent for sensitive scopes | Auto-approve all scopes |
| Use bcrypt for secrets | Store plain text secrets |

## Production Checklist

- [ ] HTTPS enabled
- [ ] JDBC repositories configured
- [ ] Key rotation strategy defined
- [ ] Token TTLs appropriate
- [ ] Client secrets encrypted
- [ ] Consent UI customized
- [ ] Monitoring enabled
