# ASP.NET Core Minimal APIs

> Official Documentation: https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/overview

## Overview

Minimal APIs were introduced in .NET 6 to create HTTP APIs with minimal dependencies and ceremony. They use top-level statements and lambda expressions to define endpoints directly in `Program.cs` without the need for controllers. In .NET 8+, minimal APIs gained feature parity with controllers through endpoint filters, route groups, typed results, and improved parameter binding.

---

## 1. Basic Endpoint Definitions

### HTTP Method Handlers

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// GET
app.MapGet("/", () => "Hello World");

// GET with route parameter
app.MapGet("/products/{id:int}", (int id) => $"Product {id}");

// POST with body
app.MapPost("/products", (CreateProductDto dto) => Results.Created($"/products/1", dto));

// PUT
app.MapPut("/products/{id:int}", (int id, UpdateProductDto dto) => Results.NoContent());

// PATCH
app.MapPatch("/products/{id:int}", (int id, PatchProductDto dto) => Results.NoContent());

// DELETE
app.MapDelete("/products/{id:int}", (int id) => Results.NoContent());

app.Run();
```

### Async Handlers

```csharp
app.MapGet("/products", async (IProductService productService, CancellationToken ct) =>
{
    var products = await productService.GetAllAsync(ct);
    return Results.Ok(products);
});

app.MapGet("/products/{id:int}", async (int id, IProductService productService, CancellationToken ct) =>
{
    var product = await productService.GetByIdAsync(id, ct);
    return product is null ? Results.NotFound() : Results.Ok(product);
});

app.MapPost("/products", async (CreateProductDto dto, IProductService productService, CancellationToken ct) =>
{
    var product = await productService.CreateAsync(dto, ct);
    return Results.Created($"/products/{product.Id}", product);
});
```

---

## 2. Route Parameters and Constraints

```csharp
// Required route parameter with type constraint
app.MapGet("/users/{id:int}", (int id) => $"User {id}");

// String route parameter
app.MapGet("/users/{username}", (string username) => $"User {username}");

// GUID parameter
app.MapGet("/orders/{id:guid}", (Guid id) => $"Order {id}");

// Optional route parameter (with default)
app.MapGet("/products/page/{pageNumber:int?}",
    (int? pageNumber) => $"Page {pageNumber ?? 1}");

// Multiple constraints
app.MapGet("/items/{id:int:min(1):max(1000000)}",
    (int id) => $"Item {id}");

// Catch-all parameter
app.MapGet("/files/{*path}",
    (string path) => $"File: {path}");

// Multiple route parameters
app.MapGet("/api/{version:int}/categories/{categoryId:int}/products/{productId:int}",
    (int version, int categoryId, int productId) =>
        $"v{version} - Category {categoryId} - Product {productId}");
```

---

## 3. Parameter Binding

### Binding Source Attributes

| Source | Attribute | Default Inference |
|---|---|---|
| Route | `[FromRoute]` | Parameters matching route template names |
| Query string | `[FromQuery]` | Simple types not in route |
| Body | `[FromBody]` | Complex types (JSON) |
| Header | `[FromHeader]` | N/A (must be explicit) |
| Form | `[FromForm]` | IFormFile parameters |
| Services | `[FromServices]` | Registered DI services (auto-inferred) |
| Key | `[FromKeyedServices("key")]` | Keyed DI services (.NET 8+) |

```csharp
// Query string parameters
app.MapGet("/search", (
    [FromQuery] string q,
    [FromQuery(Name = "page")] int pageNumber = 1,
    [FromQuery] int pageSize = 20,
    [FromQuery] string? sortBy = null) =>
{
    return Results.Ok(new { q, pageNumber, pageSize, sortBy });
});
// GET /search?q=laptop&page=2&pageSize=10&sortBy=price

// Header binding
app.MapGet("/me", (
    [FromHeader(Name = "Authorization")] string authHeader,
    [FromHeader(Name = "X-Correlation-Id")] string? correlationId) =>
{
    return Results.Ok(new { authHeader, correlationId });
});

