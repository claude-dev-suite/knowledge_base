# Spring Authorization Server - Token Customization

## Overview

Spring Authorization Server allows extensive customization of OAuth2 tokens including JWT claims, token formats, and ID token content through customizers and context mappers.

## JWT Access Token Customization

### OAuth2TokenCustomizer
```java
@Bean
public OAuth2TokenCustomizer<JwtEncodingContext> jwtTokenCustomizer() {
    return context -> {
        if (OAuth2TokenType.ACCESS_TOKEN.equals(context.getTokenType())) {
            JwtClaimsSet.Builder claims = context.getClaims();

            // Add custom claims
            claims.claim("custom_claim", "custom_value");

            // Add user info from principal
            Authentication principal = context.getPrincipal();
            if (principal instanceof UsernamePasswordAuthenticationToken) {
                UserDetails user = (UserDetails) principal.getPrincipal();

                // Add roles as claim
                Set<String> roles = user.getAuthorities().stream()
                    .map(GrantedAuthority::getAuthority)
                    .collect(Collectors.toSet());
                claims.claim("roles", roles);

                // Add username
                claims.claim("username", user.getUsername());
            }

            // Add client info
            RegisteredClient client = context.getRegisteredClient();
            claims.claim("client_name", client.getClientName());

            // Add timestamp
            claims.claim("token_created", Instant.now().toString());
        }
    };
}
```

### Conditional Claims
```java
@Bean
public OAuth2TokenCustomizer<JwtEncodingContext> conditionalClaimsCustomizer(
        UserService userService) {
    return context -> {
        if (OAuth2TokenType.ACCESS_TOKEN.equals(context.getTokenType())) {
            JwtClaimsSet.Builder claims = context.getClaims();

            Authentication principal = context.getPrincipal();
            String username = principal.getName();

            // Add claims based on scopes
            Set<String> scopes = context.getAuthorizedScopes();

            if (scopes.contains("profile")) {
                User user = userService.findByUsername(username);
                claims.claim("full_name", user.getFullName());
                claims.claim("department", user.getDepartment());
            }

            if (scopes.contains("email")) {
                User user = userService.findByUsername(username);
                claims.claim("email", user.getEmail());
                claims.claim("email_verified", user.isEmailVerified());
            }

            if (scopes.contains("admin")) {
                claims.claim("admin_level", userService.getAdminLevel(username));
            }
        }
    };
}
```

## ID Token Customization

### OIDC ID Token Claims
```java
@Bean
public OAuth2TokenCustomizer<JwtEncodingContext> idTokenCustomizer(UserService userService) {
    return context -> {
        if (OidcParameterNames.ID_TOKEN.equals(context.getTokenType().getValue())) {
            JwtClaimsSet.Builder claims = context.getClaims();

            String username = context.getPrincipal().getName();
            User user = userService.findByUsername(username);

            // Standard OIDC claims
            claims.claim(StandardClaimNames.NAME, user.getFullName());
            claims.claim(StandardClaimNames.GIVEN_NAME, user.getFirstName());
            claims.claim(StandardClaimNames.FAMILY_NAME, user.getLastName());
            claims.claim(StandardClaimNames.EMAIL, user.getEmail());
            claims.claim(StandardClaimNames.EMAIL_VERIFIED, user.isEmailVerified());
            claims.claim(StandardClaimNames.PHONE_NUMBER, user.getPhoneNumber());
            claims.claim(StandardClaimNames.PICTURE, user.getAvatarUrl());
            claims.claim(StandardClaimNames.LOCALE, user.getLocale());
            claims.claim(StandardClaimNames.ZONEINFO, user.getTimezone());
            claims.claim(StandardClaimNames.UPDATED_AT, user.getUpdatedAt().toEpochSecond());

            // Address claim (nested object)
            if (context.getAuthorizedScopes().contains("address")) {
                Map<String, Object> address = Map.of(
                    "street_address", user.getStreetAddress(),
                    "locality", user.getCity(),
                    "region", user.getState(),
                    "postal_code", user.getPostalCode(),
                    "country", user.getCountry()
                );
                claims.claim(StandardClaimNames.ADDRESS, address);
            }

            // Custom claims
            claims.claim("tenant_id", user.getTenantId());
            claims.claim("subscription_tier", user.getSubscriptionTier());
        }
    };
}
```

