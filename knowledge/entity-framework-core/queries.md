# Entity Framework Core Queries

> Official Documentation: https://learn.microsoft.com/ef/core/querying/

## Table of Contents

1. [Basic LINQ Queries](#basic-linq-queries)
2. [Loading Related Data](#loading-related-data)
3. [Filtered Includes](#filtered-includes)
4. [Projection Queries](#projection-queries)
5. [No-Tracking Queries](#no-tracking-queries)
6. [Split Queries](#split-queries)
7. [Raw SQL](#raw-sql)
8. [SqlQuery for Unmapped Types](#sqlquery-for-unmapped-types)
9. [Bulk Operations](#bulk-operations)
10. [Compiled Queries](#compiled-queries)
11. [Global Query Filters](#global-query-filters)
12. [Pagination Patterns](#pagination-patterns)
13. [Dynamic Filtering and Sorting](#dynamic-filtering-and-sorting)
14. [Client vs Server Evaluation](#client-vs-server-evaluation)
15. [Query Tags](#query-tags)
16. [Best Practices and Common Pitfalls](#best-practices-and-common-pitfalls)

---

## Basic LINQ Queries

### Filtering with Where

```csharp
// Single condition
var activeProducts = await context.Products
    .Where(p => p.IsActive)
    .ToListAsync();

// Multiple conditions
var expensiveElectronics = await context.Products
    .Where(p => p.Price > 100m && p.Category.Name == "Electronics")
    .ToListAsync();

// String operations
var searchResults = await context.Products
    .Where(p => p.Name.Contains("phone") || p.Description.StartsWith("Smart"))
    .ToListAsync();

// EF.Functions for SQL-specific operations
var fuzzySearch = await context.Products
    .Where(p => EF.Functions.Like(p.Name, "%phone%"))
    .ToListAsync();

// Case-insensitive search (SQL Server is CI by default, but for explicit control)
var results = await context.Products
    .Where(p => EF.Functions.Collate(p.Name, "SQL_Latin1_General_CP1_CI_AS") == "iphone")
    .ToListAsync();
```

### Ordering

```csharp
// Single order
var products = await context.Products
    .OrderBy(p => p.Name)
    .ToListAsync();

// Descending
var newest = await context.Products
    .OrderByDescending(p => p.CreatedAt)
    .ToListAsync();

// Multiple sort criteria
var sorted = await context.Products
    .OrderBy(p => p.Category.Name)
    .ThenByDescending(p => p.Price)
    .ThenBy(p => p.Name)
    .ToListAsync();
```

### Element Operators

```csharp
// First (throws if empty)
var first = await context.Products.FirstAsync();

// FirstOrDefault (returns null if empty)
var product = await context.Products
    .FirstOrDefaultAsync(p => p.Sku == "ABC-123");

// Single (throws if not exactly one)
var unique = await context.Products
    .SingleAsync(p => p.Sku == "ABC-123");

// SingleOrDefault
var maybeProduct = await context.Products
    .SingleOrDefaultAsync(p => p.Sku == "ABC-123");

// Find by primary key (uses cache first, then DB)
var byId = await context.Products.FindAsync(42);

// Any (existence check)
bool hasExpensive = await context.Products.AnyAsync(p => p.Price > 1000);

// Count
int activeCount = await context.Products.CountAsync(p => p.IsActive);
long totalCount = await context.Products.LongCountAsync();

// Aggregates
decimal maxPrice = await context.Products.MaxAsync(p => p.Price);
decimal avgPrice = await context.Products.AverageAsync(p => p.Price);
decimal totalValue = await context.Products.SumAsync(p => p.Price * p.StockQuantity);
```

### Grouping

```csharp
// Group by with aggregation
var categoryStats = await context.Products
    .GroupBy(p => p.Category.Name)
    .Select(g => new
    {
        Category = g.Key,
        Count = g.Count(),
        AveragePrice = g.Average(p => p.Price),
        MaxPrice = g.Max(p => p.Price),
        TotalStock = g.Sum(p => p.StockQuantity)
    })
    .OrderByDescending(x => x.Count)
    .ToListAsync();

// Group by multiple keys
var monthlyStats = await context.Orders
    .GroupBy(o => new { o.CreatedAt.Year, o.CreatedAt.Month })
    .Select(g => new
    {
        Year = g.Key.Year,
        Month = g.Key.Month,
        OrderCount = g.Count(),
        Revenue = g.Sum(o => o.TotalAmount)
    })
    .OrderBy(x => x.Year).ThenBy(x => x.Month)
    .ToListAsync();
```

---

## Loading Related Data

### Eager Loading with Include

```csharp
// Single level include
var orders = await context.Orders
    .Include(o => o.Customer)
    .ToListAsync();

// Multi-level include
var orders = await context.Orders
    .Include(o => o.Customer)
    .Include(o => o.Items)
        .ThenInclude(i => i.Product)
            .ThenInclude(p => p.Category)
    .ToListAsync();

// Multiple includes on same entity
var products = await context.Products
    .Include(p => p.Category)
    .Include(p => p.Reviews)
    .Include(p => p.Tags)
    .ToListAsync();
```

### Explicit Loading

```csharp
var order = await context.Orders.FirstAsync(o => o.Id == orderId);

// Load related collection
await context.Entry(order)
    .Collection(o => o.Items)
    .LoadAsync();

// Load related reference
await context.Entry(order)
    .Reference(o => o.Customer)
    .LoadAsync();

// Load with filtering
await context.Entry(order)
    .Collection(o => o.Items)
    .Query()
    .Where(i => i.Quantity > 1)
    .LoadAsync();
```

### Lazy Loading

```csharp
// 1. Install package
// dotnet add package Microsoft.EntityFrameworkCore.Proxies

// 2. Enable in configuration
builder.Services.AddDbContext<AppDbContext>(options =>
    options
        .UseLazyLoadingProxies()
        .UseSqlServer(connectionString));

// 3. Make navigation properties virtual
public class Order
{
    public int Id { get; set; }
    public virtual Customer Customer { get; set; } = null!;      // virtual required
    public virtual ICollection<OrderItem> Items { get; set; } = new List<OrderItem>();
}
```

> **Warning:** Lazy loading can cause N+1 query problems. Prefer eager loading with `Include` for known access patterns.

---

## Filtered Includes

Filter the related data loaded by `Include` (EF Core 5+).

```csharp
// Only include active items
var orders = await context.Orders
    .Include(o => o.Items.Where(i => i.IsActive))
    .ToListAsync();

// Include with ordering and limiting
var categories = await context.Categories
    .Include(c => c.Products
        .Where(p => p.IsActive)
        .OrderByDescending(p => p.Price)
        .Take(5))
    .ToListAsync();

// Multiple filtered includes
var customers = await context.Customers
    .Include(c => c.Orders.Where(o => o.Status == OrderStatus.Completed))
        .ThenInclude(o => o.Items.Where(i => i.Quantity > 0))
    .ToListAsync();
```

---

## Projection Queries

Projections select only the columns needed, avoiding loading entire entities.

### Anonymous Type Projection

```csharp
var products = await context.Products
    .Select(p => new
    {
        p.Id,
        p.Name,
        p.Price,
        CategoryName = p.Category.Name
    })
    .ToListAsync();
```

### DTO Projection

```csharp
public record ProductDto(int Id, string Name, decimal Price, string CategoryName);

var products = await context.Products
    .Select(p => new ProductDto(
        p.Id,
        p.Name,
        p.Price,
        p.Category.Name))
    .ToListAsync();
```

### Complex Projection with Nested Collections

```csharp
public record OrderSummaryDto(
    int OrderId,
    string CustomerName,
    decimal TotalAmount,
    int ItemCount,
    List<OrderItemDto> Items);

public record OrderItemDto(string ProductName, int Quantity, decimal UnitPrice);

var orders = await context.Orders
    .Select(o => new OrderSummaryDto(
        o.Id,
        o.Customer.FirstName + " " + o.Customer.LastName,
        o.TotalAmount,
        o.Items.Count,
        o.Items.Select(i => new OrderItemDto(
            i.Product.Name,
            i.Quantity,
            i.UnitPrice)).ToList()))
    .ToListAsync();
```

### Conditional Projection

```csharp
var products = await context.Products
    .Select(p => new
    {
        p.Name,
        PriceCategory = p.Price > 100 ? "Premium" : p.Price > 50 ? "Standard" : "Budget",
        HasReviews = p.Reviews.Any(),
        AverageRating = p.Reviews.Any() ? p.Reviews.Average(r => r.Rating) : 0
    })
    .ToListAsync();
```

---

## No-Tracking Queries

No-tracking queries skip the change tracker, improving performance for read-only operations.

```csharp
// Per-query
var products = await context.Products
    .AsNoTracking()
    .Where(p => p.IsActive)
    .ToListAsync();

// With identity resolution (deduplicate entities in results)
var orders = await context.Orders
    .AsNoTrackingWithIdentityResolution()
    .Include(o => o.Customer)
    .Include(o => o.Items)
    .ToListAsync();

// Context-level default
context.ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;

// Override context-level for specific query
var tracked = await context.Products
    .AsTracking()
    .FirstAsync(p => p.Id == 1);
```

| Method | Tracking | Identity Resolution | Performance |
|--------|----------|-------------------|-------------|
| Default (tracked) | Yes | Yes | Baseline |
| `AsNoTracking()` | No | No | Fastest |
| `AsNoTrackingWithIdentityResolution()` | No | Yes | Middle |

---

## Split Queries

Split queries load related collections in separate SQL queries instead of using JOINs, avoiding the "cartesian explosion" problem.

```csharp
// Single query (default) - may produce cartesian explosion with multiple collections
var orders = await context.Orders
    .Include(o => o.Items)
    .Include(o => o.Payments)
    .AsSingleQuery()
    .ToListAsync();

// Split query - separate SQL per collection
var orders = await context.Orders
    .Include(o => o.Items)
    .Include(o => o.Payments)
    .AsSplitQuery()
    .ToListAsync();
```

### Configure Default Splitting Behavior

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString, sqlOptions =>
        sqlOptions.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery)));
```

| Aspect | Single Query | Split Query |
|--------|-------------|-------------|
| SQL round-trips | 1 | N (one per Include) |
| Cartesian explosion | Possible | Avoided |
| Data consistency | Guaranteed | May differ between queries |
| Network overhead | Higher payload | Multiple smaller payloads |

---

## Raw SQL

### FromSqlRaw and FromSqlInterpolated

```csharp
// FromSqlRaw with parameters (safe from SQL injection)
var products = await context.Products
    .FromSqlRaw("SELECT * FROM Products WHERE Price > {0} AND CategoryId = {1}", 50m, categoryId)
    .ToListAsync();

// FromSqlInterpolated (preferred - uses parameterized query)
var minPrice = 50m;
var products = await context.Products
    .FromSqlInterpolated($"SELECT * FROM Products WHERE Price > {minPrice}")
    .OrderBy(p => p.Name)
    .ToListAsync();

// Composable - can chain LINQ on top
var products = await context.Products
    .FromSqlInterpolated($"SELECT * FROM Products WHERE Price > {minPrice}")
    .Where(p => p.IsActive)
    .OrderBy(p => p.Name)
    .Include(p => p.Category)
    .ToListAsync();

// Stored procedure
var products = await context.Products
    .FromSqlRaw("EXEC sp_GetProductsByCategory @p0", categoryId)
    .ToListAsync();
```

### ExecuteSqlRaw / ExecuteSqlInterpolated (Non-Query)

```csharp
// For INSERT, UPDATE, DELETE that don't return entities
var affected = await context.Database
    .ExecuteSqlInterpolatedAsync(
        $"UPDATE Products SET IsActive = 0 WHERE LastOrderDate < {cutoffDate}");

Console.WriteLine($"Deactivated {affected} products");
```

---

## SqlQuery for Unmapped Types

.NET 8+ allows querying scalar and unmapped types directly.

```csharp
// Scalar query
var productNames = await context.Database
    .SqlQuery<string>($"SELECT Name FROM Products WHERE IsActive = 1")
    .ToListAsync();

// Unmapped DTO
var stats = await context.Database
    .SqlQuery<CategoryStatsDto>(
        $"""
        SELECT
            c.Name AS CategoryName,
            COUNT(p.Id) AS ProductCount,
            AVG(p.Price) AS AveragePrice
        FROM Categories c
        LEFT JOIN Products p ON c.Id = p.CategoryId
        GROUP BY c.Name
        """)
    .OrderByDescending(s => s.ProductCount)
    .ToListAsync();

public record CategoryStatsDto(string CategoryName, int ProductCount, decimal AveragePrice);

// Composable scalar query
var ids = await context.Database
    .SqlQuery<int>($"SELECT Id FROM Products")
    .Where(id => id > 100)
    .ToListAsync();
```

---

## Bulk Operations

`ExecuteUpdate` and `ExecuteDelete` (EF Core 7+) perform bulk operations without loading entities.

### ExecuteUpdate

```csharp
// Update all matching rows in a single SQL statement
var updated = await context.Products
    .Where(p => p.Category.Name == "Electronics")
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(p => p.Price, p => p.Price * 1.10m)
        .SetProperty(p => p.UpdatedAt, DateTime.UtcNow));

Console.WriteLine($"Updated {updated} products");

// Conditional update
await context.Products
    .Where(p => p.StockQuantity == 0)
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(p => p.IsActive, false)
        .SetProperty(p => p.DeactivatedAt, DateTime.UtcNow));

// Update with computed values
await context.Products
    .Where(p => p.IsActive)
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(p => p.Slug,
            p => p.Name.ToLower().Replace(" ", "-")));
```

### ExecuteDelete

```csharp
// Delete all matching rows in a single SQL statement
var deleted = await context.Products
    .Where(p => !p.IsActive && p.CreatedAt < DateTime.UtcNow.AddYears(-1))
    .ExecuteDeleteAsync();

Console.WriteLine($"Deleted {deleted} stale products");

// Delete with join
await context.OrderItems
    .Where(oi => oi.Order.Status == OrderStatus.Cancelled)
    .ExecuteDeleteAsync();
```

> **Note:** `ExecuteUpdate` and `ExecuteDelete` bypass the change tracker. The DbContext will not be aware of these changes.

---

## Compiled Queries

Compiled queries cache the query plan, avoiding the overhead of translating LINQ to SQL on every execution.

```csharp
public class ProductQueries
{
    // Compiled synchronous query
    public static readonly Func<AppDbContext, decimal, IEnumerable<Product>> GetExpensiveProducts =
        EF.CompileQuery((AppDbContext context, decimal minPrice) =>
            context.Products
                .Where(p => p.Price > minPrice && p.IsActive)
                .OrderByDescending(p => p.Price));

    // Compiled async query (returns single result)
    public static readonly Func<AppDbContext, int, Task<Product?>> GetProductById =
        EF.CompileAsyncQuery((AppDbContext context, int id) =>
            context.Products
                .Include(p => p.Category)
                .FirstOrDefault(p => p.Id == id));

    // Compiled async query (returns collection)
    public static readonly Func<AppDbContext, string, IAsyncEnumerable<Product>> GetProductsByCategory =
        EF.CompileAsyncQuery((AppDbContext context, string categoryName) =>
            context.Products
                .Where(p => p.Category.Name == categoryName)
                .OrderBy(p => p.Name));
}

// Usage
var expensiveProducts = ProductQueries.GetExpensiveProducts(context, 100m).ToList();

var product = await ProductQueries.GetProductById(context, 42);

await foreach (var p in ProductQueries.GetProductsByCategory(context, "Electronics"))
{
    Console.WriteLine($"{p.Name}: {p.Price:C}");
}
```

---

## Global Query Filters

Global query filters automatically apply a `WHERE` clause to all queries for an entity.

```csharp
// In OnModelCreating or IEntityTypeConfiguration
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        // Soft-delete filter
        builder.HasQueryFilter(p => !p.IsDeleted);
    }
}

public class TenantEntityConfiguration : IEntityTypeConfiguration<TenantEntity>
{
    public void Configure(EntityTypeBuilder<TenantEntity> builder)
    {
        // Multi-tenant filter
        builder.HasQueryFilter(e => e.TenantId == TenantContext.CurrentTenantId);
    }
}
```

### Ignoring Global Filters

```csharp
// Include soft-deleted entities
var allProducts = await context.Products
    .IgnoreQueryFilters()
    .ToListAsync();

// Admin endpoint showing all including deleted
var adminView = await context.Products
    .IgnoreQueryFilters()
    .Select(p => new { p.Id, p.Name, p.IsDeleted })
    .ToListAsync();
```

### Multi-Tenant Filter with DI

```csharp
public class AppDbContext : DbContext
{
    private readonly ITenantService _tenantService;

    public AppDbContext(DbContextOptions<AppDbContext> options, ITenantService tenantService)
        : base(options)
    {
        _tenantService = tenantService;
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Product>()
            .HasQueryFilter(p => p.TenantId == _tenantService.TenantId);
    }
}
```

---

## Pagination Patterns

### Offset-Based Pagination

```csharp
public record PagedResult<T>(List<T> Items, int TotalCount, int Page, int PageSize)
{
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasPrevious => Page > 1;
    public bool HasNext => Page < TotalPages;
}

public async Task<PagedResult<ProductDto>> GetProductsAsync(int page, int pageSize)
{
    var query = context.Products
        .Where(p => p.IsActive)
        .OrderBy(p => p.Id);

    var totalCount = await query.CountAsync();

    var items = await query
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .Select(p => new ProductDto(p.Id, p.Name, p.Price, p.Category.Name))
        .ToListAsync();

    return new PagedResult<ProductDto>(items, totalCount, page, pageSize);
}
```

### Keyset Pagination (Seek Method) - Better Performance

```csharp
public async Task<List<ProductDto>> GetProductsAfterAsync(int lastId, int pageSize)
{
    return await context.Products
        .Where(p => p.IsActive && p.Id > lastId)
        .OrderBy(p => p.Id)
        .Take(pageSize)
        .Select(p => new ProductDto(p.Id, p.Name, p.Price, p.Category.Name))
        .ToListAsync();
}

// Multi-column keyset
public async Task<List<ProductDto>> GetProductsAfterAsync(
    decimal lastPrice, int lastId, int pageSize)
{
    return await context.Products
        .Where(p => p.Price > lastPrice ||
                    (p.Price == lastPrice && p.Id > lastId))
        .OrderBy(p => p.Price)
        .ThenBy(p => p.Id)
        .Take(pageSize)
        .Select(p => new ProductDto(p.Id, p.Name, p.Price, p.Category.Name))
        .ToListAsync();
}
```

| Aspect | Offset (Skip/Take) | Keyset |
|--------|-------------------|--------|
| Implementation | Simple | Moderate |
| Performance | Degrades with offset | Consistent |
| Random page access | Yes | No (sequential only) |
| Consistency | May skip/duplicate on changes | Stable |

---

## Dynamic Filtering and Sorting

### Building Queries Conditionally

```csharp
public async Task<List<ProductDto>> SearchProductsAsync(ProductSearchRequest request)
{
    IQueryable<Product> query = context.Products.AsNoTracking();

    // Conditional filters
    if (!string.IsNullOrWhiteSpace(request.Name))
        query = query.Where(p => p.Name.Contains(request.Name));

    if (request.MinPrice.HasValue)
        query = query.Where(p => p.Price >= request.MinPrice.Value);

    if (request.MaxPrice.HasValue)
        query = query.Where(p => p.Price <= request.MaxPrice.Value);

    if (request.CategoryId.HasValue)
        query = query.Where(p => p.CategoryId == request.CategoryId.Value);

    if (request.IsActive.HasValue)
        query = query.Where(p => p.IsActive == request.IsActive.Value);

    // Dynamic sorting
    query = request.SortBy?.ToLower() switch
    {
        "name" => request.SortDescending ? query.OrderByDescending(p => p.Name) : query.OrderBy(p => p.Name),
        "price" => request.SortDescending ? query.OrderByDescending(p => p.Price) : query.OrderBy(p => p.Price),
        "created" => request.SortDescending ? query.OrderByDescending(p => p.CreatedAt) : query.OrderBy(p => p.CreatedAt),
        _ => query.OrderBy(p => p.Id)
    };

    return await query
        .Skip((request.Page - 1) * request.PageSize)
        .Take(request.PageSize)
        .Select(p => new ProductDto(p.Id, p.Name, p.Price, p.Category.Name))
        .ToListAsync();
}

public record ProductSearchRequest(
    string? Name = null,
    decimal? MinPrice = null,
    decimal? MaxPrice = null,
    int? CategoryId = null,
    bool? IsActive = null,
    string? SortBy = null,
    bool SortDescending = false,
    int Page = 1,
    int PageSize = 20);
```

---

## Client vs Server Evaluation

EF Core translates LINQ to SQL. Expressions that cannot be translated run on the client. Starting with EF Core 3.0, client evaluation in projections throws an exception by default to prevent N+1 issues.

```csharp
// Server-evaluated (translated to SQL) - GOOD
var products = await context.Products
    .Where(p => p.Price > 100)
    .OrderBy(p => p.Name)
    .ToListAsync();

// Client-evaluated - THROWS QueryClientEvaluationWarning
// Custom method cannot be translated to SQL
var products = await context.Products
    .Where(p => IsValidProduct(p))  // ERROR: cannot translate
    .ToListAsync();

// Solution: use AsEnumerable() to switch to client evaluation explicitly
var products = await context.Products
    .Where(p => p.IsActive)         // Server-side filter
    .ToListAsync();

var filtered = products
    .Where(p => IsValidProduct(p))  // Client-side filter
    .ToList();

// Configure client evaluation warning behavior
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString)
        .ConfigureWarnings(w =>
            w.Throw(RelationalEventId.MultipleCollectionIncludeWarning)));
```

---

## Query Tags

Add SQL comments to generated queries for easier identification in database logs and monitoring tools.

```csharp
var products = await context.Products
    .TagWith("GetActiveProducts - ProductService.GetAllAsync")
    .Where(p => p.IsActive)
    .ToListAsync();
// Generated SQL: /* GetActiveProducts - ProductService.GetAllAsync */
// SELECT [p].[Id], [p].[Name], ... FROM [Products] AS [p] WHERE [p].[IsActive] = 1

// Multi-line tags
var orders = await context.Orders
    .TagWith("""
        GetRecentOrders
        Called from: OrderService.GetRecentAsync
        Caller: AdminDashboard
        """)
    .Where(o => o.CreatedAt > DateTime.UtcNow.AddDays(-30))
    .ToListAsync();

// Combine with caller info
public async Task<List<Product>> GetAsync([CallerMemberName] string caller = "")
{
    return await context.Products
        .TagWith($"Called from: {caller}")
        .AsNoTracking()
        .ToListAsync();
}
```

---

## Best Practices and Common Pitfalls

### Best Practices

| Practice | Description |
|----------|-------------|
| Use `AsNoTracking()` for read-only queries | Reduces memory and CPU overhead |
| Project to DTOs with `Select` | Load only required columns |
| Use `AsSplitQuery()` for multiple Includes | Prevent cartesian explosion |
| Use `ExecuteUpdate`/`ExecuteDelete` for bulk ops | Single SQL statement, no entity loading |
| Add `TagWith()` in production queries | Enables query identification in DB monitoring |
| Use keyset pagination for large datasets | Consistent performance regardless of page number |
| Use compiled queries for hot paths | Eliminate LINQ-to-SQL translation overhead |
| Apply global query filters for soft-delete | Ensures deleted records are never accidentally returned |
| Use `FindAsync` for primary key lookups | Checks change tracker before hitting the DB |

### Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| N+1 queries (lazy loading) | One query per related entity | Use `Include` for eager loading |
| Loading full entities for display | Excess data over the wire | Use `Select` projections |
| `ToList()` before `Where()` | Loads all rows then filters in memory | Filter before materializing |
| Cartesian explosion with multiple Includes | Exponential row count | Use `AsSplitQuery()` |
| Ignoring `CancellationToken` | Queries run to completion on cancel | Pass tokens through from controllers |
| Calling `Count()` + `ToList()` separately | Two database round-trips | Use a single query or `PagedResult` pattern |
| `Contains` with large lists | Generates huge `IN (...)` clause | Batch into chunks or use temp tables |
| Mixing tracked and no-tracking results | Unexpected behavior | Be consistent within a unit of work |
| Client evaluation in `Where` | Throws or loads all rows | Ensure expressions are translatable |
| Not indexing filtered columns | Full table scans | Add indexes for frequent `Where` columns |
