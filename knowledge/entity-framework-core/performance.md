# Entity Framework Core Performance

> Official Documentation: https://learn.microsoft.com/ef/core/performance/

## Table of Contents

1. [AsNoTracking for Read-Only Queries](#asnotracking-for-read-only-queries)
2. [Split Queries vs Single Query](#split-queries-vs-single-query)
3. [Projection Instead of Full Entities](#projection-instead-of-full-entities)
4. [Bulk Operations](#bulk-operations)
5. [Compiled Queries](#compiled-queries)
6. [DbContext Pooling](#dbcontext-pooling)
7. [Efficient Pagination](#efficient-pagination)
8. [Index Configuration](#index-configuration)
9. [Query Tags for Monitoring](#query-tags-for-monitoring)
10. [Lazy Loading Pitfalls](#lazy-loading-pitfalls)
11. [Eager Loading Strategies](#eager-loading-strategies)
12. [Batch Size Configuration](#batch-size-configuration)
13. [Connection Resiliency](#connection-resiliency)
14. [Logging and Diagnostics](#logging-and-diagnostics)
15. [Performance Profiling Tools](#performance-profiling-tools)
16. [Best Practices and Common Pitfalls](#best-practices-and-common-pitfalls)

---

## AsNoTracking for Read-Only Queries

When you don't need to update entities, `AsNoTracking()` eliminates change tracking overhead and reduces memory usage.

```csharp
// Per-query no-tracking
var products = await context.Products
    .AsNoTracking()
    .Where(p => p.IsActive)
    .ToListAsync();

// With identity resolution (deduplicates entities in the result set)
var orders = await context.Orders
    .AsNoTrackingWithIdentityResolution()
    .Include(o => o.Customer)
    .Include(o => o.Items)
    .ToListAsync();

// Context-level default
context.ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;
```

### Performance Comparison

| Scenario | Tracked | AsNoTracking | Improvement |
|----------|---------|-------------|-------------|
| 1,000 entities | ~5ms | ~2ms | ~60% faster |
| 10,000 entities | ~50ms | ~15ms | ~70% faster |
| 100,000 entities | ~500ms | ~80ms | ~84% faster |
| Memory (10K entities) | ~40MB | ~15MB | ~62% less |

> Actual numbers vary by entity complexity and hardware. Always benchmark your specific scenario.

---

## Split Queries vs Single Query

### Cartesian Explosion Problem

```csharp
// BAD: Cartesian explosion with multiple collection includes
// If Order has 10 Items and 5 Payments, each row is duplicated: 10 * 5 = 50 rows
var orders = await context.Orders
    .Include(o => o.Items)       // Collection
    .Include(o => o.Payments)    // Collection
    .AsSingleQuery()             // Default behavior
    .ToListAsync();
```

### Split Query Solution

```csharp
// GOOD: Separate SQL query per collection
var orders = await context.Orders
    .Include(o => o.Items)
    .Include(o => o.Payments)
    .AsSplitQuery()
    .ToListAsync();
// Generates 3 separate queries:
// 1. SELECT ... FROM Orders
// 2. SELECT ... FROM OrderItems WHERE OrderId IN (...)
// 3. SELECT ... FROM Payments WHERE OrderId IN (...)
```

### Configure Default Globally

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString, sqlOptions =>
        sqlOptions.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery)));
```

### When to Use Each

| Scenario | Recommended | Reason |
|----------|------------|--------|
| Single collection Include | Single query | Minimal overhead |
| Multiple collection Includes | Split query | Avoid cartesian explosion |
| Consistency critical | Single query | Atomic snapshot |
| Large result sets | Split query | Less data transfer |
| Reference-only Includes | Single query | No duplication risk |

---

## Projection Instead of Full Entities

Loading only the columns you need dramatically reduces data transfer and memory usage.

```csharp
// BAD: Loading full entities with all columns
var products = await context.Products
    .Include(p => p.Category)
    .Include(p => p.Reviews)
    .ToListAsync();

// GOOD: Project to DTO with only needed columns
var products = await context.Products
    .Select(p => new ProductListDto(
        p.Id,
        p.Name,
        p.Price,
        p.Category.Name,
        p.Reviews.Count(),
        p.Reviews.Any() ? p.Reviews.Average(r => r.Rating) : 0))
    .ToListAsync();

public record ProductListDto(
    int Id,
    string Name,
    decimal Price,
    string CategoryName,
    int ReviewCount,
    double AverageRating);
```

### Projection Avoids N+1

```csharp
// Projection with nested collections (no N+1)
var orders = await context.Orders
    .Select(o => new OrderDto(
        o.Id,
        o.Customer.Name,
        o.TotalAmount,
        o.Items.Select(i => new OrderItemDto(
            i.Product.Name,
            i.Quantity,
            i.UnitPrice)).ToList()))
    .ToListAsync();
```

---

## Bulk Operations

### ExecuteUpdate and ExecuteDelete (EF Core 7+)

These execute a single SQL statement without loading entities into memory.

```csharp
// BAD: Load all entities, modify, save (N+1 round trips)
var products = await context.Products
    .Where(p => p.Category.Name == "Electronics")
    .ToListAsync();
foreach (var p in products)
    p.Price *= 1.10m;
await context.SaveChangesAsync();

// GOOD: Single SQL UPDATE statement
await context.Products
    .Where(p => p.Category.Name == "Electronics")
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(p => p.Price, p => p.Price * 1.10m)
        .SetProperty(p => p.UpdatedAt, DateTime.UtcNow));

// GOOD: Single SQL DELETE statement
await context.Products
    .Where(p => !p.IsActive && p.CreatedAt < DateTime.UtcNow.AddYears(-2))
    .ExecuteDeleteAsync();
```

### Performance Comparison

| Operation | Traditional (10K rows) | ExecuteUpdate/Delete | Improvement |
|-----------|----------------------|---------------------|-------------|
| Update | ~5s (load + modify + save) | ~50ms (single SQL) | ~100x faster |
| Delete | ~3s (load + remove + save) | ~30ms (single SQL) | ~100x faster |
| Memory | Loads all entities | Zero entity allocation | ~0 MB vs ~50 MB |

### Batch Insert with AddRange

```csharp
// EF Core batches inserts automatically
var products = Enumerable.Range(1, 10000)
    .Select(i => new Product { Name = $"Product {i}", Price = i * 0.99m })
    .ToList();

context.Products.AddRange(products);
await context.SaveChangesAsync(); // Batched INSERT statements
```

---

## Compiled Queries

Compiled queries cache the LINQ-to-SQL translation, eliminating compilation overhead on repeated calls.

```csharp
public static class ProductQueries
{
    // Compiled query returning a collection
    public static readonly Func<AppDbContext, decimal, IAsyncEnumerable<Product>>
        GetByMinPrice = EF.CompileAsyncQuery(
            (AppDbContext ctx, decimal minPrice) =>
                ctx.Products
                    .Where(p => p.Price >= minPrice && p.IsActive)
                    .OrderBy(p => p.Name));

    // Compiled query returning a single entity
    public static readonly Func<AppDbContext, int, Task<Product?>>
        GetById = EF.CompileAsyncQuery(
            (AppDbContext ctx, int id) =>
                ctx.Products
                    .Include(p => p.Category)
                    .FirstOrDefault(p => p.Id == id));

    // Compiled query returning a scalar
    public static readonly Func<AppDbContext, string, Task<int>>
        CountByCategory = EF.CompileAsyncQuery(
            (AppDbContext ctx, string category) =>
                ctx.Products.Count(p => p.Category.Name == category));
}

// Usage
await foreach (var product in ProductQueries.GetByMinPrice(context, 50m))
{
    Console.WriteLine($"{product.Name}: {product.Price:C}");
}

var product = await ProductQueries.GetById(context, 42);
var count = await ProductQueries.CountByCategory(context, "Electronics");
```

### When to Use Compiled Queries

| Scenario | Use Compiled? | Reason |
|----------|--------------|--------|
| Hot path (called 100s/sec) | Yes | Eliminates compilation overhead |
| Complex LINQ query | Yes | More compilation work to skip |
| Simple query (e.g., FindAsync) | No | Already efficient |
| Dynamic/conditional queries | No | Cannot compile dynamic LINQ |

---

## DbContext Pooling

Pooling reuses `DbContext` instances, reducing allocation overhead and GC pressure.

```csharp
// Standard registration (new instance per request)
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));

// Pooled registration (reuses instances)
builder.Services.AddDbContextPool<AppDbContext>(options =>
    options.UseSqlServer(connectionString),
    poolSize: 256);

// Pooled factory (for Blazor, background services)
builder.Services.AddPooledDbContextFactory<AppDbContext>(options =>
    options.UseSqlServer(connectionString),
    poolSize: 256);
```

### Pooling Performance Impact

| Metric | Without Pooling | With Pooling | Improvement |
|--------|----------------|-------------|-------------|
| Allocations per request | ~10 KB | ~0.5 KB | ~95% fewer |
| Request throughput (RPS) | 40,000 | 48,000 | ~20% more |
| P99 latency | 12ms | 9ms | ~25% faster |

> Numbers from benchmark scenarios. Actual impact depends on your workload.

---

## Efficient Pagination

### Offset Pagination (Simple but Slow at High Offsets)

```csharp
// Performance degrades as page number increases
var page = await context.Products
    .OrderBy(p => p.Id)
    .Skip((pageNumber - 1) * pageSize)
    .Take(pageSize)
    .AsNoTracking()
    .ToListAsync();
```

### Keyset Pagination (Consistent Performance)

```csharp
// Always scans from index position - constant performance
public async Task<List<ProductDto>> GetProductsAfterAsync(int lastId, int pageSize)
{
    return await context.Products
        .Where(p => p.Id > lastId)
        .OrderBy(p => p.Id)
        .Take(pageSize)
        .AsNoTracking()
        .Select(p => new ProductDto(p.Id, p.Name, p.Price))
        .ToListAsync();
}

// Multi-column keyset (for sorting by non-unique columns)
public async Task<List<ProductDto>> GetProductsAsync(
    decimal? lastPrice, int? lastId, int pageSize)
{
    var query = context.Products.AsNoTracking();

    if (lastPrice.HasValue && lastId.HasValue)
    {
        query = query.Where(p =>
            p.Price > lastPrice.Value ||
            (p.Price == lastPrice.Value && p.Id > lastId.Value));
    }

    return await query
        .OrderBy(p => p.Price)
        .ThenBy(p => p.Id)
        .Take(pageSize)
        .Select(p => new ProductDto(p.Id, p.Name, p.Price))
        .ToListAsync();
}
```

### Performance Comparison

| Page Number | Offset (Skip/Take) | Keyset | Difference |
|-------------|-------------------|--------|------------|
| 1 | 1ms | 1ms | Same |
| 100 | 5ms | 1ms | 5x faster |
| 10,000 | 200ms | 1ms | 200x faster |
| 1,000,000 | 15s+ | 1ms | 15,000x faster |

---

## Index Configuration

Proper indexes are the single most impactful performance optimization.

```csharp
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        // Simple index
        builder.HasIndex(p => p.Sku)
            .IsUnique()
            .HasDatabaseName("IX_Products_Sku");

        // Composite index
        builder.HasIndex(p => new { p.CategoryId, p.Price })
            .HasDatabaseName("IX_Products_Category_Price");

        // Filtered index (SQL Server)
        builder.HasIndex(p => p.Name)
            .HasFilter("[IsActive] = 1")
            .HasDatabaseName("IX_Products_Name_Active");

        // Include columns (covering index)
        builder.HasIndex(p => p.CategoryId)
            .IncludeProperties(p => new { p.Name, p.Price })
            .HasDatabaseName("IX_Products_CategoryId_Inc");

        // Descending index
        builder.HasIndex(p => p.CreatedAt)
            .IsDescending()
            .HasDatabaseName("IX_Products_CreatedAt_Desc");
    }
}
```

### Index Strategy Guidelines

| Query Pattern | Index Strategy |
|--------------|---------------|
| `WHERE Column = value` | Single-column index on Column |
| `WHERE A = x AND B = y` | Composite index on (A, B) |
| `WHERE A = x ORDER BY B` | Composite index on (A, B) |
| `SELECT A, B WHERE C = x` | Index on C, INCLUDE (A, B) |
| `WHERE IsActive = 1` | Filtered index where IsActive = 1 |
| `ORDER BY CreatedAt DESC` | Descending index on CreatedAt |

### Missing Index Detection

```csharp
// Enable sensitive data logging to see parameter values
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString)
        .EnableSensitiveDataLogging()
        .LogTo(Console.WriteLine, LogLevel.Information));
```

---

## Query Tags for Monitoring

```csharp
// Tag queries for identification in SQL Server profiler / Azure Monitor
var products = await context.Products
    .TagWith("ProductService.GetActiveProducts")
    .TagWith($"RequestId: {httpContext.TraceIdentifier}")
    .AsNoTracking()
    .Where(p => p.IsActive)
    .ToListAsync();

// Output in SQL:
// -- ProductService.GetActiveProducts
// -- RequestId: abc-123
// SELECT [p].[Id], [p].[Name], ...
```

---

## Lazy Loading Pitfalls

### N+1 Problem

```csharp
// BAD: N+1 queries (1 for orders + N for each customer)
var orders = await context.Orders.ToListAsync();
foreach (var order in orders)
{
    Console.WriteLine(order.Customer.Name); // Each access triggers a query!
}

// GOOD: Eager loading
var orders = await context.Orders
    .Include(o => o.Customer)
    .ToListAsync();
foreach (var order in orders)
{
    Console.WriteLine(order.Customer.Name); // Already loaded
}

// GOOD: Projection (no navigation needed)
var orderSummaries = await context.Orders
    .Select(o => new { o.Id, CustomerName = o.Customer.Name })
    .ToListAsync();
```

### Detecting N+1 in Development

```csharp
// Throw on multiple lazy loads (detect N+1 in dev)
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString)
        .ConfigureWarnings(w =>
            w.Throw(CoreEventId.NavigationBaseIncludeIgnored)
             .Throw(CoreEventId.LazyLoadOnDisposedContextWarning)));
```

---

## Eager Loading Strategies

### Include Only What You Need

```csharp
// BAD: Including everything
var order = await context.Orders
    .Include(o => o.Customer)
        .ThenInclude(c => c.Orders)       // All customer orders?!
            .ThenInclude(o => o.Items)     // Items for ALL orders?!
    .Include(o => o.Items)
        .ThenInclude(i => i.Product)
            .ThenInclude(p => p.Category)
            .ThenInclude(p => p.Reviews)   // All reviews for each product?!
    .FirstAsync(o => o.Id == orderId);

// GOOD: Include only what this endpoint needs
var order = await context.Orders
    .Include(o => o.Items)
        .ThenInclude(i => i.Product)
    .AsNoTracking()
    .FirstAsync(o => o.Id == orderId);
```

### Filtered Includes

```csharp
// Load only relevant related data
var customer = await context.Customers
    .Include(c => c.Orders
        .Where(o => o.Status == OrderStatus.Active)
        .OrderByDescending(o => o.CreatedAt)
        .Take(5))
    .AsNoTracking()
    .FirstAsync(c => c.Id == customerId);
```

---

## Batch Size Configuration

EF Core automatically batches INSERT, UPDATE, and DELETE commands.

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString, sqlOptions =>
    {
        // SQL Server default batch size: 42
        sqlOptions.MaxBatchSize(100);

        // Minimum commands to batch (default: 4 for SQL Server)
        sqlOptions.MinBatchSize(2);
    }));
```

| Provider | Default Max Batch Size | Recommended |
|----------|----------------------|-------------|
| SQL Server | 42 | 42-100 |
| PostgreSQL | 100 | 100-200 |
| SQLite | 1 (no batching) | N/A |

---

## Connection Resiliency

### SQL Server Retry on Transient Failures

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString, sqlOptions =>
    {
        sqlOptions.EnableRetryOnFailure(
            maxRetryCount: 5,
            maxRetryDelay: TimeSpan.FromSeconds(30),
            errorNumbersToAdd: null);
    }));
```

### Custom Retry Strategy

```csharp
public class CustomRetryStrategy : SqlServerRetryingExecutionStrategy
{
    public CustomRetryStrategy(DbContext context)
        : base(context, maxRetryCount: 5, maxRetryDelay: TimeSpan.FromSeconds(30),
            errorNumbersToAdd: new[] { 4060 }) // Add custom error numbers
    {
    }

    protected override bool ShouldRetryOn(Exception exception)
    {
        // Add custom retry logic
        if (exception is TimeoutException)
            return true;

        return base.ShouldRetryOn(exception);
    }
}

// Registration
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString, sqlOptions =>
        sqlOptions.ExecutionStrategy(deps => new CustomRetryStrategy(deps.CurrentContext.Context))));