## UserInfo Endpoint Customization

### OidcUserInfoMapper
```java
@Bean
public Function<OidcUserInfoAuthenticationContext, OidcUserInfo> userInfoMapper(
        UserService userService) {
    return context -> {
        OidcUserInfoAuthenticationToken authentication = context.getAuthentication();
        JwtAuthenticationToken principal = (JwtAuthenticationToken) authentication.getPrincipal();

        String username = principal.getName();
        User user = userService.findByUsername(username);
        Set<String> scopes = context.getAccessToken().getScopes();

        OidcUserInfo.Builder builder = OidcUserInfo.builder()
            .subject(username);

        if (scopes.contains(OidcScopes.PROFILE)) {
            builder
                .name(user.getFullName())
                .givenName(user.getFirstName())
                .familyName(user.getLastName())
                .nickname(user.getNickname())
                .preferredUsername(user.getUsername())
                .profile(user.getProfileUrl())
                .picture(user.getAvatarUrl())
                .website(user.getWebsite())
                .gender(user.getGender())
                .birthdate(user.getBirthdate().toString())
                .zoneinfo(user.getTimezone())
                .locale(user.getLocale())
                .updatedAt(user.getUpdatedAt().toString());
        }

        if (scopes.contains(OidcScopes.EMAIL)) {
            builder
                .email(user.getEmail())
                .emailVerified(user.isEmailVerified());
        }

        if (scopes.contains(OidcScopes.PHONE)) {
            builder
                .phoneNumber(user.getPhoneNumber())
                .phoneNumberVerified(user.isPhoneVerified());
        }

        if (scopes.contains(OidcScopes.ADDRESS)) {
            builder.address(OidcUserAddress.builder()
                .formatted(user.getFormattedAddress())
                .streetAddress(user.getStreetAddress())
                .locality(user.getCity())
                .region(user.getState())
                .postalCode(user.getPostalCode())
                .country(user.getCountry())
                .build());
        }

        // Custom claims
        builder.claim("tenant_id", user.getTenantId());
        builder.claim("roles", user.getRoles());

        return builder.build();
    };
}
```

## Token Response Customization

### OAuth2TokenGenerator
```java
@Bean
public OAuth2TokenGenerator<?> tokenGenerator(
        JWKSource<SecurityContext> jwkSource,
        OAuth2TokenCustomizer<JwtEncodingContext> jwtCustomizer) {

    JwtEncoder jwtEncoder = new NimbusJwtEncoder(jwkSource);
    JwtGenerator jwtGenerator = new JwtGenerator(jwtEncoder);
    jwtGenerator.setJwtCustomizer(jwtCustomizer);

    OAuth2AccessTokenGenerator accessTokenGenerator = new OAuth2AccessTokenGenerator();
    OAuth2RefreshTokenGenerator refreshTokenGenerator = new OAuth2RefreshTokenGenerator();

    return new DelegatingOAuth2TokenGenerator(
        jwtGenerator,
        accessTokenGenerator,
        refreshTokenGenerator
    );
}
```

### Access Token Response Customization
```java
@Bean
public OAuth2TokenCustomizer<OAuth2TokenClaimsContext> accessTokenCustomizer() {
    return context -> {
        OAuth2TokenClaimsSet.Builder claims = context.getClaims();

        // For opaque tokens (REFERENCE format)
        if (context.getTokenType().equals(OAuth2TokenType.ACCESS_TOKEN)) {
            claims.claim("custom_field", "value");
        }
    };
}
```

## Multi-Tenant Token Customization

