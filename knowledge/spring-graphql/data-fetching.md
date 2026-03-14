# Spring GraphQL - Data Fetching

## DataLoader Pattern

DataLoader solves the N+1 query problem by batching and caching data fetches.

### Configuration

```java
@Configuration
public class DataLoaderConfig {

    @Bean
    public BatchLoaderRegistry batchLoaderRegistry() {
        return new DefaultBatchLoaderRegistry();
    }
}
```

### Register Batch Loaders

```java
@Component
public class DataLoaderRegistrar implements BatchLoaderRegistry {

    private final UserRepository userRepository;
    private final OrderRepository orderRepository;

    @Override
    public void registerBatchLoaders(BatchLoaderRegistry registry) {
        // User batch loader by ID
        registry.forTypePair(Long.class, User.class)
            .registerMappedBatchLoader((userIds, env) ->
                Mono.fromCallable(() -> userRepository.findAllById(userIds))
                    .map(users -> users.stream()
                        .collect(Collectors.toMap(User::getId, u -> u)))
            );

        // Orders by user ID
        registry.forTypePair(Long.class, List.class)
            .withName("ordersByUserId")
            .registerMappedBatchLoader((userIds, env) ->
                Mono.fromCallable(() -> {
                    List<Order> orders = orderRepository.findByUserIdIn(userIds);
                    return userIds.stream().collect(Collectors.toMap(
                        id -> id,
                        id -> orders.stream()
                            .filter(o -> o.getUserId().equals(id))
                            .toList()
                    ));
                })
            );
    }
}
```

### Using DataLoader in Controllers

```java
@Controller
public class UserController {

    @SchemaMapping(typeName = "Order")
    public CompletableFuture<User> user(Order order, DataLoader<Long, User> userLoader) {
        return userLoader.load(order.getUserId());
    }

    @SchemaMapping(typeName = "User")
    public CompletableFuture<List<Order>> orders(
            User user,
            @ContextValue DataLoader<Long, List<Order>> ordersByUserId) {
        return ordersByUserId.load(user.getId());
    }
}
```

## @BatchMapping (Recommended)

Spring's preferred approach for batch loading.

```java
@Controller
public class ProductController {

    private final ProductRepository productRepository;
    private final CategoryRepository categoryRepository;

    // Load categories for multiple products
    @BatchMapping
    public Map<Product, Category> category(List<Product> products) {
        Set<Long> categoryIds = products.stream()
            .map(Product::getCategoryId)
            .collect(Collectors.toSet());

        Map<Long, Category> categories = categoryRepository.findAllById(categoryIds)
            .stream()
            .collect(Collectors.toMap(Category::getId, c -> c));

        return products.stream()
            .collect(Collectors.toMap(
                product -> product,
                product -> categories.get(product.getCategoryId())
            ));
    }

    // Reactive batch mapping
    @BatchMapping
    public Flux<Category> categoryReactive(List<Product> products) {
        return categoryRepository.findAllByIdIn(
            products.stream().map(Product::getCategoryId).toList()
        );
    }
}
```

### List Values

```java
@Controller
public class OrderController {

    // Each Order has multiple OrderItems
    @BatchMapping
    public Map<Order, List<OrderItem>> items(List<Order> orders) {
        List<Long> orderIds = orders.stream()
            .map(Order::getId)
            .toList();

        List<OrderItem> allItems = orderItemRepository.findByOrderIdIn(orderIds);

        Map<Long, List<OrderItem>> itemsByOrderId = allItems.stream()
            .collect(Collectors.groupingBy(OrderItem::getOrderId));

        return orders.stream()
            .collect(Collectors.toMap(
                order -> order,
                order -> itemsByOrderId.getOrDefault(order.getId(), List.of())
            ));
    }
}
```

## Field Selection Optimization

