# Spring HATEOAS - Links

## Link Building

### WebMvcLinkBuilder

```java
import static org.springframework.hateoas.server.mvc.WebMvcLinkBuilder.*;

@RestController
@RequestMapping("/api/products")
public class ProductController {

    @GetMapping("/{id}")
    public EntityModel<Product> getProduct(@PathVariable Long id) {
        Product product = productService.findById(id);

        return EntityModel.of(product,
            // Self link
            linkTo(methodOn(ProductController.class).getProduct(id)).withSelfRel(),

            // Related resources
            linkTo(methodOn(ProductController.class).getProductReviews(id)).withRel("reviews"),
            linkTo(methodOn(CategoryController.class).getCategory(product.getCategoryId())).withRel("category"),

            // Collection link
            linkTo(methodOn(ProductController.class).getAllProducts()).withRel("products"),

            // Actions
            linkTo(methodOn(ProductController.class).deleteProduct(id)).withRel("delete"),
            linkTo(methodOn(CartController.class).addToCart(id, null)).withRel("add-to-cart")
        );
    }
}
```

### Link Types

```java
// Basic link
Link selfLink = Link.of("/products/1");

// With relation
Link ordersLink = Link.of("/users/1/orders", "orders");

// From LinkBuilder
Link productLink = linkTo(ProductController.class).slash(productId).withRel("product");

// Method reference
Link detailLink = linkTo(methodOn(ProductController.class).getProduct(1L)).withSelfRel();
```

### Templated Links

```java
@GetMapping
public CollectionModel<EntityModel<Product>> searchProducts(
        @RequestParam(required = false) String name,
        @RequestParam(required = false) String category) {

    List<Product> products = productService.search(name, category);

    // Create templated link for search
    Link searchLink = linkTo(methodOn(ProductController.class)
        .searchProducts(null, null))
        .withRel("search");

    // Result: "search": { "href": "/products?name={name}&category={category}", "templated": true }

    return CollectionModel.of(toModels(products), searchLink);
}
```

### Affordances

Describe available actions on a resource.

```java
@GetMapping("/{id}")
public EntityModel<Product> getProduct(@PathVariable Long id) {
    Product product = productService.findById(id);

    Link selfLink = linkTo(methodOn(ProductController.class).getProduct(id))
        .withSelfRel()
        .andAffordance(afford(methodOn(ProductController.class).updateProduct(id, null)))
        .andAffordance(afford(methodOn(ProductController.class).deleteProduct(id)));

    return EntityModel.of(product, selfLink);
}
```

## Link Relations

### Standard IANA Relations

```java
import org.springframework.hateoas.IanaLinkRelations;

// Navigation
.withRel(IanaLinkRelations.SELF)      // self
.withRel(IanaLinkRelations.NEXT)      // next
.withRel(IanaLinkRelations.PREV)      // prev
.withRel(IanaLinkRelations.FIRST)     // first
.withRel(IanaLinkRelations.LAST)      // last

// Collection/Item
.withRel(IanaLinkRelations.COLLECTION) // collection
.withRel(IanaLinkRelations.ITEM)       // item

// Actions
.withRel(IanaLinkRelations.EDIT)       // edit
.withRel(IanaLinkRelations.DELETE)     // delete (custom, not IANA)

// Related
.withRel(IanaLinkRelations.RELATED)    // related
.withRel(IanaLinkRelations.AUTHOR)     // author
```

### Custom Link Relations

```java
public final class CustomLinkRelations {

    public static final LinkRelation ORDERS = LinkRelation.of("orders");
    public static final LinkRelation REVIEWS = LinkRelation.of("reviews");
    public static final LinkRelation CART = LinkRelation.of("cart");
    public static final LinkRelation CHECKOUT = LinkRelation.of("checkout");
    public static final LinkRelation ADD_TO_CART = LinkRelation.of("add-to-cart");
    public static final LinkRelation REMOVE_FROM_CART = LinkRelation.of("remove-from-cart");

    private CustomLinkRelations() {}
}

// Usage
.withRel(CustomLinkRelations.ORDERS)
.withRel(CustomLinkRelations.ADD_TO_CART)
```