```java
@Component
public class MultiTenantTokenCustomizer implements OAuth2TokenCustomizer<JwtEncodingContext> {

    private final TenantService tenantService;

    @Override
    public void customize(JwtEncodingContext context) {
        if (OAuth2TokenType.ACCESS_TOKEN.equals(context.getTokenType())) {
            JwtClaimsSet.Builder claims = context.getClaims();

            // Extract tenant from authentication
            Authentication principal = context.getPrincipal();
            String tenantId = extractTenantId(principal);

            if (tenantId != null) {
                Tenant tenant = tenantService.findById(tenantId);

                claims.claim("tenant_id", tenantId);
                claims.claim("tenant_name", tenant.getName());
                claims.claim("tenant_tier", tenant.getSubscriptionTier());

                // Tenant-specific audience
                claims.audience(List.of(
                    "https://api.example.com",
                    "https://" + tenantId + ".example.com"
                ));

                // Tenant-specific permissions
                Set<String> permissions = tenantService.getPermissions(tenantId, principal.getName());
                claims.claim("permissions", permissions);
            }
        }
    }

    private String extractTenantId(Authentication principal) {
        if (principal instanceof TenantAwareAuthentication tenantAuth) {
            return tenantAuth.getTenantId();
        }
        return null;
    }
}
```

## Refresh Token Customization

```java
@Bean
public OAuth2TokenCustomizer<JwtEncodingContext> refreshTokenCustomizer() {
    return context -> {
        if (OAuth2TokenType.REFRESH_TOKEN.equals(context.getTokenType())) {
            // Refresh tokens are typically opaque, but you can customize claims
            // if using JWT refresh tokens

            context.getClaims()
                .claim("token_family", UUID.randomUUID().toString())
                .claim("rotation_count", 0);
        }
    };
}
```

## Token Introspection Customization

```java
@Bean
public OAuth2TokenIntrospectionClaimsCustomizer introspectionCustomizer(
        UserService userService) {
    return context -> {
        OAuth2Authorization authorization = context.getAuthorization();

        // Add custom claims to introspection response
        context.getClaims()
            .claim("custom_active_claim", "value")
            .claim("user_status", userService.getStatus(authorization.getPrincipalName()));
    };
}
```

## Claims Based on Client

```java
@Bean
public OAuth2TokenCustomizer<JwtEncodingContext> clientBasedCustomizer() {
    return context -> {
        RegisteredClient client = context.getRegisteredClient();
        JwtClaimsSet.Builder claims = context.getClaims();

        // Different claims based on client
        String clientId = client.getClientId();

        if (clientId.startsWith("mobile-")) {
            claims.claim("device_type", "mobile");
            claims.claim("offline_capable", true);
        } else if (clientId.startsWith("web-")) {
            claims.claim("device_type", "web");
            claims.claim("session_tracking", true);
        } else if (clientId.startsWith("api-")) {
            claims.claim("device_type", "service");
            claims.claim("rate_limit_tier", getRateLimitTier(clientId));
        }

        // Client metadata as claims
        Map<String, Object> clientSettings = client.getClientSettings().getSettings();
        if (clientSettings.containsKey("custom_setting")) {
            claims.claim("client_custom", clientSettings.get("custom_setting"));
        }
    };

    private String getRateLimitTier(String clientId) {
        // Lookup rate limit tier for client
        return "standard";
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Add only necessary claims | Include sensitive data in tokens |
| Use scopes to control claims | Add all user data unconditionally |
| Keep tokens reasonably sized | Create bloated tokens |
| Use consistent claim names | Inconsistent naming conventions |
| Document custom claims | Undocumented proprietary claims |
| Validate claim values | Trust input without validation |

## Production Checklist

- [ ] Custom claims documented
- [ ] Scope-based claim filtering
- [ ] Sensitive data excluded
- [ ] Token size monitored
- [ ] UserInfo endpoint configured
- [ ] Multi-tenant support if needed