// Body binding (inferred for complex types)
app.MapPost("/products", (CreateProductDto dto) =>
{
    // dto is deserialized from JSON body
    return Results.Created($"/products/1", dto);
});

// Service injection (auto-inferred from DI registration)
app.MapGet("/products", async (
    IProductService productService,        // Resolved from DI
    ILogger<Program> logger,               // Resolved from DI
    CancellationToken ct) =>               // Bound from request
{
    logger.LogInformation("Fetching all products");
    var products = await productService.GetAllAsync(ct);
    return Results.Ok(products);
});

// Keyed services (.NET 8+)
app.MapPost("/upload", async (
    IFormFile file,
    [FromKeyedServices("s3")] IStorageService storage) =>
{
    var url = await storage.UploadAsync(file.OpenReadStream(), file.FileName);
    return Results.Ok(new { url });
});
```

### [AsParameters] for Parameter Objects

```csharp
public record ProductSearchParams(
    [FromQuery] string? Query,
    [FromQuery] decimal? MinPrice,
    [FromQuery] decimal? MaxPrice,
    [FromQuery] string? Category,
    [FromQuery] int Page = 1,
    [FromQuery] int PageSize = 20,
    [FromQuery] string SortBy = "name",
    [FromQuery] string SortDir = "asc");

app.MapGet("/products/search", async (
    [AsParameters] ProductSearchParams searchParams,
    IProductService productService,
    CancellationToken ct) =>
{
    var results = await productService.SearchAsync(searchParams, ct);
    return Results.Ok(results);
});
// GET /products/search?query=laptop&minPrice=500&category=electronics&page=1

public record PaginationParams(
    [FromQuery] int Page = 1,
    [FromQuery] int PageSize = 20);

public record GetProductParams(
    [FromRoute] int Id,
    [FromHeader(Name = "X-Include-Details")] bool IncludeDetails = false);

app.MapGet("/products/{id:int}", async (
    [AsParameters] GetProductParams p,
    IProductService productService,
    CancellationToken ct) =>
{
    var product = await productService.GetByIdAsync(p.Id, p.IncludeDetails, ct);
    return product is null ? Results.NotFound() : Results.Ok(product);
});
```

### File Upload

```csharp
// Single file
app.MapPost("/upload", async (IFormFile file, IStorageService storage) =>
{
    if (file.Length == 0)
        return Results.BadRequest("File is empty");

    if (file.Length > 10 * 1024 * 1024) // 10MB limit
        return Results.BadRequest("File exceeds 10MB limit");

    var allowedTypes = new[] { "image/jpeg", "image/png", "application/pdf" };
    if (!allowedTypes.Contains(file.ContentType))
        return Results.BadRequest("Unsupported file type");

    var url = await storage.UploadAsync(file.OpenReadStream(), file.FileName);
    return Results.Ok(new { url, file.FileName, file.Length });
})
.DisableAntiforgery(); // Required for file uploads without antiforgery token

// Multiple files
app.MapPost("/upload/batch", async (IFormFileCollection files, IStorageService storage) =>
{
    var results = new List<object>();
    foreach (var file in files)
    {
        var url = await storage.UploadAsync(file.OpenReadStream(), file.FileName);
        results.Add(new { url, file.FileName, file.Length });
    }
    return Results.Ok(results);
})
.DisableAntiforgery();
```

---

## 4. TypedResults

TypedResults provide compile-time type checking and automatic OpenAPI metadata generation.

### Results vs TypedResults

```csharp
// Results - returns IResult (no type info for OpenAPI)
app.MapGet("/products/{id:int}", async (int id, IProductService svc) =>
{
    var product = await svc.GetByIdAsync(id);
    return product is null ? Results.NotFound() : Results.Ok(product);
});

