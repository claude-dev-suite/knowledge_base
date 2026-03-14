# Spring LDAP - Security

## Overview

Spring LDAP integrates with Spring Security for LDAP-based authentication and authorization, supporting bind authentication, password comparison, and group-based authorities.

## Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-ldap</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-ldap</artifactId>
</dependency>
```

## Basic LDAP Authentication

### Configuration
```java
@Configuration
@EnableWebSecurity
public class LdapSecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .formLogin(Customizer.withDefaults());

        return http.build();
    }

    @Bean
    public LdapContextSource contextSource() {
        LdapContextSource contextSource = new LdapContextSource();
        contextSource.setUrl("ldap://localhost:389");
        contextSource.setBase("dc=example,dc=com");
        contextSource.setUserDn("cn=admin,dc=example,dc=com");
        contextSource.setPassword("admin-password");
        return contextSource;
    }

    @Bean
    public LdapAuthenticationProvider ldapAuthenticationProvider(
            LdapContextSource contextSource) {

        BindAuthenticator authenticator = new BindAuthenticator(contextSource);
        authenticator.setUserDnPatterns(new String[]{"uid={0},ou=users"});

        DefaultLdapAuthoritiesPopulator authoritiesPopulator =
            new DefaultLdapAuthoritiesPopulator(contextSource, "ou=groups");
        authoritiesPopulator.setGroupSearchFilter("(member={0})");
        authoritiesPopulator.setGroupRoleAttribute("cn");
        authoritiesPopulator.setRolePrefix("ROLE_");

        return new LdapAuthenticationProvider(authenticator, authoritiesPopulator);
    }
}
```

## Bind Authentication

### User DN Patterns
```java
@Bean
public LdapAuthenticationProvider bindAuthProvider(LdapContextSource contextSource) {
    BindAuthenticator authenticator = new BindAuthenticator(contextSource);

    // Multiple patterns tried in order
    authenticator.setUserDnPatterns(new String[]{
        "uid={0},ou=users",
        "cn={0},ou=people",
        "mail={0},ou=employees"
    });

    return new LdapAuthenticationProvider(authenticator);
}
```

### User Search
```java
@Bean
public LdapAuthenticationProvider searchBindAuthProvider(LdapContextSource contextSource) {
    BindAuthenticator authenticator = new BindAuthenticator(contextSource);

    // Search for user first, then bind
    FilterBasedLdapUserSearch userSearch = new FilterBasedLdapUserSearch(
        "ou=users",                    // Search base
        "(uid={0})",                   // Search filter
        contextSource
    );
    authenticator.setUserSearch(userSearch);

    return new LdapAuthenticationProvider(authenticator);
}
```

## Password Comparison

```java
@Bean
public LdapAuthenticationProvider passwordCompareAuthProvider(
        LdapContextSource contextSource) {

    PasswordComparisonAuthenticator authenticator =
        new PasswordComparisonAuthenticator(contextSource);

    authenticator.setUserDnPatterns(new String[]{"uid={0},ou=users"});
    authenticator.setPasswordAttributeName("userPassword");
    authenticator.setPasswordEncoder(new BCryptPasswordEncoder());

    return new LdapAuthenticationProvider(authenticator);
}
```

## Authority Mapping

### Group-Based Authorities
```java
@Bean
public LdapAuthoritiesPopulator authoritiesPopulator(LdapContextSource contextSource) {
    DefaultLdapAuthoritiesPopulator populator =
        new DefaultLdapAuthoritiesPopulator(contextSource, "ou=groups");

    populator.setGroupSearchFilter("(member={0})");
    populator.setGroupRoleAttribute("cn");
    populator.setRolePrefix("ROLE_");
    populator.setSearchSubtree(true);
    populator.setConvertToUpperCase(true);

    // Default role for all users
    populator.setDefaultRole("ROLE_USER");

    return populator;
}
```

### Custom Authorities Populator
```java
public class CustomLdapAuthoritiesPopulator implements LdapAuthoritiesPopulator {

    private final LdapTemplate ldapTemplate;

    @Override
    public Collection<? extends GrantedAuthority> getGrantedAuthorities(
            DirContextOperations userData, String username) {

        Set<GrantedAuthority> authorities = new HashSet<>();

        // Add base role
        authorities.add(new SimpleGrantedAuthority("ROLE_USER"));

        // Get department-based role
        String department = userData.getStringAttribute("department");
        if ("IT".equals(department)) {
            authorities.add(new SimpleGrantedAuthority("ROLE_TECH"));
        }

        // Get group memberships
        String[] groups = userData.getStringAttributes("memberOf");
        if (groups != null) {
            for (String groupDn : groups) {
                String groupName = extractGroupName(groupDn);
                authorities.add(new SimpleGrantedAuthority("ROLE_" + groupName.toUpperCase()));
            }
        }

        return authorities;
    }

    private String extractGroupName(String groupDn) {
        // Extract CN from DN
        LdapName name = LdapUtils.newLdapName(groupDn);
        return LdapUtils.getStringValue(name, "cn");
    }
}
```

## UserDetailsContextMapper

```java
public class CustomUserDetailsContextMapper implements UserDetailsContextMapper {

