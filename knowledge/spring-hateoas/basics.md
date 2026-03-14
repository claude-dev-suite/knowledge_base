# Spring HATEOAS - Basics

## Overview

Spring HATEOAS provides APIs to create REST representations that follow the HATEOAS (Hypermedia as the Engine of Application State) principle. It helps create self-describing APIs where responses include links to related resources.

## Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-hateoas</artifactId>
</dependency>
```

## Core Concepts

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      HATEOAS Response                                    │
│                                                                         │
│  {                                                                       │
│    "id": 1,                                                              │
│    "name": "Product",                                                    │
│    "_links": {                                     ◀── Hypermedia Links  │
│      "self": { "href": "/products/1" },                                 │
│      "category": { "href": "/categories/5" },                           │
│      "reviews": { "href": "/products/1/reviews" }                       │
│    }                                                                     │
│  }                                                                       │
└─────────────────────────────────────────────────────────────────────────┘
```

## RepresentationModel

Base class for resources with hypermedia links.

```java
public class UserModel extends RepresentationModel<UserModel> {

    private Long id;
    private String name;
    private String email;

    // getters, setters, constructors
}
```

### Usage in Controller

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public UserModel getUser(@PathVariable Long id) {
        User user = userService.findById(id);

        UserModel model = new UserModel();
        model.setId(user.getId());
        model.setName(user.getName());
        model.setEmail(user.getEmail());

        // Add self link
        model.add(linkTo(methodOn(UserController.class).getUser(id)).withSelfRel());

        // Add related links
        model.add(linkTo(methodOn(UserController.class).getUserOrders(id)).withRel("orders"));
        model.add(linkTo(methodOn(UserController.class).getAllUsers()).withRel("users"));

        return model;
    }
}
```

## EntityModel

Wrapper for domain objects with links.

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public EntityModel<User> getUser(@PathVariable Long id) {
        User user = userService.findById(id);

        return EntityModel.of(user,
            linkTo(methodOn(UserController.class).getUser(id)).withSelfRel(),
            linkTo(methodOn(UserController.class).getUserOrders(id)).withRel("orders"),
            linkTo(methodOn(UserController.class).getAllUsers()).withRel("users")
        );
    }
}
```

## CollectionModel

Wrapper for collections with links.

```java
@GetMapping
public CollectionModel<EntityModel<User>> getAllUsers() {
    List<EntityModel<User>> users = userService.findAll().stream()
        .map(user -> EntityModel.of(user,
            linkTo(methodOn(UserController.class).getUser(user.getId())).withSelfRel(),
            linkTo(methodOn(UserController.class).getAllUsers()).withRel("users")
        ))
        .toList();

    return CollectionModel.of(users,
        linkTo(methodOn(UserController.class).getAllUsers()).withSelfRel()
    );
}
```

## PagedModel

For paginated resources.

```java
@GetMapping
public PagedModel<EntityModel<User>> getUsers(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size) {

    Page<User> userPage = userService.findAll(PageRequest.of(page, size));

    PagedModel<EntityModel<User>> pagedModel = pagedResourcesAssembler.toModel(
        userPage,
        user -> EntityModel.of(user,
            linkTo(methodOn(UserController.class).getUser(user.getId())).withSelfRel()
        )
    );

    return pagedModel;
}
```

## Link Building

### Using WebMvcLinkBuilder

```java
import static org.springframework.hateoas.server.mvc.WebMvcLinkBuilder.*;

// Link to method
Link selfLink = linkTo(methodOn(UserController.class).getUser(1L)).withSelfRel();

// Link to controller
Link usersLink = linkTo(UserController.class).withRel("users");

// With path variables
Link userLink = linkTo(UserController.class).slash(userId).withRel("user");

// Templated links
Link searchLink = linkTo(methodOn(UserController.class).searchUsers(null, null))
    .withRel("search");
```

### Link Relations

```java
// Standard relations
link.withRel(IanaLinkRelations.SELF);
link.withRel(IanaLinkRelations.NEXT);
link.withRel(IanaLinkRelations.PREV);
link.withRel(IanaLinkRelations.FIRST);
link.withRel(IanaLinkRelations.LAST);
link.withRel(IanaLinkRelations.COLLECTION);
link.withRel(IanaLinkRelations.ITEM);

// Custom relations
link.withRel("orders");
link.withRel("reviews");
```

## Configuration

### application.yml
```yaml
spring:
  hateoas:
    use-hal-as-default-json-media-type: true
```

### Custom HAL Configuration

```java
@Configuration
public class HateoasConfig {

    @Bean
    public HalConfiguration halConfiguration() {
        return new HalConfiguration()
            .withRenderSingleLinks(HalConfiguration.RenderSingleLinks.AS_ARRAY)
            .withRenderSingleLinksFor(
                IanaLinkRelations.ITEM,
                HalConfiguration.RenderSingleLinks.AS_SINGLE
            );
    }
}
```

## Response Format (HAL)

```json
{
  "id": 1,
  "name": "John Doe",
  "email": "john@example.com",
  "_links": {
    "self": {
      "href": "http://localhost:8080/api/users/1"
    },
    "orders": {
      "href": "http://localhost:8080/api/users/1/orders"
    },
    "users": {
      "href": "http://localhost:8080/api/users"
    }
  }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use standard link relations | Invent custom relations for common cases |
| Include self links | Omit navigation links |
| Use RepresentationModelAssembler | Duplicate assembly logic |
| Include pagination links | Manual pagination handling |
| Document link relations | Leave relations undocumented |

## Production Checklist

- [ ] All resources have self links
- [ ] Related resources are linked
- [ ] Pagination includes navigation links
- [ ] Link relations follow standards
- [ ] API documentation includes link descriptions
- [ ] HAL format validated