```

### Manual Execution Strategy

```csharp
// For transactions with retry
var strategy = context.Database.CreateExecutionStrategy();

await strategy.ExecuteAsync(async () =>
{
    await using var transaction = await context.Database.BeginTransactionAsync();

    context.Products.Add(new Product { Name = "Widget", Price = 9.99m });
    await context.SaveChangesAsync();

    context.Orders.Add(new Order { TotalAmount = 9.99m });
    await context.SaveChangesAsync();

    await transaction.CommitAsync();
});
```

---

## Logging and Diagnostics

### Simple Logging

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString)
        .LogTo(Console.WriteLine, LogLevel.Information)
        .EnableSensitiveDataLogging()  // Shows parameter values (dev only!)
        .EnableDetailedErrors());       // Detailed error messages (dev only!)
```

### ILogger Integration

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString)
        .LogTo(
            filter: (eventId, level) => eventId.Id == CoreEventId.ContextInitialized.Id
                || level == LogLevel.Warning,
            logger: (eventData) =>
            {
                // Custom logging
            }));
```

### Diagnostic Listener

```csharp
// Monitor all EF Core events
public class EfCoreDiagnosticListener : IObserver<DiagnosticListener>
{
    public void OnNext(DiagnosticListener listener)
    {
        if (listener.Name == DbLoggerCategory.Name)
        {
            listener.Subscribe(new EfCoreEventObserver());
        }
    }

