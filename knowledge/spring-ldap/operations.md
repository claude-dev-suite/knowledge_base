# Spring LDAP - Operations

## Overview

Spring LDAP provides LdapTemplate for performing LDAP operations including search, bind, modify, and delete operations with built-in connection management.

## LdapTemplate Operations

### Search Operations
```java
@Service
@RequiredArgsConstructor
public class LdapSearchService {

    private final LdapTemplate ldapTemplate;

    // Search with AttributesMapper
    public List<User> findAllUsers() {
        return ldapTemplate.search(
            LdapQueryBuilder.query()
                .where("objectclass").is("person"),
            new UserAttributesMapper()
        );
    }

    // Search with filter
    public List<User> findByDepartment(String department) {
        return ldapTemplate.search(
            LdapQueryBuilder.query()
                .base("ou=users")
                .where("objectclass").is("person")
                .and("department").is(department),
            new UserAttributesMapper()
        );
    }

    // Search with complex filter
    public List<User> findByFilter(String name, String email) {
        return ldapTemplate.search(
            LdapQueryBuilder.query()
                .where("objectclass").is("person")
                .and(query()
                    .where("cn").like(name + "*")
                    .or("mail").is(email)),
            new UserAttributesMapper()
        );
    }

    // Search for single object
    public User findByUid(String uid) {
        return ldapTemplate.searchForObject(
            LdapQueryBuilder.query()
                .where("uid").is(uid),
            new UserAttributesMapper()
        );
    }

    // Search returning DNs
    public List<Name> findUserDns(String department) {
        return ldapTemplate.search(
            LdapQueryBuilder.query()
                .where("department").is(department),
            (NameClassPairMapper<Name>) result ->
                LdapUtils.newLdapName(result.getNameInNamespace())
        );
    }
}
```

### Context Mapper
```java
public class UserContextMapper implements ContextMapper<User> {

    @Override
    public User mapFromContext(Object ctx) throws NamingException {
        DirContextAdapter context = (DirContextAdapter) ctx;

        User user = new User();
        user.setDn(context.getDn());
        user.setUid(context.getStringAttribute("uid"));
        user.setFullName(context.getStringAttribute("cn"));
        user.setEmail(context.getStringAttribute("mail"));
        user.setDepartment(context.getStringAttribute("department"));
        user.setTelephone(context.getStringAttribute("telephoneNumber"));

        // Multi-valued attribute
        String[] groups = context.getStringAttributes("memberOf");
        if (groups != null) {
            user.setGroups(Arrays.asList(groups));
        }

        return user;
    }
}

@Service
public class ContextMapperService {

    @Autowired
    private LdapTemplate ldapTemplate;

    public List<User> findUsers() {
        return ldapTemplate.search(
            LdapQueryBuilder.query()
                .where("objectclass").is("inetOrgPerson"),
            new UserContextMapper()
        );
    }
}
```

### Bind Operations (Create)
```java
@Service
public class LdapBindService {

    @Autowired
    private LdapTemplate ldapTemplate;

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
        context.setAttributeValue("userPassword", user.getPassword());

        if (user.getDepartment() != null) {
            context.setAttributeValue("department", user.getDepartment());
        }

        ldapTemplate.bind(context);
    }

    // Using ODM
    public void createUserOdm(UserEntry user) {
        ldapTemplate.create(user);
    }
}
```

### Modify Operations
```java
@Service
public class LdapModifyService {

    @Autowired
    private LdapTemplate ldapTemplate;

    // Rebind (replace entire entry)
    public void updateUser(User user) {
        Name dn = LdapNameBuilder.newInstance()
            .add("ou", "users")
            .add("uid", user.getUid())
            .build();

        DirContextAdapter context = new DirContextAdapter(dn);
        mapToContext(user, context);
        ldapTemplate.rebind(context);
    }

    // Modify specific attributes
    public void updateEmail(String uid, String newEmail) {
        Name dn = LdapNameBuilder.newInstance()
            .add("ou", "users")
            .add("uid", uid)
            .build();

        ModificationItem[] mods = new ModificationItem[]{
            new ModificationItem(DirContext.REPLACE_ATTRIBUTE,
                new BasicAttribute("mail", newEmail))
        };

        ldapTemplate.modifyAttributes(dn, mods);
    }

    // Using DirContextOperations
    public void modifyUser(String uid, Consumer<DirContextOperations> modifier) {
        Name dn = LdapNameBuilder.newInstance()
            .add("ou", "users")
            .add("uid", uid)
            .build();

        DirContextOperations context = ldapTemplate.lookupContext(dn);
        modifier.accept(context);
        ldapTemplate.modifyAttributes(context);
    }

    // Add attribute value
    public void addGroupMembership(String uid, String groupDn) {
        Name dn = LdapNameBuilder.newInstance()
            .add("ou", "users")
            .add("uid", uid)
            .build();

        ModificationItem mod = new ModificationItem(
            DirContext.ADD_ATTRIBUTE,
            new BasicAttribute("memberOf", groupDn)
        );

        ldapTemplate.modifyAttributes(dn, new ModificationItem[]{mod});
    }

    // Remove attribute value
    public void removeGroupMembership(String uid, String groupDn) {
        Name dn = LdapNameBuilder.newInstance()
            .add("ou", "users")
            .add("uid", uid)
            .build();

        ModificationItem mod = new ModificationItem(
            DirContext.REMOVE_ATTRIBUTE,
            new BasicAttribute("memberOf", groupDn)
        );

        ldapTemplate.modifyAttributes(dn, new ModificationItem[]{mod});
    }
}
```

