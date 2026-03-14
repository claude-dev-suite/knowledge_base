# Spring GraphQL - Controllers

## Controller Annotations

### @QueryMapping
Maps methods to GraphQL Query fields.

```java
@Controller
public class ProductController {

    @QueryMapping
    public Product productById(@Argument Long id) {
        return productService.findById(id);
    }

    @QueryMapping("allProducts")  // Explicit field name
    public List<Product> products() {
        return productService.findAll();
    }

    @QueryMapping
    public Page<Product> searchProducts(
            @Argument String query,
            @Argument int page,
            @Argument int size) {
        return productService.search(query, PageRequest.of(page, size));
    }
}
```

### @MutationMapping
Maps methods to GraphQL Mutation fields.

```java
@Controller
public class ProductController {

    @MutationMapping
    public Product createProduct(@Argument CreateProductInput input) {
        return productService.create(input);
    }

    @MutationMapping
    public Product updateProduct(
            @Argument Long id,
            @Argument UpdateProductInput input) {
        return productService.update(id, input);
    }

    @MutationMapping
    public boolean deleteProduct(@Argument Long id) {
        return productService.delete(id);
    }
}
```

### @SubscriptionMapping
Maps methods to GraphQL Subscription fields.

```java
@Controller
public class ProductController {

    private final Sinks.Many<Product> productSink =
        Sinks.many().multicast().onBackpressureBuffer();

    @SubscriptionMapping
    public Flux<Product> productCreated() {
        return productSink.asFlux();
    }

    @SubscriptionMapping
    public Flux<Product> productUpdated(@Argument Long id) {
        return productSink.asFlux()
            .filter(product -> product.getId().equals(id));
    }
}
```

### @SchemaMapping
Maps methods to fields of specific types.

```java
@Controller
public class OrderController {

    // Resolves Order.items field
    @SchemaMapping(typeName = "Order", field = "items")
    public List<OrderItem> items(Order order) {
        return orderItemService.findByOrderId(order.getId());
    }

    // Shorthand: typeName inferred from method parameter type
    @SchemaMapping
    public User user(Order order) {
        return userService.findById(order.getUserId());
    }

    // Resolves Order.totalWithTax field
    @SchemaMapping(typeName = "Order")
    public BigDecimal totalWithTax(Order order) {
        return order.getTotal().multiply(new BigDecimal("1.10"));
    }
}
```

### @BatchMapping
Efficient batch loading for nested fields.

```java
@Controller
public class UserController {

    // Batch load orders for multiple users
    @BatchMapping
    public Map<User, List<Order>> orders(List<User> users) {
        List<Long> userIds = users.stream()
            .map(User::getId)
            .toList();

        List<Order> orders = orderService.findByUserIds(userIds);

        return users.stream().collect(Collectors.toMap(
            user -> user,
            user -> orders.stream()
                .filter(order -> order.getUserId().equals(user.getId()))
                .toList()
        ));
    }

    // Reactive batch mapping
    @BatchMapping
    public Mono<Map<User, List<Order>>> ordersReactive(List<User> users) {
        return orderService.findByUserIds(
            users.stream().map(User::getId).toList()
        )
        .collectList()
        .map(orders -> users.stream().collect(Collectors.toMap(
            user -> user,
            user -> orders.stream()
                .filter(o -> o.getUserId().equals(user.getId()))
                .toList()
        )));
    }
}
```

## Argument Handling

### @Argument
```java
@QueryMapping
public List<Product> searchProducts(
        @Argument String query,
        @Argument(name = "categoryId") Long category,
        @Argument BigDecimal minPrice,
        @Argument BigDecimal maxPrice) {
    return productService.search(query, category, minPrice, maxPrice);
}
```

### Complex Input Types
```java
// Schema
// input ProductFilter {
//     name: String
//     category: String
//     priceRange: PriceRange
// }

@QueryMapping
public List<Product> filterProducts(@Argument ProductFilter filter) {
    return productService.filter(filter);
}

public record ProductFilter(
    String name,
    String category,
    PriceRange priceRange
) {}

public record PriceRange(BigDecimal min, BigDecimal max) {}
```

### Arguments from Container
```java
@QueryMapping
public List<Product> products(@Arguments ProductQueryArgs args) {
    return productService.findAll(args);
}

public record ProductQueryArgs(
    int page,
    int size,
    String sortBy,
    String sortDirection
) {}
```

## Context Access

### GraphQL Context
```java
@QueryMapping
public User currentUser(GraphQLContext context) {
    return context.get("currentUser");
}

@QueryMapping
public List<Order> myOrders(DataFetchingEnvironment env) {
    GraphQLContext context = env.getGraphQlContext();
    User user = context.get("currentUser");
    return orderService.findByUserId(user.getId());
}
```

### DataFetchingEnvironment
```java
@QueryMapping
public List<Product> products(DataFetchingEnvironment env) {
    // Access selected fields
    DataFetchingFieldSelectionSet selectionSet = env.getSelectionSet();
    boolean includeCategory = selectionSet.contains("category");

    // Access arguments
    Map<String, Object> arguments = env.getArguments();

    // Access context
    Object context = env.getContext();

    return productService.findAll(includeCategory);
}
```

## Security

### @PreAuthorize
```java
@Controller
public class AdminController {

    @QueryMapping
    @PreAuthorize("hasRole('ADMIN')")
    public List<User> allUsers() {
        return userService.findAll();
    }

    @MutationMapping
    @PreAuthorize("hasRole('ADMIN')")
    public boolean deleteUser(@Argument Long id) {
        return userService.delete(id);
    }
}
```

### Current User Access
```java
@Controller
public class UserController {

    @QueryMapping
    public User me(@AuthenticationPrincipal UserDetails userDetails) {
        return userService.findByEmail(userDetails.getUsername());
    }

    @QueryMapping
    public List<Order> myOrders(Principal principal) {
        User user = userService.findByEmail(principal.getName());
        return orderService.findByUserId(user.getId());
    }
}
```

## Validation

```java
@Controller
@Validated
public class ProductController {

    @MutationMapping
    public Product createProduct(
            @Argument @Valid CreateProductInput input) {
        return productService.create(input);
    }
}

public record CreateProductInput(
    @NotBlank String name,
    @NotBlank String description,
    @Positive BigDecimal price,
    @Min(0) Integer stock
) {}
```

## Pagination

```java
// Schema
// type Query {
//     products(page: Int!, size: Int!): ProductConnection!
// }
//
// type ProductConnection {
//     content: [Product!]!
//     pageInfo: PageInfo!
// }
//
// type PageInfo {
//     totalElements: Int!
//     totalPages: Int!
//     hasNext: Boolean!
//     hasPrevious: Boolean!
// }

@Controller
public class ProductController {

    @QueryMapping
    public ProductConnection products(
            @Argument int page,
            @Argument int size) {
        Page<Product> productPage = productService.findAll(PageRequest.of(page, size));
        return new ProductConnection(productPage);
    }
}

public record ProductConnection(
    List<Product> content,
    PageInfo pageInfo
) {
    public ProductConnection(Page<Product> page) {
        this(
            page.getContent(),
            new PageInfo(
                page.getTotalElements(),
                page.getTotalPages(),
                page.hasNext(),
                page.hasPrevious()
            )
        );
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use @BatchMapping for N+1 | Fetch in loops |
| Validate input types | Trust client data |
| Use security annotations | Manual auth checks everywhere |
| Return specific types | Return generic Map |
| Handle nulls properly | Ignore optional fields |
| Document with schema | Skip documentation |
