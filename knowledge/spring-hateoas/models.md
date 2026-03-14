# Spring HATEOAS - Representation Models

## Model Types

### RepresentationModel

Base class for resources with links. Use when you need full control.

```java
public class UserModel extends RepresentationModel<UserModel> {

    private Long id;
    private String name;
    private String email;
    private LocalDateTime createdAt;

    public UserModel() {}

    public UserModel(User user) {
        this.id = user.getId();
        this.name = user.getName();
        this.email = user.getEmail();
        this.createdAt = user.getCreatedAt();
    }

    // getters, setters
}
```

### EntityModel

Wrapper for domain objects. Use when you want to add links to existing entities.

```java
EntityModel<User> userModel = EntityModel.of(user,
    linkTo(methodOn(UserController.class).getUser(user.getId())).withSelfRel(),
    linkTo(methodOn(UserController.class).getAllUsers()).withRel("users")
);
```

### CollectionModel

Wrapper for collections. Use for list responses.

```java
CollectionModel<EntityModel<User>> users = CollectionModel.of(
    userList.stream()
        .map(user -> EntityModel.of(user,
            linkTo(methodOn(UserController.class).getUser(user.getId())).withSelfRel()
        ))
        .toList(),
    linkTo(methodOn(UserController.class).getAllUsers()).withSelfRel()
);
```

### PagedModel

For paginated responses. Use with PagedResourcesAssembler.

```java
@Autowired
private PagedResourcesAssembler<User> pagedAssembler;

@GetMapping
public PagedModel<EntityModel<User>> getUsers(Pageable pageable) {
    Page<User> userPage = userService.findAll(pageable);
    return pagedAssembler.toModel(userPage);
}
```

## RepresentationModelAssembler

Reusable component for converting entities to models.

```java
@Component
public class UserModelAssembler implements RepresentationModelAssembler<User, EntityModel<User>> {

    @Override
    public EntityModel<User> toModel(User user) {
        return EntityModel.of(user,
            linkTo(methodOn(UserController.class).getUser(user.getId())).withSelfRel(),
            linkTo(methodOn(UserController.class).getUserOrders(user.getId())).withRel("orders"),
            linkTo(methodOn(UserController.class).getAllUsers()).withRel("users")
        );
    }

    @Override
    public CollectionModel<EntityModel<User>> toCollectionModel(Iterable<? extends User> users) {
        CollectionModel<EntityModel<User>> models = RepresentationModelAssembler.super.toCollectionModel(users);
        models.add(linkTo(methodOn(UserController.class).getAllUsers()).withSelfRel());
        return models;
    }
}
```

### Usage in Controller

```java
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;
    private final UserModelAssembler userAssembler;

    @GetMapping("/{id}")
    public EntityModel<User> getUser(@PathVariable Long id) {
        User user = userService.findById(id);
        return userAssembler.toModel(user);
    }

    @GetMapping
    public CollectionModel<EntityModel<User>> getAllUsers() {
        List<User> users = userService.findAll();
        return userAssembler.toCollectionModel(users);
    }
}
```

## Custom RepresentationModel

For complex responses with computed fields.

```java
public class OrderSummaryModel extends RepresentationModel<OrderSummaryModel> {

    private Long id;
    private String status;
    private BigDecimal total;
    private int itemCount;
    private LocalDateTime createdAt;
    private LocalDateTime estimatedDelivery;

    public static OrderSummaryModel from(Order order) {
        OrderSummaryModel model = new OrderSummaryModel();
        model.id = order.getId();
        model.status = order.getStatus().name();
        model.total = order.getTotal();
        model.itemCount = order.getItems().size();
        model.createdAt = order.getCreatedAt();
        model.estimatedDelivery = calculateEstimatedDelivery(order);

        model.add(linkTo(methodOn(OrderController.class).getOrder(order.getId())).withSelfRel());
        model.add(linkTo(methodOn(OrderController.class).getOrderItems(order.getId())).withRel("items"));

        if (order.isTrackable()) {
            model.add(linkTo(methodOn(OrderController.class).trackOrder(order.getId())).withRel("track"));
        }

        return model;
    }
}
```

## Embedded Resources

### Using _embedded

```java
public class OrderModel extends RepresentationModel<OrderModel> {

    private Long id;
    private String status;
    private BigDecimal total;

    @JsonProperty("_embedded")
    private EmbeddedResources embedded;

    public static class EmbeddedResources {
        private List<EntityModel<OrderItem>> items;
        private EntityModel<User> customer;
        // getters, setters
    }
}
```

### Using CollectionModel with Embedded

```java
@GetMapping("/{id}")
public EntityModel<Order> getOrder(@PathVariable Long id) {
    Order order = orderService.findById(id);
    List<OrderItem> items = orderItemService.findByOrderId(id);

    EntityModel<Order> model = EntityModel.of(order);

    // Add embedded items as links
    model.add(linkTo(methodOn(OrderController.class).getOrderItems(id)).withRel("items"));

    return model;
}
```

## HAL-FORMS Support

For hypermedia-driven forms.

```java
@GetMapping("/{id}")
public EntityModel<User> getUser(@PathVariable Long id) {
    User user = userService.findById(id);

    Link editLink = linkTo(methodOn(UserController.class).updateUser(id, null))
        .withRel("edit")
        .andAffordance(afford(methodOn(UserController.class).updateUser(id, null)));

    return EntityModel.of(user,
        linkTo(methodOn(UserController.class).getUser(id)).withSelfRel(),
        editLink
    );
}
```

## Projection Models

For different representations of the same entity.

```java
// Summary projection
public class UserSummaryModel extends RepresentationModel<UserSummaryModel> {
    private Long id;
    private String name;
}

// Detail projection
public class UserDetailModel extends RepresentationModel<UserDetailModel> {
    private Long id;
    private String name;
    private String email;
    private String phone;
    private AddressModel address;
    private List<OrderSummaryModel> recentOrders;
}

// Assemblers
@Component
public class UserSummaryAssembler implements RepresentationModelAssembler<User, UserSummaryModel> {
    @Override
    public UserSummaryModel toModel(User user) {
        UserSummaryModel model = new UserSummaryModel();
        model.setId(user.getId());
        model.setName(user.getName());
        model.add(linkTo(methodOn(UserController.class).getUser(user.getId())).withSelfRel());
        return model;
    }
}

@Component
public class UserDetailAssembler implements RepresentationModelAssembler<User, UserDetailModel> {
    @Override
    public UserDetailModel toModel(User user) {
        UserDetailModel model = new UserDetailModel();
        // ... populate all fields
        model.add(linkTo(methodOn(UserController.class).getUser(user.getId())).withSelfRel());
        model.add(linkTo(methodOn(UserController.class).getUserOrders(user.getId())).withRel("orders"));
        return model;
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use RepresentationModelAssembler | Duplicate assembly logic |
| Create specific model types | Expose domain entities directly |
| Add conditional links | Show unavailable actions |
| Use EntityModel for simple cases | Over-engineer simple responses |
| Include relevant embedded resources | Over-fetch related data |
| Follow HAL conventions | Custom response formats |