    public void OnError(Exception error) { }
    public void OnCompleted() { }
}

public class EfCoreEventObserver : IObserver<KeyValuePair<string, object?>>
{
    public void OnNext(KeyValuePair<string, object?> pair)
    {
        if (pair.Key == RelationalEventId.CommandExecuted.Name
            && pair.Value is CommandExecutedEventData data)
        {
            if (data.Duration.TotalMilliseconds > 100)
            {
                Console.WriteLine($"Slow query ({data.Duration.TotalMilliseconds}ms): {data.Command.CommandText}");
            }
        }
    }

    public void OnError(Exception error) { }
    public void OnCompleted() { }
}
```

---

## Performance Profiling Tools

| Tool | Type | Description |
|------|------|-------------|
| **MiniProfiler** | In-app | Inline SQL profiling in web UI |
| **EF Core Logging** | Built-in | Log generated SQL statements |
| **SQL Server Profiler** | Database | Trace all database queries |
| **Azure Data Studio** | IDE | Query plan analysis |
| **dotnet-counters** | CLI | Runtime performance counters |
| **Application Insights** | Cloud | Dependency tracking for SQL |
| **BenchmarkDotNet** | Library | Micro-benchmark EF operations |

### MiniProfiler Setup

```csharp
// Install: dotnet add package MiniProfiler.EntityFrameworkCore

