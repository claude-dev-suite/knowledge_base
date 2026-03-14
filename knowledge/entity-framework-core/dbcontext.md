# Entity Framework Core DbContext

> Official Documentation: https://learn.microsoft.com/ef/core/dbcontext-configuration/

## Table of Contents

1. [DbContext Class Design](#dbcontext-class-design)
2. [DbSet Properties](#dbset-properties)
3. [OnModelCreating (Fluent API)](#onmodelcreating-fluent-api)
4. [OnConfiguring vs External Configuration](#onconfiguring-vs-external-configuration)
5. [Connection Strings](#connection-strings)
6. [DbContext Pooling](#dbcontext-pooling)
7. [DbContext Factory](#dbcontext-factory)
8. [DbContext Lifetime and Scope](#dbcontext-lifetime-and-scope)
9. [Multiple DbContexts](#multiple-dbcontexts)
10. [Design-Time DbContext Creation](#design-time-dbcontext-creation)
11. [SaveChanges and SaveChangesAsync](#savechanges-and-savechangesasync)
12. [Change Tracking](#change-tracking)
13. [Entity States](#entity-states)
14. [Interceptors](#interceptors)
15. [Best Practices and Common Pitfalls](#best-practices-and-common-pitfalls)

---

## DbContext Class Design

The `DbContext` is the primary class for interacting with the database. It represents a session with the database and provides APIs for querying, saving, and configuring entities.

```csharp
using Microsoft.EntityFrameworkCore;

namespace MyApp.Data;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options)
    {
    }

    public DbSet<Product> Products => Set<Product>();
    public DbSet<Category> Categories => Set<Category>();
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();
    public DbSet<Customer> Customers => Set<Customer>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}
```

### Minimal DbContext with File-Scoped Namespace

```csharp
namespace MyApp.Data;

public class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<Product> Products => Set<Product>();
    public DbSet<Category> Categories => Set<Category>();
}
```

---

## DbSet Properties

`DbSet<T>` represents a collection of entities that can be queried and saved. Each DbSet maps to a database table.

```csharp
public class AppDbContext : DbContext
{
    // Recommended: expression-bodied property (no null warning)
    public DbSet<Product> Products => Set<Product>();

    // Alternative: auto-property with null-forgiving operator
    public DbSet<Category> Categories { get; set; } = null!;

    // Auto-property (may produce nullable warning)
    public DbSet<Order> Orders { get; set; }
}
```

### Querying a DbSet

```csharp
// DbSet implements IQueryable<T>
var activeProducts = await context.Products
    .Where(p => p.IsActive)
    .OrderBy(p => p.Name)
    .ToListAsync();

// DbSet also implements IAsyncEnumerable<T>
await foreach (var product in context.Products)
{
    Console.WriteLine(product.Name);
}
```

### Entities Without a DbSet

You can query entities that have no `DbSet` property by using `Set<T>()`:

```csharp
var auditLogs = await context.Set<AuditLog>()
    .Where(a => a.Timestamp > DateTime.UtcNow.AddDays(-7))
    .ToListAsync();
```

---

## OnModelCreating (Fluent API)

The `OnModelCreating` method configures the model using the Fluent API, which takes precedence over data annotations.

### Inline Configuration

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Product>(entity =>
    {
        entity.ToTable("Products", "inventory");
        entity.HasKey(e => e.Id);
        entity.Property(e => e.Name)
            .HasMaxLength(200)
            .IsRequired();
        entity.Property(e => e.Price)
            .HasPrecision(18, 2);
        entity.Property(e => e.Sku)
            .HasMaxLength(50)
            .IsUnicode(false);
        entity.HasIndex(e => e.Sku)
            .IsUnique();
        entity.HasOne(e => e.Category)
            .WithMany(c => c.Products)
            .HasForeignKey(e => e.CategoryId)
            .OnDelete(DeleteBehavior.Restrict);
    });
}
```

### Separate Configuration Classes (IEntityTypeConfiguration)

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace MyApp.Data.Configurations;

public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.ToTable("Products", "inventory");
        builder.HasKey(e => e.Id);

        builder.Property(e => e.Name)
            .HasMaxLength(200)
            .IsRequired();

        builder.Property(e => e.Price)
            .HasPrecision(18, 2);

        builder.Property(e => e.CreatedAt)
            .HasDefaultValueSql("GETUTCDATE()");

        builder.HasIndex(e => e.Sku)
            .IsUnique()
            .HasDatabaseName("IX_Products_Sku");

        builder.HasQueryFilter(e => !e.IsDeleted);
    }
}

public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("Orders");
        builder.HasKey(e => e.Id);

        builder.Property(e => e.TotalAmount)
            .HasPrecision(18, 2);

        builder.HasMany(e => e.Items)
            .WithOne(i => i.Order)
            .HasForeignKey(i => i.OrderId)
            .OnDelete(DeleteBehavior.Cascade);
    }
}
```

### Auto-Apply All Configurations

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // Discovers and applies all IEntityTypeConfiguration<T> in the assembly
    modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
}
```

---

## OnConfiguring vs External Configuration

### OnConfiguring (Embedded Connection)

```csharp
public class AppDbContext : DbContext
{
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        if (!optionsBuilder.IsConfigured)
        {
            optionsBuilder.UseSqlServer(
                "Server=localhost;Database=MyApp;Trusted_Connection=true;TrustServerCertificate=true;");
        }
    }
}
```

### External Configuration via Dependency Injection (Preferred)

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
```

```json
// appsettings.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyApp;Trusted_Connection=true;TrustServerCertificate=true;"
  }
}
```

---

## Connection Strings

### SQL Server

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString, sqlOptions =>
    {
        sqlOptions.MigrationsAssembly("MyApp.Migrations");
        sqlOptions.EnableRetryOnFailure(
            maxRetryCount: 5,
            maxRetryDelay: TimeSpan.FromSeconds(30),
            errorNumbersToAdd: null);
        sqlOptions.CommandTimeout(60);
        sqlOptions.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery);
    }));
```

### PostgreSQL (Npgsql)

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(connectionString, npgsqlOptions =>
    {
        npgsqlOptions.MigrationsAssembly("MyApp.Migrations");
        npgsqlOptions.EnableRetryOnFailure(maxRetryCount: 3);
        npgsqlOptions.MapEnum<OrderStatus>("order_status");
    }));
```

### SQLite

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite("Data Source=myapp.db", sqliteOptions =>
    {
        sqliteOptions.MigrationsAssembly("MyApp.Migrations");
    }));
```

### In-Memory (Testing Only)

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseInMemoryDatabase("TestDb"));
```

---

## DbContext Pooling

DbContext pooling reuses `DbContext` instances from a pool instead of creating new ones on each request, reducing allocation overhead.

```csharp
// Register with pooling (default pool size: 1024)
builder.Services.AddDbContextPool<AppDbContext>(options =>
    options.UseSqlServer(connectionString),
    poolSize: 256);
```

| Feature | AddDbContext | AddDbContextPool |
|---------|-------------|------------------|
| Instance reuse | No | Yes |
| Constructor called | Every request | Once per pool slot |
| OnConfiguring called | Every request | Once per pool slot |
| Supports DI in DbContext | Yes | Limited |
| Performance | Baseline | ~20% faster |
| Pool size | N/A | Default 1024 |

### Pooling Constraints

- Cannot store per-request state in the DbContext
- OnConfiguring is only called once (not per request)
- Constructor services from DI are not refreshed per request

```csharp
// This pattern does NOT work with pooling - currentUser is captured once
public class AppDbContext : DbContext
{
    private readonly ICurrentUserService _currentUser; // DO NOT inject with pooling

    public AppDbContext(
        DbContextOptions<AppDbContext> options,
        ICurrentUserService currentUser) : base(options)
    {
        _currentUser = currentUser;
    }
}
```

---

## DbContext Factory

`IDbContextFactory<T>` creates `DbContext` instances on demand, useful for Blazor, background services, and parallel operations.

```csharp
// Registration
builder.Services.AddDbContextFactory<AppDbContext>(options =>
    options.UseSqlServer(connectionString));

// Also available with pooling
builder.Services.AddPooledDbContextFactory<AppDbContext>(options =>
    options.UseSqlServer(connectionString));
```

### Usage in Services

```csharp
public class ProductService
{
    private readonly IDbContextFactory<AppDbContext> _factory;

    public ProductService(IDbContextFactory<AppDbContext> factory)
    {
        _factory = factory;
    }

    public async Task<List<Product>> GetAllAsync()
    {
        await using var context = await _factory.CreateDbContextAsync();
        return await context.Products.ToListAsync();
    }

    public async Task ProcessInParallelAsync(IEnumerable<int> productIds)
    {
        var tasks = productIds.Select(async id =>
        {
            await using var context = await _factory.CreateDbContextAsync();
            var product = await context.Products.FindAsync(id);
            if (product is not null)
            {
                product.LastProcessed = DateTime.UtcNow;
                await context.SaveChangesAsync();
            }
        });

        await Task.WhenAll(tasks);
    }
}
```

### Usage in Blazor Components

```csharp
@inject IDbContextFactory<AppDbContext> DbFactory

@code {
    private List<Product>? products;

    protected override async Task OnInitializedAsync()
    {
        await using var context = await DbFactory.CreateDbContextAsync();
        products = await context.Products.ToListAsync();
    }
}
```

---

## DbContext Lifetime and Scope

| Scenario | Recommended Lifetime | Registration |
|----------|---------------------|--------------|
| Web API / MVC | Scoped (per request) | AddDbContext |
| Blazor Server | Factory (per operation) | AddDbContextFactory |
| Background service | Factory (per operation) | AddDbContextFactory |
| Console app | Transient or manual | New or Factory |
| Parallel operations | Factory | AddDbContextFactory |

```csharp
// Scoped (default for web apps) - one DbContext per HTTP request
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));

// Background service using factory
public class OrderProcessingService : BackgroundService
{
    private readonly IDbContextFactory<AppDbContext> _factory;

    public OrderProcessingService(IDbContextFactory<AppDbContext> factory)
    {
        _factory = factory;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await using var context = await _factory.CreateDbContextAsync(stoppingToken);

            var pendingOrders = await context.Orders
                .Where(o => o.Status == OrderStatus.Pending)
                .ToListAsync(stoppingToken);

            foreach (var order in pendingOrders)
            {
                order.Status = OrderStatus.Processing;
            }

            await context.SaveChangesAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken);
        }
    }
}
```

---

## Multiple DbContexts

```csharp
public class CatalogDbContext : DbContext
{
    public CatalogDbContext(DbContextOptions<CatalogDbContext> options) : base(options) { }
    public DbSet<Product> Products => Set<Product>();
    public DbSet<Category> Categories => Set<Category>();
}

public class OrderDbContext : DbContext
{
    public OrderDbContext(DbContextOptions<OrderDbContext> options) : base(options) { }
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();
}

// Registration
builder.Services.AddDbContext<CatalogDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("CatalogDb")));

builder.Services.AddDbContext<OrderDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("OrderDb")));
```

---

## Design-Time DbContext Creation

`IDesignTimeDbContextFactory<T>` allows EF Core tools (`dotnet ef`) to create a DbContext at design time for migrations.

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Design;

namespace MyApp.Data;

public class AppDbContextFactory : IDesignTimeDbContextFactory<AppDbContext>
{
    public AppDbContext CreateDbContext(string[] args)
    {
        var configuration = new ConfigurationBuilder()
            .SetBasePath(Directory.GetCurrentDirectory())
            .AddJsonFile("appsettings.json")
            .AddJsonFile("appsettings.Development.json", optional: true)
            .Build();

        var optionsBuilder = new DbContextOptionsBuilder<AppDbContext>();
        optionsBuilder.UseSqlServer(
            configuration.GetConnectionString("DefaultConnection"));

        return new AppDbContext(optionsBuilder.Options);
    }
}
```

---

## SaveChanges and SaveChangesAsync

```csharp
// Synchronous (avoid in async contexts)
context.SaveChanges();

// Asynchronous (preferred)
await context.SaveChangesAsync();

// With CancellationToken
await context.SaveChangesAsync(cancellationToken);
```

### Overriding SaveChanges for Auditing

```csharp
public class AppDbContext : DbContext
{
    public override int SaveChanges(bool acceptAllChangesOnSuccess)
    {
        ApplyAuditInfo();
        return base.SaveChanges(acceptAllChangesOnSuccess);
    }

    public override async Task<int> SaveChangesAsync(
        bool acceptAllChangesOnSuccess,
        CancellationToken cancellationToken = default)
    {
        ApplyAuditInfo();
        return await base.SaveChangesAsync(acceptAllChangesOnSuccess, cancellationToken);
    }

    private void ApplyAuditInfo()
    {
        var entries = ChangeTracker.Entries<IAuditable>();

        foreach (var entry in entries)
        {
            switch (entry.State)
            {
                case EntityState.Added:
                    entry.Entity.CreatedAt = DateTime.UtcNow;
                    entry.Entity.UpdatedAt = DateTime.UtcNow;
                    break;
                case EntityState.Modified:
                    entry.Entity.UpdatedAt = DateTime.UtcNow;
                    break;
            }
        }
    }
}

public interface IAuditable
{
    DateTime CreatedAt { get; set; }
    DateTime UpdatedAt { get; set; }
}
```

### Transactions

```csharp
// Implicit transaction (SaveChanges wraps all changes in one transaction)
context.Products.Add(new Product { Name = "Widget" });
context.Orders.Add(new Order { CustomerId = 1 });
await context.SaveChangesAsync(); // Both or neither

// Explicit transaction
await using var transaction = await context.Database.BeginTransactionAsync();
try
{
    context.Products.Add(new Product { Name = "Widget", Price = 9.99m });
    await context.SaveChangesAsync();

    context.Orders.Add(new Order { CustomerId = 1, TotalAmount = 9.99m });
    await context.SaveChangesAsync();

    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

---

## Change Tracking

EF Core tracks changes to entities that were queried from the database. The change tracker detects modifications when `SaveChanges` is called.

```csharp
// Tracked query (default)
var product = await context.Products.FirstAsync(p => p.Id == 1);
product.Price = 19.99m;
await context.SaveChangesAsync(); // UPDATE generated

// No-tracking query
var products = await context.Products
    .AsNoTracking()
    .ToListAsync();

// Disable tracking at context level
context.ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;

// Check tracked entities
var trackedEntities = context.ChangeTracker.Entries()
    .Where(e => e.State != EntityState.Unchanged)
    .Select(e => new
    {
        Entity = e.Entity.GetType().Name,
        State = e.State,
        Modified = e.Properties
            .Where(p => p.IsModified)
            .Select(p => p.Metadata.Name)
    });
```

### Auto-Detect Changes

```csharp
// Disable auto-detect for bulk operations (performance optimization)
context.ChangeTracker.AutoDetectChangesEnabled = false;

foreach (var product in productsToAdd)
{
    context.Products.Add(product);
}

context.ChangeTracker.DetectChanges(); // Manual detection
await context.SaveChangesAsync();

context.ChangeTracker.AutoDetectChangesEnabled = true; // Re-enable
```

---

## Entity States

| State | Description | On SaveChanges |
|-------|-------------|---------------|
| `Added` | Entity is new, not yet in DB | INSERT |
| `Unchanged` | Entity exists and has not been modified | No action |
| `Modified` | Entity exists and properties have changed | UPDATE |
| `Deleted` | Entity is marked for deletion | DELETE |
| `Detached` | Entity is not tracked by the context | No action |

```csharp
// Check entity state
var entry = context.Entry(product);
Console.WriteLine(entry.State); // EntityState.Unchanged

// Manually set state
context.Entry(product).State = EntityState.Modified;

// Attach detached entity
var detachedProduct = new Product { Id = 1, Name = "Updated", Price = 29.99m };
context.Attach(detachedProduct);
context.Entry(detachedProduct).State = EntityState.Modified;
await context.SaveChangesAsync();

// Mark specific properties as modified
context.Attach(detachedProduct);
context.Entry(detachedProduct).Property(p => p.Price).IsModified = true;
await context.SaveChangesAsync(); // Only updates Price column
```

---

## Interceptors

Interceptors allow you to intercept, modify, and suppress EF Core operations.

### SaveChanges Interceptor

```csharp
using Microsoft.EntityFrameworkCore.Diagnostics;

public class AuditSaveChangesInterceptor : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        var context = eventData.Context;
        if (context is null) return ValueTask.FromResult(result);

        var auditEntries = new List<AuditEntry>();

        foreach (var entry in context.ChangeTracker.Entries())
        {
            if (entry.State is EntityState.Added or EntityState.Modified or EntityState.Deleted)
            {
                auditEntries.Add(new AuditEntry
                {
                    EntityName = entry.Entity.GetType().Name,
                    Action = entry.State.ToString(),
                    Timestamp = DateTime.UtcNow,
                    Changes = entry.Properties
                        .Where(p => p.IsModified)
                        .ToDictionary(p => p.Metadata.Name, p => p.CurrentValue?.ToString())
                });
            }
        }

        context.Set<AuditEntry>().AddRange(auditEntries);
        return ValueTask.FromResult(result);
    }
}
```

### Command Interceptor

```csharp
public class SlowQueryInterceptor : DbCommandInterceptor
{
    private readonly ILogger<SlowQueryInterceptor> _logger;
    private const int SlowQueryThresholdMs = 500;

    public SlowQueryInterceptor(ILogger<SlowQueryInterceptor> logger)
    {
        _logger = logger;
    }

    public override ValueTask<DbDataReader> ReaderExecutedAsync(
        DbCommand command,
        CommandExecutedEventData eventData,
        DbDataReader result,
        CancellationToken cancellationToken = default)
    {
        if (eventData.Duration.TotalMilliseconds > SlowQueryThresholdMs)
        {
            _logger.LogWarning(
                "Slow query detected ({Duration}ms): {CommandText}",
                eventData.Duration.TotalMilliseconds,
                command.CommandText);
        }

        return ValueTask.FromResult(result);
    }
}
```

### Connection Interceptor

```csharp
public class ConnectionInterceptor : DbConnectionInterceptor
{
    public override async ValueTask<InterceptionResult> ConnectionOpeningAsync(
        DbConnection connection,
        ConnectionEventData eventData,
        InterceptionResult result,
        CancellationToken cancellationToken = default)
    {
        // Modify connection before opening (e.g., set access token)
        if (connection is SqlConnection sqlConnection)
        {
            sqlConnection.AccessToken = await GetAccessTokenAsync();
        }

        return result;
    }
}
```

### Registering Interceptors

```csharp
builder.Services.AddDbContext<AppDbContext>((sp, options) =>
{
    options.UseSqlServer(connectionString);
    options.AddInterceptors(
        sp.GetRequiredService<AuditSaveChangesInterceptor>(),
        sp.GetRequiredService<SlowQueryInterceptor>());
});

// Register interceptors in DI
builder.Services.AddSingleton<AuditSaveChangesInterceptor>();
builder.Services.AddSingleton<SlowQueryInterceptor>();
```

---

## Best Practices and Common Pitfalls

### Best Practices

| Practice | Description |
|----------|-------------|
| Use DI for DbContext | Register with `AddDbContext` or `AddDbContextFactory` |
| Prefer async methods | Use `SaveChangesAsync`, `ToListAsync`, etc. |
| Use `IEntityTypeConfiguration` | Keep `OnModelCreating` clean with separate config classes |
| Apply configurations from assembly | `ApplyConfigurationsFromAssembly` avoids manual registration |
| Use DbContext factory for Blazor | Blazor Server has different lifetime semantics |
| Enable retry on failure | Use `EnableRetryOnFailure` for production resilience |
| Override SaveChanges for auditing | Centralize audit logic in one place |
| Use interceptors over overrides | Interceptors are composable and testable |

### Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Long-lived DbContext | Memory leaks, stale data | Scope per request or use factory |
| Injecting scoped services with pooling | Services captured once | Use factory pattern instead |
| Forgetting `AsNoTracking` for reads | Unnecessary memory overhead | Add `AsNoTracking()` for read-only queries |
| Calling `SaveChanges` in a loop | N database round-trips | Batch changes, call `SaveChanges` once |
| Not using `CancellationToken` | Cannot cancel long queries | Pass token from controller to EF |
| Using `DbContext` across threads | Not thread-safe | Create new instance per thread via factory |
| Ignoring `OnModelCreating` order | Config may conflict | Use `IEntityTypeConfiguration` classes |
| Not disposing DbContext | Connection leaks | Use `using` or let DI handle lifetime |