// TypedResults - returns typed result (OpenAPI metadata automatic)
app.MapGet("/products/{id:int}", async Task<Results<Ok<ProductDto>, NotFound>> (
    int id, IProductService svc) =>
{
    var product = await svc.GetByIdAsync(id);
    return product is null
        ? TypedResults.NotFound()
        : TypedResults.Ok(product);
});
```

### Common TypedResults Methods

| Method | Status | Description |
|---|---|---|
| `TypedResults.Ok(value)` | 200 | Success with body |
| `TypedResults.Created(uri, value)` | 201 | Resource created |
| `TypedResults.Accepted(uri)` | 202 | Accepted for processing |
| `TypedResults.NoContent()` | 204 | Success with no body |
| `TypedResults.BadRequest(error)` | 400 | Bad request |
| `TypedResults.Unauthorized()` | 401 | Unauthorized |
| `TypedResults.Forbid()` | 403 | Forbidden |
| `TypedResults.NotFound()` | 404 | Not found |
| `TypedResults.Conflict(error)` | 409 | Conflict |
| `TypedResults.UnprocessableEntity(error)` | 422 | Validation failure |
| `TypedResults.ValidationProblem(errors)` | 400 | Validation errors dict |
| `TypedResults.Problem(detail, status)` | Any | ProblemDetails |
| `TypedResults.File(bytes, contentType)` | 200 | File download |
| `TypedResults.Json(value)` | 200 | JSON response |

```csharp
// Full CRUD with TypedResults
app.MapPost("/products",
    async Task<Results<Created<ProductDto>, ValidationProblem, Conflict>> (
    CreateProductDto dto,
    IProductService svc,
    IValidator<CreateProductDto> validator,
    CancellationToken ct) =>
{
    var validation = await validator.ValidateAsync(dto, ct);
    if (!validation.IsValid)
    {
        return TypedResults.ValidationProblem(
            validation.Errors.ToDictionary(
                e => e.PropertyName,
                e => new[] { e.ErrorMessage }));
    }

    if (await svc.ExistsBySkuAsync(dto.Sku, ct))
        return TypedResults.Conflict();

    var product = await svc.CreateAsync(dto, ct);
    return TypedResults.Created($"/products/{product.Id}", product);
});

app.MapPut("/products/{id:int}",
    async Task<Results<NoContent, NotFound, ValidationProblem>> (
    int id, UpdateProductDto dto, IProductService svc, CancellationToken ct) =>
{
    if (!await svc.ExistsAsync(id, ct))
        return TypedResults.NotFound();

    await svc.UpdateAsync(id, dto, ct);
    return TypedResults.NoContent();
});

app.MapDelete("/products/{id:int}",
    async Task<Results<NoContent, NotFound>> (
    int id, IProductService svc, CancellationToken ct) =>
{
    if (!await svc.ExistsAsync(id, ct))
        return TypedResults.NotFound();

    await svc.DeleteAsync(id, ct);
    return TypedResults.NoContent();
});
```

---

## 5. Route Groups

Route groups allow organizing endpoints under a common prefix with shared metadata, filters, and conventions.

```csharp
var app = builder.Build();

// Basic group
var api = app.MapGroup("/api");
var v1 = api.MapGroup("/v1");

// Products group
var products = v1.MapGroup("/products")
    .WithTags("Products")               // OpenAPI tag
    .RequireAuthorization();             // All endpoints require auth

products.MapGet("/", async (IProductService svc, CancellationToken ct) =>
{
    var items = await svc.GetAllAsync(ct);
    return TypedResults.Ok(items);
});

products.MapGet("/{id:int}", async (int id, IProductService svc, CancellationToken ct) =>
{
    var product = await svc.GetByIdAsync(id, ct);
    return product is null ? TypedResults.NotFound() : TypedResults.Ok(product);
});

products.MapPost("/", async (CreateProductDto dto, IProductService svc, CancellationToken ct) =>
{
    var product = await svc.CreateAsync(dto, ct);
    return TypedResults.Created($"/api/v1/products/{product.Id}", product);
})
.RequireAuthorization("Admin"); // Additional auth for POST