```java
@QueryMapping
public List<Product> products(DataFetchingEnvironment env) {
    DataFetchingFieldSelectionSet selectionSet = env.getSelectionSet();

    // Check which fields are requested
    boolean includeCategory = selectionSet.contains("category");
    boolean includeReviews = selectionSet.contains("reviews");

    // Optimize query based on selection
    if (includeCategory && includeReviews) {
        return productRepository.findAllWithCategoryAndReviews();
    } else if (includeCategory) {
        return productRepository.findAllWithCategory();
    } else {
        return productRepository.findAll();
    }
}
```

## Pagination

### Offset-Based Pagination

```graphql
type Query {
    products(page: Int!, size: Int!): ProductPage!
}

type ProductPage {
    content: [Product!]!
    totalElements: Int!
    totalPages: Int!
    hasNext: Boolean!
}
```

```java
@QueryMapping
public ProductPage products(@Argument int page, @Argument int size) {
    Page<Product> productPage = productRepository.findAll(PageRequest.of(page, size));
    return new ProductPage(productPage);
}
```

### Cursor-Based Pagination

```graphql
type Query {
    products(first: Int!, after: String): ProductConnection!
}

type ProductConnection {
    edges: [ProductEdge!]!
    pageInfo: PageInfo!
}

type ProductEdge {
    cursor: String!
    node: Product!
}

type PageInfo {
    hasNextPage: Boolean!
    endCursor: String
}
```

```java
@QueryMapping
public ProductConnection products(@Argument int first, @Argument String after) {
    Long afterId = after != null ? decodeCursor(after) : 0L;

    List<Product> products = productRepository.findByIdGreaterThan(afterId,
        PageRequest.of(0, first + 1));

    boolean hasNextPage = products.size() > first;
    List<Product> pageContent = hasNextPage
        ? products.subList(0, first)
        : products;

    List<ProductEdge> edges = pageContent.stream()
        .map(p -> new ProductEdge(encodeCursor(p.getId()), p))
        .toList();

    String endCursor = edges.isEmpty() ? null : edges.get(edges.size() - 1).cursor();

    return new ProductConnection(edges, new PageInfo(hasNextPage, endCursor));
}
```

## Caching

### Query Result Caching

```java
@Controller
public class ProductController {

    private final CacheManager cacheManager;

    @QueryMapping
    @Cacheable(value = "products", key = "#id")
    public Product productById(@Argument Long id) {
        return productRepository.findById(id).orElseThrow();
    }

    @MutationMapping
    @CacheEvict(value = "products", key = "#id")
    public Product updateProduct(@Argument Long id, @Argument UpdateProductInput input) {
        return productService.update(id, input);
    }
}
```

### DataLoader Caching

```java
registry.forTypePair(Long.class, User.class)
    .withOptions(options -> options
        .setCachingEnabled(true)
        .setBatchingEnabled(true)
        .setMaxBatchSize(100)
    )
    .registerMappedBatchLoader((ids, env) -> loadUsers(ids));
```

## Async Data Fetching

```java
@Controller
public class ProductController {

    @QueryMapping
    public CompletableFuture<Product> productById(@Argument Long id) {
        return CompletableFuture.supplyAsync(() ->
            productRepository.findById(id).orElseThrow()
        );
    }

    @QueryMapping
    public Mono<Product> productByIdReactive(@Argument Long id) {
        return productRepository.findById(id);
    }

    @QueryMapping
    public Flux<Product> products() {
        return productRepository.findAll();
    }
}
```

## Error Handling in Data Fetching

```java
@Controller
public class ProductController {

    @QueryMapping
    public Product productById(@Argument Long id) {
        return productRepository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
    }

    @SchemaMapping
    public List<Review> reviews(Product product) {
        try {
            return reviewService.findByProductId(product.getId());
        } catch (ExternalServiceException e) {
            // Return partial data with error
            throw new DataFetcherExceptionResolver.ResolverException(
                "Reviews temporarily unavailable",
                List.of("reviews")
            );
        }
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use @BatchMapping for collections | N+1 queries in loops |
| Enable DataLoader caching | Fetch same data multiple times |
| Check field selection | Always fetch all relations |
| Use cursor pagination for large sets | Offset pagination with high offsets |
| Handle partial failures gracefully | All-or-nothing responses |
| Async fetch independent data | Sequential dependent fetches |