builder.Services.AddMiniProfiler(options =>
{
    options.RouteBasePath = "/profiler";
}).AddEntityFramework();

app.UseMiniProfiler();
```

### BenchmarkDotNet Example

```csharp
[MemoryDiagnoser]
public class QueryBenchmarks
{
    private AppDbContext _context = null!;

    [GlobalSetup]
    public void Setup()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlServer("Server=localhost;Database=BenchmarkDb;...")
            .Options;
        _context = new AppDbContext(options);
    }

    [Benchmark(Baseline = true)]
    public async Task<List<Product>> FullEntity()
    {
        return await _context.Products.AsNoTracking().ToListAsync();
    }

    [Benchmark]
    public async Task<List<ProductDto>> Projection()
    {
        return await _context.Products
            .Select(p => new ProductDto(p.Id, p.Name, p.Price))
            .ToListAsync();
    }

    [GlobalCleanup]
    public void Cleanup() => _context.Dispose();
}
```

---

## Best Practices and Common Pitfalls

### Performance Checklist

| Category | Action | Impact |
|----------|--------|--------|
| Queries | Use `AsNoTracking()` for reads | High |
| Queries | Project to DTOs with `Select` | High |
| Queries | Use `AsSplitQuery()` for multiple collections | High |
| Bulk | Use `ExecuteUpdate`/`ExecuteDelete` | Very High |
| Indexes | Add indexes for `WHERE` and `ORDER BY` columns | Very High |
| Indexes | Use covering indexes (`IncludeProperties`) | High |
| Loading | Prefer eager loading over lazy loading | High |
| Pagination | Use keyset pagination for large datasets | High |
| Context | Enable DbContext pooling | Medium |
| Context | Use `DbContextFactory` for parallel operations | Medium |
| Queries | Use compiled queries for hot paths | Medium |
| Batching | Configure `MaxBatchSize` for bulk inserts | Medium |
| Monitoring | Add `TagWith()` for production queries | Low (observability) |

### Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| N+1 queries | Lazy loading triggers a query per entity | Use `Include` or projection |
| Tracking read-only data | Wasted memory and CPU | Add `AsNoTracking()` |
| Loading all columns | Excess data transferred | Use `Select` projection |
| Cartesian explosion | JOINs multiply row counts | Use `AsSplitQuery()` |
| Missing indexes | Full table scans | Add targeted indexes |
| Offset pagination at high pages | Scans and discards rows | Switch to keyset pagination |
| `SaveChanges` in a loop | N round-trips to database | Batch changes, call once |
| `ToList()` before filtering | Loads entire table to memory | Filter with `Where` before `ToList` |
| No retry strategy | Transient failures crash the app | Enable `EnableRetryOnFailure` |
| No query timeout | Runaway queries block connections | Set `CommandTimeout` |
| Not monitoring slow queries | Silent performance degradation | Use logging, interceptors, or profilers |
| Premature optimization | Overcomplicating code | Profile first, optimize bottlenecks |