products.MapDelete("/{id:int}", async (int id, IProductService svc, CancellationToken ct) =>
{
    await svc.DeleteAsync(id, ct);
    return TypedResults.NoContent();
})
.RequireAuthorization("Admin");

// Orders group with filters
var orders = v1.MapGroup("/orders")
    .WithTags("Orders")
    .RequireAuthorization()
    .AddEndpointFilter<ValidationFilter>();

orders.MapGet("/", GetAllOrders);
orders.MapGet("/{id:int}", GetOrderById);
orders.MapPost("/", CreateOrder);

app.Run();
```

---

## 6. Endpoint Filters

Endpoint filters are the minimal API equivalent of action filters in controllers.

```csharp
// Inline filter
app.MapPost("/products", async (CreateProductDto dto, IProductService svc) =>
{
    var product = await svc.CreateAsync(dto);
    return TypedResults.Created($"/products/{product.Id}", product);
})
.AddEndpointFilter(async (context, next) =>
{
    var logger = context.HttpContext.RequestServices.GetRequiredService<ILogger<Program>>();
    logger.LogInformation("Before handler");

    var result = await next(context);

    logger.LogInformation("After handler");
    return result;
});

// Typed filter class
public class ValidationFilter<T> : IEndpointFilter where T : class
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var dto = context.Arguments.OfType<T>().FirstOrDefault();

        if (dto is null)
        {
            return TypedResults.BadRequest("Request body is required");
        }

        var validator = context.HttpContext.RequestServices
            .GetService<IValidator<T>>();

        if (validator is not null)
        {
            var result = await validator.ValidateAsync(dto);
            if (!result.IsValid)
            {
                return TypedResults.ValidationProblem(
                    result.Errors.ToDictionary(
                        e => e.PropertyName,
                        e => new[] { e.ErrorMessage }));
            }
        }

        return await next(context);
    }
}

// Usage
app.MapPost("/products", CreateProduct)
    .AddEndpointFilter<ValidationFilter<CreateProductDto>>();

// Logging filter
public class LoggingFilter : IEndpointFilter
{
    private readonly ILogger<LoggingFilter> _logger;

    public LoggingFilter(ILogger<LoggingFilter> logger)
    {
        _logger = logger;
    }

    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var stopwatch = Stopwatch.StartNew();
        var path = context.HttpContext.Request.Path;
        var method = context.HttpContext.Request.Method;

        _logger.LogInformation("{Method} {Path} - Start", method, path);

        var result = await next(context);

        stopwatch.Stop();
        _logger.LogInformation(
            "{Method} {Path} - Completed in {Elapsed}ms",
            method, path, stopwatch.ElapsedMilliseconds);

        return result;
    }
}

// Apply filter to group (all endpoints in group get the filter)
var api = app.MapGroup("/api")
    .AddEndpointFilter<LoggingFilter>();
```

---

## 7. Endpoint Metadata and OpenAPI

```csharp
app.MapGet("/products/{id:int}", async (int id, IProductService svc) =>
{
    var product = await svc.GetByIdAsync(id);
    return product is null ? Results.NotFound() : Results.Ok(product);
})
.WithName("GetProductById")                      // Operation ID
.WithTags("Products")                            // OpenAPI tag
.WithSummary("Get a product by ID")              // Operation summary
.WithDescription("Returns a single product")     // Operation description
.Produces<ProductDto>(StatusCodes.Status200OK)    // Success response
.Produces(StatusCodes.Status404NotFound)          // Not found response
.WithOpenApi();                                   // Enable OpenAPI metadata

app.MapPost("/products", async (CreateProductDto dto, IProductService svc) =>
{
    var product = await svc.CreateAsync(dto);
    return Results.Created($"/products/{product.Id}", product);
})
.WithName("CreateProduct")
.WithTags("Products")
.Accepts<CreateProductDto>("application/json")
.Produces<ProductDto>(StatusCodes.Status201Created)
.ProducesValidationProblem()
.Produces(StatusCodes.Status409Conflict)
.RequireAuthorization("Admin")
.WithOpenApi();
```

---

## 8. Authorization and Rate Limiting

```csharp
// Authentication
app.MapGet("/products", GetAllProducts);         // Public