### Delete Operations
```java
@Service
public class LdapDeleteService {

    @Autowired
    private LdapTemplate ldapTemplate;

    public void deleteUser(String uid) {
        Name dn = LdapNameBuilder.newInstance()
            .add("ou", "users")
            .add("uid", uid)
            .build();

        ldapTemplate.unbind(dn);
    }

    public void deleteRecursively(Name dn) {
        ldapTemplate.unbind(dn, true);
    }

    // Using ODM
    public void deleteUserOdm(UserEntry user) {
        ldapTemplate.delete(user);
    }
}
```

### Rename/Move Operations
```java
@Service
public class LdapRenameService {

    @Autowired
    private LdapTemplate ldapTemplate;

    public void renameUser(String oldUid, String newUid) {
        Name oldDn = LdapNameBuilder.newInstance()
            .add("ou", "users")
            .add("uid", oldUid)
            .build();

        Name newDn = LdapNameBuilder.newInstance()
            .add("ou", "users")
            .add("uid", newUid)
            .build();

        ldapTemplate.rename(oldDn, newDn);
    }

    public void moveUser(String uid, String fromOu, String toOu) {
        Name oldDn = LdapNameBuilder.newInstance()
            .add("ou", fromOu)
            .add("uid", uid)
            .build();

        Name newDn = LdapNameBuilder.newInstance()
            .add("ou", toOu)
            .add("uid", uid)
            .build();

        ldapTemplate.rename(oldDn, newDn);
    }
}
```

## Group Operations

```java
@Service
public class GroupService {

    @Autowired
    private LdapTemplate ldapTemplate;

    public void createGroup(String groupName, List<String> memberDns) {
        Name dn = LdapNameBuilder.newInstance()
            .add("ou", "groups")
            .add("cn", groupName)
            .build();

        DirContextAdapter context = new DirContextAdapter(dn);
        context.setAttributeValues("objectclass",
            new String[]{"groupOfNames", "top"});
        context.setAttributeValue("cn", groupName);
        context.setAttributeValues("member",
            memberDns.toArray(new String[0]));

        ldapTemplate.bind(context);
    }

    public void addMember(String groupName, String memberDn) {
        Name groupDn = LdapNameBuilder.newInstance()
            .add("ou", "groups")
            .add("cn", groupName)
            .build();

        ModificationItem mod = new ModificationItem(
            DirContext.ADD_ATTRIBUTE,
            new BasicAttribute("member", memberDn)
        );

        ldapTemplate.modifyAttributes(groupDn, new ModificationItem[]{mod});
    }

    public List<String> getGroupMembers(String groupName) {
        Name groupDn = LdapNameBuilder.newInstance()
            .add("ou", "groups")
            .add("cn", groupName)
            .build();

        DirContextAdapter context = (DirContextAdapter)
            ldapTemplate.lookup(groupDn);

        String[] members = context.getStringAttributes("member");
        return members != null ? Arrays.asList(members) : Collections.emptyList();
    }
}
```

## Paged Search

```java
@Service
public class PagedSearchService {

    @Autowired
    private LdapTemplate ldapTemplate;

    public List<User> searchAllUsers() {
        List<User> allUsers = new ArrayList<>();

        PagedResultsDirContextProcessor processor =
            new PagedResultsDirContextProcessor(100); // Page size

        SearchControls controls = new SearchControls();
        controls.setSearchScope(SearchControls.SUBTREE_SCOPE);

        do {
            List<User> page = ldapTemplate.search(
                "ou=users",
                "(objectclass=person)",
                controls,
                new UserAttributesMapper(),
                processor
            );
            allUsers.addAll(page);
        } while (processor.hasMore());

        return allUsers;
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use LdapQueryBuilder | Build filters manually |
| Use DirContextAdapter | Raw attribute manipulation |
| Use paged results for large queries | Unbounded searches |
| Handle NamingException properly | Ignore exceptions |
| Use connection pooling | Create connections per operation |
| Escape special characters | Pass user input directly |

## Production Checklist

- [ ] Connection pooling enabled
- [ ] Search scope appropriate
- [ ] Pagination for large results
- [ ] Error handling implemented
- [ ] DN builders used consistently
- [ ] Attribute escaping in place