    @Override
    public UserDetails mapUserFromContext(
            DirContextOperations ctx,
            String username,
            Collection<? extends GrantedAuthority> authorities) {

        String email = ctx.getStringAttribute("mail");
        String fullName = ctx.getStringAttribute("cn");
        String department = ctx.getStringAttribute("department");
        boolean enabled = !"disabled".equals(ctx.getStringAttribute("status"));

        CustomUserDetails user = new CustomUserDetails();
        user.setUsername(username);
        user.setEmail(email);
        user.setFullName(fullName);
        user.setDepartment(department);
        user.setEnabled(enabled);
        user.setAuthorities(authorities);

        return user;
    }

    @Override
    public void mapUserToContext(UserDetails user, DirContextAdapter ctx) {
        // For updating user in LDAP
        if (user instanceof CustomUserDetails custom) {
            ctx.setAttributeValue("mail", custom.getEmail());
            ctx.setAttributeValue("cn", custom.getFullName());
        }
    }
}

@Bean
public LdapAuthenticationProvider customMappingProvider(LdapContextSource contextSource) {
    BindAuthenticator authenticator = new BindAuthenticator(contextSource);
    authenticator.setUserDnPatterns(new String[]{"uid={0},ou=users"});

    LdapAuthenticationProvider provider =
        new LdapAuthenticationProvider(authenticator, authoritiesPopulator());

    provider.setUserDetailsContextMapper(new CustomUserDetailsContextMapper());

    return provider;
}
```

## Active Directory Integration

```java
@Configuration
@EnableWebSecurity
public class ActiveDirectorySecurityConfig {

    @Bean
    public ActiveDirectoryLdapAuthenticationProvider adAuthProvider() {
        ActiveDirectoryLdapAuthenticationProvider provider =
            new ActiveDirectoryLdapAuthenticationProvider(
                "example.com",           // Domain
                "ldap://ad.example.com"  // URL
            );

        provider.setSearchFilter("(&(objectClass=user)(sAMAccountName={1}))");
        provider.setConvertSubErrorCodesToExceptions(true);

        provider.setAuthoritiesMapper(authorities -> {
            Set<GrantedAuthority> mapped = new HashSet<>();
            for (GrantedAuthority authority : authorities) {
                // Map AD groups to roles
                String role = authority.getAuthority();
                if (role.startsWith("DOMAIN_ADMINS")) {
                    mapped.add(new SimpleGrantedAuthority("ROLE_ADMIN"));
                } else if (role.startsWith("DEVELOPERS")) {
                    mapped.add(new SimpleGrantedAuthority("ROLE_DEVELOPER"));
                }
            }
            mapped.add(new SimpleGrantedAuthority("ROLE_USER"));
            return mapped;
        });

        return provider;
    }
}
```

## Password Policy

```java
@Bean
public LdapAuthenticationProvider passwordPolicyProvider(LdapContextSource contextSource) {
    BindAuthenticator authenticator = new BindAuthenticator(contextSource);
    authenticator.setUserDnPatterns(new String[]{"uid={0},ou=users"});

    LdapAuthenticationProvider provider =
        new LdapAuthenticationProvider(authenticator);

    // Handle password policy responses
    provider.setAuthenticationErrorHandler((exception) -> {
        if (exception instanceof PasswordPolicyException ppe) {
            switch (ppe.getStatus()) {
                case PASSWORD_EXPIRED -> throw new CredentialsExpiredException("Password expired");
                case ACCOUNT_LOCKED -> throw new LockedException("Account locked");
                case MUST_CHANGE_PASSWORD -> throw new CredentialsExpiredException("Must change password");
            }
        }
    });

    return provider;
}
```

## LDAPS (Secure LDAP)

```java
@Bean
public LdapContextSource secureContextSource() {
    LdapContextSource contextSource = new LdapContextSource();
    contextSource.setUrl("ldaps://ldap.example.com:636");
    contextSource.setBase("dc=example,dc=com");
    contextSource.setUserDn("cn=admin,dc=example,dc=com");
    contextSource.setPassword("password");

    // Trust all certificates (dev only!)
    // contextSource.setAuthenticationStrategy(new SimpleDirContextAuthenticationStrategy());

    return contextSource;
}

// Production: Use proper truststore
@Bean
public LdapContextSource productionSecureContextSource() {
    DefaultSpringSecurityContextSource contextSource =
        new DefaultSpringSecurityContextSource("ldaps://ldap.example.com:636/dc=example,dc=com");

    Map<String, Object> env = new HashMap<>();
    env.put("java.naming.ldap.factory.socket", "javax.net.ssl.SSLSocketFactory");
    contextSource.setBaseEnvironmentProperties(env);

    return contextSource;
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use bind authentication | Store LDAP passwords locally |
| Configure proper role mapping | Hard-code roles |
| Use LDAPS in production | Plain LDAP for auth |
| Implement password policies | Skip password validation |
| Cache authentication results | Hit LDAP for every request |
| Handle locked accounts | Ignore account status |

## Production Checklist

- [ ] LDAPS configured
- [ ] Bind authentication enabled
- [ ] Authorities populator configured
- [ ] Password policies handled
- [ ] Account lockout handling
- [ ] Connection pooling enabled
- [ ] Audit logging in place