app.MapPost("/products", CreateProduct)
    .RequireAuthorization();                     // Any authenticated user

app.MapDelete("/products/{id}", DeleteProduct)
    .RequireAuthorization("Admin");              // Admin policy only

app.MapGet("/me", GetCurrentUser)
    .RequireAuthorization(new AuthorizeAttribute { Roles = "User,Admin" });

// Allow anonymous override on an authenticated group
var api = app.MapGroup("/api").RequireAuthorization();
api.MapGet("/health", () => Results.Ok("Healthy"))
    .AllowAnonymous();

// Rate limiting (.NET 8+)
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("fixed", opt =>
    {
        opt.PermitLimit = 100;
        opt.Window = TimeSpan.FromMinutes(1);
    });

    options.AddSlidingWindowLimiter("sliding", opt =>
    {
        opt.PermitLimit = 100;
        opt.Window = TimeSpan.FromMinutes(1);
        opt.SegmentsPerWindow = 4;
    });

    options.AddTokenBucketLimiter("token", opt =>
    {
        opt.TokenLimit = 100;
        opt.ReplenishmentPeriod = TimeSpan.FromSeconds(10);
        opt.TokensPerPeriod = 10;
    });
});

app.UseRateLimiter();

app.MapGet("/products", GetAllProducts)
    .RequireRateLimiting("fixed");

// CORS
builder.Services.AddCors(options =>
{
    options.AddPolicy("Frontend", policy =>
    {
        policy.WithOrigins("https://myapp.com")
            .AllowAnyMethod()
            .AllowAnyHeader()
            .AllowCredentials();
    });
});

app.UseCors();

app.MapGet("/products", GetAllProducts)
    .RequireCors("Frontend");
```

---

## 9. Organizing Minimal APIs

### Extension Methods Pattern

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddApplicationServices();

var app = builder.Build();
app.MapProductEndpoints();
app.MapOrderEndpoints();
app.MapUserEndpoints();
app.Run();

// Endpoints/ProductEndpoints.cs
namespace MyApp.Endpoints;

public static class ProductEndpoints
{
    public static WebApplication MapProductEndpoints(this WebApplication app)
    {
        var group = app.MapGroup("/api/products")
            .WithTags("Products")
            .RequireAuthorization();

        group.MapGet("/", GetAll);
        group.MapGet("/{id:int}", GetById);
        group.MapPost("/", Create).RequireAuthorization("Admin");
        group.MapPut("/{id:int}", Update).RequireAuthorization("Admin");
        group.MapDelete("/{id:int}", Delete).RequireAuthorization("Admin");

        return app;
    }

    private static async Task<Results<Ok<PagedResult<ProductDto>>, BadRequest>> GetAll(
        [AsParameters] ProductSearchParams search,
        IProductService svc,
        CancellationToken ct)
    {
        var result = await svc.SearchAsync(search, ct);
        return TypedResults.Ok(result);
    }

    private static async Task<Results<Ok<ProductDto>, NotFound>> GetById(
        int id,
        IProductService svc,
        CancellationToken ct)
    {
        var product = await svc.GetByIdAsync(id, ct);
        return product is null
            ? TypedResults.NotFound()
            : TypedResults.Ok(product);
    }

    private static async Task<Results<Created<ProductDto>, ValidationProblem, Conflict>> Create(
        CreateProductDto dto,
        IProductService svc,
        IValidator<CreateProductDto> validator,
        CancellationToken ct)
    {
        var validation = await validator.ValidateAsync(dto, ct);
        if (!validation.IsValid)
            return TypedResults.ValidationProblem(validation.ToDictionary());

        if (await svc.ExistsBySkuAsync(dto.Sku, ct))
            return TypedResults.Conflict();

        var product = await svc.CreateAsync(dto, ct);
        return TypedResults.Created($"/api/products/{product.Id}", product);
    }

    private static async Task<Results<NoContent, NotFound>> Update(
        int id,
        UpdateProductDto dto,
        IProductService svc,
        CancellationToken ct)
    {
        if (!await svc.ExistsAsync(id, ct))
            return TypedResults.NotFound();

        await svc.UpdateAsync(id, dto, ct);
        return TypedResults.NoContent();
    }

    private static async Task<Results<NoContent, NotFound>> Delete(
        int id,
        IProductService svc,
        CancellationToken ct)
    {
        if (!await svc.ExistsAsync(id, ct))
            return TypedResults.NotFound();

        await svc.DeleteAsync(id, ct);
        return TypedResults.NoContent();
    }
}

// Endpoints/OrderEndpoints.cs
public static class OrderEndpoints
{
    public static WebApplication MapOrderEndpoints(this WebApplication app)
    {
        var group = app.MapGroup("/api/orders")
            .WithTags("Orders")
            .RequireAuthorization()
            .AddEndpointFilter<LoggingFilter>();

        group.MapGet("/", GetAll);
        group.MapGet("/{id:int}", GetById);
        group.MapPost("/", Create);
        group.MapPut("/{id:int}/status", UpdateStatus);

        return app;
    }

    // ... handler methods
}
```