## Conditional Links

```java
@GetMapping("/{id}")
public EntityModel<Order> getOrder(@PathVariable Long id, Principal principal) {
    Order order = orderService.findById(id);
    User user = userService.findByEmail(principal.getName());

    EntityModel<Order> model = EntityModel.of(order,
        linkTo(methodOn(OrderController.class).getOrder(id, null)).withSelfRel()
    );

    // Conditional links based on state
    if (order.getStatus() == OrderStatus.PENDING) {
        model.add(linkTo(methodOn(OrderController.class).cancelOrder(id)).withRel("cancel"));
    }

    if (order.getStatus() == OrderStatus.SHIPPED) {
        model.add(linkTo(methodOn(OrderController.class).trackOrder(id)).withRel("track"));
    }

    // Conditional links based on user role
    if (user.isAdmin()) {
        model.add(linkTo(methodOn(OrderController.class).refundOrder(id)).withRel("refund"));
    }

    // Conditional links based on ownership
    if (order.getUserId().equals(user.getId())) {
        model.add(linkTo(methodOn(OrderController.class).getOrderInvoice(id)).withRel("invoice"));
    }

    return model;
}
```

## Link Customization

### Link with Type and Title

```java
Link documentLink = Link.of("/documents/1")
    .withRel("document")
    .withType("application/pdf")
    .withTitle("Download Invoice");

// Result: { "href": "/documents/1", "rel": "document", "type": "application/pdf", "title": "Download Invoice" }
```

### Link with Profile

```java
Link profiledLink = Link.of("/users/1")
    .withRel("user")
    .withProfile("https://api.example.com/profiles/user");
```

## Pagination Links

```java
@GetMapping
public PagedModel<EntityModel<Product>> getProducts(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        PagedResourcesAssembler<Product> assembler) {

    Page<Product> productPage = productService.findAll(PageRequest.of(page, size));

    return assembler.toModel(productPage,
        product -> EntityModel.of(product,
            linkTo(methodOn(ProductController.class).getProduct(product.getId())).withSelfRel()
        ),
        linkTo(methodOn(ProductController.class).getProducts(page, size)).withSelfRel()
    );
}
```

Response includes:
```json
{
  "_embedded": {
    "products": [...]
  },
  "_links": {
    "self": { "href": "/products?page=1&size=20" },
    "first": { "href": "/products?page=0&size=20" },
    "prev": { "href": "/products?page=0&size=20" },
    "next": { "href": "/products?page=2&size=20" },
    "last": { "href": "/products?page=10&size=20" }
  },
  "page": {
    "size": 20,
    "totalElements": 200,
    "totalPages": 10,
    "number": 1
  }
}
```

## WebFlux Support

```java
import static org.springframework.hateoas.server.reactive.WebFluxLinkBuilder.*;

@RestController
@RequestMapping("/api/products")
public class ReactiveProductController {

    @GetMapping("/{id}")
    public Mono<EntityModel<Product>> getProduct(@PathVariable Long id) {
        return productService.findById(id)
            .flatMap(product ->
                linkTo(methodOn(ReactiveProductController.class).getProduct(id)).withSelfRel()
                    .andAffordance(methodOn(ReactiveProductController.class).updateProduct(id, null))
                    .toMono()
                    .map(selfLink -> EntityModel.of(product, selfLink))
            );
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use methodOn for type-safe links | Hardcode URLs |
| Include relevant navigation links | Overload with all possible links |
| Use standard IANA relations | Invent custom relations unnecessarily |
| Add conditional links based on state | Show unavailable actions |
| Document custom link relations | Leave relations undocumented |
| Use templated links for search | Hardcode query parameters |
