# Spring LDAP - Basics

## Overview

Spring LDAP simplifies LDAP programming with template-based data access, object-directory mapping (ODM), and integration with Spring Security.

## Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-ldap</artifactId>
</dependency>
```

## Configuration

### application.yml
```yaml
spring:
  ldap:
    urls: ldap://localhost:389
    base: dc=example,dc=com
    username: cn=admin,dc=example,dc=com
    password: ${LDAP_PASSWORD}
```

### Java Configuration
```java
@Configuration
public class LdapConfig {

    @Bean
    public LdapContextSource contextSource() {
        LdapContextSource contextSource = new LdapContextSource();
        contextSource.setUrl("ldap://localhost:389");
        contextSource.setBase("dc=example,dc=com");
        contextSource.setUserDn("cn=admin,dc=example,dc=com");
        contextSource.setPassword("secret");
        contextSource.afterPropertiesSet();
        return contextSource;
    }

    @Bean
    public LdapTemplate ldapTemplate(LdapContextSource contextSource) {
        return new LdapTemplate(contextSource);
    }
}
```

## LdapTemplate

### Search Operations
```java
@Service
@RequiredArgsConstructor
public class UserLdapService {

    private final LdapTemplate ldapTemplate;

    public List<String> getAllUserNames() {
        return ldapTemplate.search(
            LdapQueryBuilder.query()
                .where("objectclass").is("person"),
            (AttributesMapper<String>) attrs -> (String) attrs.get("cn").get()
        );
    }

    public List<User> findByDepartment(String department) {
        return ldapTemplate.search(
            LdapQueryBuilder.query()
                .where("objectclass").is("person")
                .and("department").is(department),
            new UserAttributesMapper()
        );
    }

    public User findByUid(String uid) {
        return ldapTemplate.searchForObject(
            LdapQueryBuilder.query()
                .where("uid").is(uid),
            new UserAttributesMapper()
        );
    }
}
```

### AttributesMapper
```java
public class UserAttributesMapper implements AttributesMapper<User> {

    @Override
    public User mapFromAttributes(Attributes attrs) throws NamingException {
        User user = new User();
        user.setUid((String) attrs.get("uid").get());
        user.setCn((String) attrs.get("cn").get());
        user.setEmail(getAttributeValue(attrs, "mail"));
        user.setDepartment(getAttributeValue(attrs, "department"));
        return user;
    }

    private String getAttributeValue(Attributes attrs, String name) throws NamingException {
        Attribute attr = attrs.get(name);
        return attr != null ? (String) attr.get() : null;
    }
}
```

## Object-Directory Mapping (ODM)

### Entry Mapping
```java
@Entry(objectClasses = {"inetOrgPerson", "organizationalPerson", "person", "top"})
public class User {

    @Id
    private Name dn;

    @Attribute(name = "uid")
    private String uid;

    @Attribute(name = "cn")
    private String fullName;

    @Attribute(name = "sn")
    private String lastName;

    @Attribute(name = "mail")
    private String email;

    @Attribute(name = "department")
    private String department;

    @DnAttribute(value = "ou", index = 0)
    private String organizationalUnit;

    // getters and setters
}
```

### LdapRepository
```java
public interface UserRepository extends LdapRepository<User> {

    User findByUid(String uid);

    List<User> findByDepartment(String department);

    List<User> findByFullNameContaining(String name);

    @Query("(&(objectclass=person)(mail={0}))")
    User findByEmail(String email);
}
```

### Usage
```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;

    public User findByUid(String uid) {
        return userRepository.findByUid(uid);
    }

    public User create(User user) {
        Name dn = LdapNameBuilder.newInstance()
            .add("ou", user.getOrganizationalUnit())
            .add("uid", user.getUid())
            .build();
        user.setDn(dn);
        return userRepository.save(user);
    }

    public void update(User user) {
        userRepository.save(user);
    }

    public void delete(User user) {
        userRepository.delete(user);
    }
}
```

## CRUD Operations

### Create
```java
public void createUser(User user) {
    Name dn = LdapNameBuilder.newInstance()
        .add("ou", "users")
        .add("uid", user.getUid())
        .build();

    DirContextAdapter context = new DirContextAdapter(dn);
    context.setAttributeValues("objectclass",
        new String[]{"inetOrgPerson", "organizationalPerson", "person", "top"});
    context.setAttributeValue("uid", user.getUid());
    context.setAttributeValue("cn", user.getFullName());
    context.setAttributeValue("sn", user.getLastName());
    context.setAttributeValue("mail", user.getEmail());

    ldapTemplate.bind(context);
}
```

### Update
```java
public void updateUser(String uid, String newEmail) {
    Name dn = LdapNameBuilder.newInstance()
        .add("ou", "users")
        .add("uid", uid)
        .build();

    ModificationItem[] mods = new ModificationItem[] {
        new ModificationItem(DirContext.REPLACE_ATTRIBUTE,
            new BasicAttribute("mail", newEmail))
    };

    ldapTemplate.modifyAttributes(dn, mods);
}
```

### Delete
```java
public void deleteUser(String uid) {
    Name dn = LdapNameBuilder.newInstance()
        .add("ou", "users")
        .add("uid", uid)
        .build();

    ldapTemplate.unbind(dn);
}
```

## Authentication

### Simple Bind
```java
public boolean authenticate(String uid, String password) {
    try {
        ldapTemplate.authenticate(
            LdapQueryBuilder.query()
                .where("uid").is(uid),
            password
        );
        return true;
    } catch (AuthenticationException e) {
        return false;
    }
}
```

### With Spring Security
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .anyRequest().authenticated()
            )
            .formLogin(Customizer.withDefaults());
        return http.build();
    }

    @Bean
    public LdapAuthenticationProvider ldapAuthenticationProvider(
            LdapContextSource contextSource) {
        BindAuthenticator authenticator = new BindAuthenticator(contextSource);
        authenticator.setUserDnPatterns(new String[]{"uid={0},ou=users"});

        LdapAuthoritiesPopulator authoritiesPopulator =
            new DefaultLdapAuthoritiesPopulator(contextSource, "ou=groups");

        return new LdapAuthenticationProvider(authenticator, authoritiesPopulator);
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use connection pooling | Create new connections per request |
| Use ODM for complex mappings | Manual attribute mapping everywhere |
| Configure proper timeouts | Infinite connection timeouts |
| Use DN builders | Concatenate DN strings |
| Escape special characters | Pass user input directly |
| Paginate large searches | Return unbounded results |

## Production Checklist

- [ ] TLS/LDAPS configured
- [ ] Connection pooling enabled
- [ ] Timeouts configured
- [ ] Credentials secured
- [ ] Proper indexes on LDAP server
- [ ] Monitoring and logging