---

## 10. Minimal API vs Controllers Comparison

| Feature | Minimal APIs | Controllers |
|---|---|---|
| Setup | `MapGet`, `MapPost`, etc. | `[ApiController]`, action methods |
| Routing | Inline route templates | `[Route]`, `[HttpGet]` attributes |
| DI | Method parameter injection | Constructor injection |
| Filters | `AddEndpointFilter` | `[ServiceFilter]`, `[TypeFilter]` |
| Grouping | `MapGroup()` | Controller class + `[Route]` |
| Model validation | Manual or filter-based | Automatic with `[ApiController]` |
| OpenAPI | Explicit `.Produces()`, `.WithTags()` | `[ProducesResponseType]`, conventions |
| File uploads | `IFormFile` parameter | `[FromForm] IFormFile` |
| Content negotiation | Limited (JSON default) | Full (XML, CSV, custom) |
| Performance | Slightly faster (less overhead) | Slightly more overhead |
| Best for | Microservices, simple APIs | Large APIs, complex scenarios |

---

## Best Practices

1. **Organize with extension methods** - Move endpoint definitions out of `Program.cs` into static classes with `MapXxxEndpoints()` methods.
2. **Use TypedResults** - They provide automatic OpenAPI metadata and compile-time type safety.
3. **Use route groups** - Apply shared configuration (auth, tags, filters) at the group level.
4. **Use `[AsParameters]`** - Group related query parameters into a record for cleaner signatures.
5. **Apply endpoint filters** - Use `IEndpointFilter` for cross-cutting concerns like validation and logging.
6. **Accept `CancellationToken`** - Add it to every async handler for graceful cancellation.
7. **Document endpoints for OpenAPI** - Use `WithName`, `WithTags`, `Produces`, `WithSummary` for Swagger docs.
8. **Use named handlers** - Extract handler logic to static methods for testability and readability.

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---|---|---|
| Everything in `Program.cs` | Unmaintainable single file | Use extension methods and separate endpoint classes |
| Missing `DisableAntiforgery()` | File uploads fail with 400 | Add `.DisableAntiforgery()` to file upload endpoints |
| No model validation | Invalid data reaches handlers | Use endpoint filters or FluentValidation |
| Forgetting `CancellationToken` | Cannot cancel long-running requests | Always include `CancellationToken` parameter |
| Complex type in query string | Binding fails | Use `[AsParameters]` or `[FromQuery]` on individual properties |
| Services not resolved | `InvalidOperationException` | Ensure services are registered in DI before `Build()` |
| Missing OpenAPI metadata | Swagger shows incomplete docs | Use `Produces`, `WithTags`, `WithName` on every endpoint |
| Mixing Results and TypedResults | Inconsistent OpenAPI metadata | Pick one style per endpoint and use it consistently |
