# ASP.NET Core Dependency Injection

> Official Documentation: https://learn.microsoft.com/aspnet/core/fundamentals/dependency-injection

## Overview

ASP.NET Core has a built-in Inversion of Control (IoC) container that supports constructor injection by default. Services are registered in `Program.cs` with specific lifetimes and resolved automatically when types are requested through constructors, middleware, or other injection points. The DI system is a first-class citizen in ASP.NET Core -- nearly every framework feature relies on it.

---

## 1. Service Lifetimes

### Lifetime Comparison

| Lifetime | Registration | Instance Creation | Disposal |
|---|---|---|---|
| **Transient** | `AddTransient<T>()` | New instance per request from the container | Disposed when scope ends |
| **Scoped** | `AddScoped<T>()` | One instance per HTTP request (scope) | Disposed at end of request |
| **Singleton** | `AddSingleton<T>()` | One instance for app lifetime | Disposed at app shutdown |

### When to Use Each Lifetime

| Lifetime | Best For | Examples |
|---|---|---|
| Transient | Lightweight, stateless services | Validators, formatters, mappers |
| Scoped | Services that hold per-request state | DbContext, UnitOfWork, current user |
| Singleton | Expensive-to-create, thread-safe services | HttpClient factory, caching, config |

```csharp
var builder = WebApplication.CreateBuilder(args);

// Transient - new instance every time it is injected
builder.Services.AddTransient<IEmailSender, SmtpEmailSender>();

// Scoped - one instance per HTTP request
builder.Services.AddScoped<IOrderService, OrderService>();
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();

// Singleton - one instance for the entire app lifetime
builder.Services.AddSingleton<ICacheService, RedisCacheService>();
builder.Services.AddSingleton<IConnectionMultiplexer>(sp =>
    ConnectionMultiplexer.Connect(builder.Configuration.GetConnectionString("Redis")!));
```

---

## 2. Registration Methods

### Interface-to-Implementation

```csharp
// Generic registration
builder.Services.AddTransient<IProductService, ProductService>();
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.AddSingleton<IMetricsCollector, PrometheusMetricsCollector>();

// Implementation-only (concrete type)
builder.Services.AddScoped<ProductService>();

// Instance registration (singleton only)
var settings = new AppSettings { ApiKey = "abc123" };
builder.Services.AddSingleton(settings);
```

### Factory-Based Registration

```csharp
// Factory delegate -- use when construction requires logic
builder.Services.AddScoped<INotificationService>(sp =>
{
    var config = sp.GetRequiredService<IConfiguration>();
    var logger = sp.GetRequiredService<ILogger<NotificationService>>();
    var provider = config["Notifications:Provider"];

    return provider switch
    {
        "sendgrid" => new SendGridNotificationService(
            config["Notifications:SendGridApiKey"]!, logger),
        "ses" => new SesNotificationService(
            config["Notifications:AwsRegion"]!, logger),
        _ => new ConsoleNotificationService(logger)
    };
});

// Factory with async initialization (use a wrapper)
builder.Services.AddSingleton<IDatabaseConnection>(sp =>
{
    var connection = new DatabaseConnection(
        sp.GetRequiredService<IConfiguration>().GetConnectionString("Main")!);
    connection.OpenAsync().GetAwaiter().GetResult(); // OK in startup
    return connection;
});
```

### TryAdd Methods (Prevent Duplicate Registration)

```csharp
using Microsoft.Extensions.DependencyInjection.Extensions;

// Only registers if no service of this type exists
builder.Services.TryAddTransient<IEmailSender, SmtpEmailSender>();
builder.Services.TryAddScoped<IOrderService, OrderService>();
builder.Services.TryAddSingleton<ICacheService, MemoryCacheService>();

// Only registers if no implementation of this type exists
builder.Services.TryAddEnumerable(
    ServiceDescriptor.Transient<IValidator, EmailValidator>());
builder.Services.TryAddEnumerable(
    ServiceDescriptor.Transient<IValidator, PhoneValidator>());
// Both register because implementations differ
```

---

## 3. Constructor Injection

```csharp
namespace MyApp.Services;

public interface IOrderService
{
    Task<OrderDto> CreateAsync(CreateOrderDto dto, CancellationToken ct);
    Task<OrderDto?> GetByIdAsync(int id, CancellationToken ct);
}

public class OrderService : IOrderService
{
    private readonly IOrderRepository _orderRepository;
    private readonly IProductService _productService;
    private readonly IEmailSender _emailSender;
    private readonly ILogger<OrderService> _logger;

    // Constructor injection - all dependencies resolved by DI container
    public OrderService(
        IOrderRepository orderRepository,
        IProductService productService,
        IEmailSender emailSender,
        ILogger<OrderService> logger)
    {
        _orderRepository = orderRepository;
        _productService = productService;
        _emailSender = emailSender;
        _logger = logger;
    }

    public async Task<OrderDto> CreateAsync(CreateOrderDto dto, CancellationToken ct)
    {
        _logger.LogInformation("Creating order for customer {CustomerId}", dto.CustomerId);

        foreach (var item in dto.Items)
        {
            var product = await _productService.GetByIdAsync(item.ProductId, ct)
                ?? throw new NotFoundException($"Product {item.ProductId} not found");

            if (product.StockQuantity < item.Quantity)
                throw new InsufficientStockException(product.Id, item.Quantity);
        }

        var order = await _orderRepository.CreateAsync(dto, ct);
        await _emailSender.SendOrderConfirmationAsync(order, ct);

        return order;
    }

    public async Task<OrderDto?> GetByIdAsync(int id, CancellationToken ct)
    {
        return await _orderRepository.GetByIdAsync(id, ct);
    }
}
```

### Primary Constructor Injection (.NET 8+)

```csharp
public class OrderService(
    IOrderRepository orderRepository,
    IProductService productService,
    IEmailSender emailSender,
    ILogger<OrderService> logger) : IOrderService
{
    public async Task<OrderDto> CreateAsync(CreateOrderDto dto, CancellationToken ct)
    {
        logger.LogInformation("Creating order for customer {CustomerId}", dto.CustomerId);
        var order = await orderRepository.CreateAsync(dto, ct);
        await emailSender.SendOrderConfirmationAsync(order, ct);
        return order;
    }

    public async Task<OrderDto?> GetByIdAsync(int id, CancellationToken ct)
    {
        return await orderRepository.GetByIdAsync(id, ct);
    }
}
```

---

## 4. Options Pattern

### IOptions<T>, IOptionsSnapshot<T>, IOptionsMonitor<T>

| Interface | Lifetime | Reloading | Use Case |
|---|---|---|---|
| `IOptions<T>` | Singleton | No -- reads once at startup | Static configuration |
| `IOptionsSnapshot<T>` | Scoped | Yes -- per request | Config that changes between deploys |
| `IOptionsMonitor<T>` | Singleton | Yes -- real-time notification | Singleton services needing live config |

```json
// appsettings.json
{
  "EmailSettings": {
    "SmtpHost": "smtp.example.com",
    "SmtpPort": 587,
    "UseSsl": true,
    "FromAddress": "noreply@example.com",
    "FromName": "My App"
  },
  "StorageSettings": {
    "Provider": "S3",
    "BucketName": "my-app-uploads",
    "MaxFileSizeMb": 50
  }
}
```

```csharp
// Configuration classes
public class EmailSettings
{
    public const string SectionName = "EmailSettings";

    public required string SmtpHost { get; set; }
    public int SmtpPort { get; set; } = 587;
    public bool UseSsl { get; set; } = true;
    public required string FromAddress { get; set; }
    public required string FromName { get; set; }
}

public class StorageSettings
{
    public const string SectionName = "StorageSettings";

    public required string Provider { get; set; }
    public required string BucketName { get; set; }
    public int MaxFileSizeMb { get; set; } = 10;
}

// Registration in Program.cs
builder.Services.Configure<EmailSettings>(
    builder.Configuration.GetSection(EmailSettings.SectionName));
builder.Services.Configure<StorageSettings>(
    builder.Configuration.GetSection(StorageSettings.SectionName));

// Usage in services
public class EmailSender(IOptions<EmailSettings> emailOptions) : IEmailSender
{
    private readonly EmailSettings _settings = emailOptions.Value;

    public async Task SendAsync(string to, string subject, string body)
    {
        using var client = new SmtpClient(_settings.SmtpHost, _settings.SmtpPort);
        client.EnableSsl = _settings.UseSsl;

        var message = new MailMessage(
            new MailAddress(_settings.FromAddress, _settings.FromName),
            new MailAddress(to))
        {
            Subject = subject,
            Body = body,
            IsBodyHtml = true
        };

        await client.SendMailAsync(message);
    }
}

// IOptionsMonitor for singleton services with live reload
public class CacheService : ICacheService, IDisposable
{
    private readonly IDisposable? _optionsChangeListener;
    private CacheSettings _settings;

    public CacheService(IOptionsMonitor<CacheSettings> optionsMonitor)
    {
        _settings = optionsMonitor.CurrentValue;
        _optionsChangeListener = optionsMonitor.OnChange(newSettings =>
        {
            _settings = newSettings;
            // React to configuration changes
        });
    }

    public void Dispose() => _optionsChangeListener?.Dispose();
}
```

### Options Validation

```csharp
builder.Services.AddOptions<EmailSettings>()
    .BindConfiguration(EmailSettings.SectionName)
    .ValidateDataAnnotations()   // Validate [Required], [Range], etc.
    .ValidateOnStart();          // Fail fast at startup if invalid

// Data annotations on the settings class
public class EmailSettings
{
    public const string SectionName = "EmailSettings";

    [Required]
    public required string SmtpHost { get; set; }

    [Range(1, 65535)]
    public int SmtpPort { get; set; } = 587;

    public bool UseSsl { get; set; } = true;

    [Required, EmailAddress]
    public required string FromAddress { get; set; }

    [Required, StringLength(100)]
    public required string FromName { get; set; }
}

// Custom validation with Validate delegate
builder.Services.AddOptions<StorageSettings>()
    .BindConfiguration(StorageSettings.SectionName)
    .Validate(settings =>
    {
        if (settings.Provider == "S3" && string.IsNullOrEmpty(settings.BucketName))
            return false;
        return true;
    }, "BucketName is required when Provider is S3")
    .ValidateOnStart();

// Custom IValidateOptions<T> implementation
public class StorageSettingsValidator : IValidateOptions<StorageSettings>
{
    public ValidateOptionsResult Validate(string? name, StorageSettings options)
    {
        var failures = new List<string>();

        if (options.MaxFileSizeMb > 100)
            failures.Add("MaxFileSizeMb cannot exceed 100");

        if (options.Provider is not ("S3" or "Azure" or "Local"))
            failures.Add($"Invalid provider: {options.Provider}");

        return failures.Count > 0
            ? ValidateOptionsResult.Fail(failures)
            : ValidateOptionsResult.Success;
    }
}

builder.Services.AddSingleton<IValidateOptions<StorageSettings>, StorageSettingsValidator>();
```

---

## 5. Keyed Services (.NET 8+)

```csharp
// Register keyed services
builder.Services.AddKeyedSingleton<IStorageService, S3StorageService>("s3");
builder.Services.AddKeyedSingleton<IStorageService, AzureBlobStorageService>("azure");
builder.Services.AddKeyedSingleton<IStorageService, LocalStorageService>("local");

// Inject by key using [FromKeyedServices]
public class FileUploadService(
    [FromKeyedServices("s3")] IStorageService primaryStorage,
    [FromKeyedServices("local")] IStorageService fallbackStorage,
    ILogger<FileUploadService> logger) : IFileUploadService
{
    public async Task<string> UploadAsync(Stream file, string fileName)
    {
        try
        {
            return await primaryStorage.UploadAsync(file, fileName);
        }
        catch (Exception ex)
        {
            logger.LogWarning(ex, "Primary storage failed, using fallback");
            return await fallbackStorage.UploadAsync(file, fileName);
        }
    }
}

// Resolve keyed services manually
public class StorageFactory(IServiceProvider serviceProvider)
{
    public IStorageService GetStorage(string provider)
    {
        return serviceProvider.GetRequiredKeyedService<IStorageService>(provider);
    }
}

// Keyed services in minimal APIs
app.MapPost("/upload/{provider}", async (
    string provider,
    IFormFile file,
    [FromKeyedServices("s3")] IStorageService s3Storage,
    [FromKeyedServices("azure")] IStorageService azureStorage) =>
{
    var storage = provider == "s3" ? s3Storage : azureStorage;
    var url = await storage.UploadAsync(file.OpenReadStream(), file.FileName);
    return Results.Ok(new { url });
});
```

---

## 6. Open Generics Registration

```csharp
// Generic repository interface
public interface IRepository<T> where T : class, IEntity
{
    Task<T?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<IReadOnlyList<T>> GetAllAsync(CancellationToken ct = default);
    Task<T> AddAsync(T entity, CancellationToken ct = default);
    Task UpdateAsync(T entity, CancellationToken ct = default);
    Task DeleteAsync(int id, CancellationToken ct = default);
}

public class Repository<T> : IRepository<T> where T : class, IEntity
{
    private readonly AppDbContext _context;

    public Repository(AppDbContext context)
    {
        _context = context;
    }

    public async Task<T?> GetByIdAsync(int id, CancellationToken ct = default)
        => await _context.Set<T>().FindAsync(new object[] { id }, ct);

    public async Task<IReadOnlyList<T>> GetAllAsync(CancellationToken ct = default)
        => await _context.Set<T>().ToListAsync(ct);

    public async Task<T> AddAsync(T entity, CancellationToken ct = default)
    {
        await _context.Set<T>().AddAsync(entity, ct);
        await _context.SaveChangesAsync(ct);
        return entity;
    }

    public async Task UpdateAsync(T entity, CancellationToken ct = default)
    {
        _context.Set<T>().Update(entity);
        await _context.SaveChangesAsync(ct);
    }

    public async Task DeleteAsync(int id, CancellationToken ct = default)
    {
        var entity = await GetByIdAsync(id, ct)
            ?? throw new NotFoundException(typeof(T).Name, id);
        _context.Set<T>().Remove(entity);
        await _context.SaveChangesAsync(ct);
    }
}

// Register open generic -- resolves to Repository<Product>, Repository<Order>, etc.
builder.Services.AddScoped(typeof(IRepository<>), typeof(Repository<>));

// Generic pipeline behavior (MediatR-style)
public interface IPipelineBehavior<TRequest, TResponse>
{
    Task<TResponse> Handle(TRequest request, Func<Task<TResponse>> next, CancellationToken ct);
}

public class LoggingBehavior<TRequest, TResponse>(
    ILogger<LoggingBehavior<TRequest, TResponse>> logger)
    : IPipelineBehavior<TRequest, TResponse>
{
    public async Task<TResponse> Handle(
        TRequest request, Func<Task<TResponse>> next, CancellationToken ct)
    {
        var requestName = typeof(TRequest).Name;
        logger.LogInformation("Handling {Request}", requestName);

        var response = await next();

        logger.LogInformation("Handled {Request}", requestName);
        return response;
    }
}

builder.Services.AddScoped(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
```

---

## 7. Multiple Implementations (IEnumerable<T>)

```csharp
public interface INotificationChannel
{
    string Name { get; }
    Task SendAsync(Notification notification, CancellationToken ct = default);
    bool CanHandle(NotificationType type);
}

public class EmailNotificationChannel(IEmailSender emailSender) : INotificationChannel
{
    public string Name => "Email";
    public bool CanHandle(NotificationType type) => type is NotificationType.Email or NotificationType.All;

    public async Task SendAsync(Notification notification, CancellationToken ct)
    {
        await emailSender.SendAsync(notification.Recipient, notification.Subject, notification.Body);
    }
}

public class SmsNotificationChannel(ISmsProvider smsProvider) : INotificationChannel
{
    public string Name => "SMS";
    public bool CanHandle(NotificationType type) => type is NotificationType.Sms or NotificationType.All;

    public async Task SendAsync(Notification notification, CancellationToken ct)
    {
        await smsProvider.SendAsync(notification.PhoneNumber!, notification.Body);
    }
}

public class PushNotificationChannel(IPushService pushService) : INotificationChannel
{
    public string Name => "Push";
    public bool CanHandle(NotificationType type) => type is NotificationType.Push or NotificationType.All;

    public async Task SendAsync(Notification notification, CancellationToken ct)
    {
        await pushService.SendAsync(notification.DeviceToken!, notification.Title, notification.Body);
    }
}

// Register all implementations
builder.Services.AddScoped<INotificationChannel, EmailNotificationChannel>();
builder.Services.AddScoped<INotificationChannel, SmsNotificationChannel>();
builder.Services.AddScoped<INotificationChannel, PushNotificationChannel>();

// Inject all implementations
public class NotificationDispatcher(
    IEnumerable<INotificationChannel> channels,
    ILogger<NotificationDispatcher> logger) : INotificationDispatcher
{
    public async Task DispatchAsync(Notification notification, CancellationToken ct)
    {
        var applicableChannels = channels.Where(c => c.CanHandle(notification.Type));

        foreach (var channel in applicableChannels)
        {
            try
            {
                await channel.SendAsync(notification, ct);
                logger.LogInformation("Sent via {Channel}", channel.Name);
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "Failed to send via {Channel}", channel.Name);
            }
        }
    }
}
```

---

## 8. Decorator Pattern with DI

```csharp
// Base service
public interface IProductService
{
    Task<ProductDto?> GetByIdAsync(int id, CancellationToken ct);
}

public class ProductService(IProductRepository repo) : IProductService
{
    public async Task<ProductDto?> GetByIdAsync(int id, CancellationToken ct)
        => await repo.GetByIdAsync(id, ct);
}

// Caching decorator
public class CachedProductService(
    IProductService inner,
    IDistributedCache cache,
    ILogger<CachedProductService> logger) : IProductService
{
    public async Task<ProductDto?> GetByIdAsync(int id, CancellationToken ct)
    {
        var cacheKey = $"product:{id}";
        var cached = await cache.GetStringAsync(cacheKey, ct);

        if (cached is not null)
        {
            logger.LogDebug("Cache hit for {CacheKey}", cacheKey);
            return JsonSerializer.Deserialize<ProductDto>(cached);
        }

        var product = await inner.GetByIdAsync(id, ct);

        if (product is not null)
        {
            await cache.SetStringAsync(cacheKey,
                JsonSerializer.Serialize(product),
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
                }, ct);
        }

        return product;
    }
}

// Register with decorator
builder.Services.AddScoped<ProductService>();
builder.Services.AddScoped<IProductService>(sp =>
{
    var inner = sp.GetRequiredService<ProductService>();
    var cache = sp.GetRequiredService<IDistributedCache>();
    var logger = sp.GetRequiredService<ILogger<CachedProductService>>();
    return new CachedProductService(inner, cache, logger);
});

// Using Scrutor library for cleaner decorator registration
// Install: Scrutor
builder.Services.AddScoped<IProductService, ProductService>();
builder.Services.Decorate<IProductService, CachedProductService>();
builder.Services.Decorate<IProductService, LoggingProductService>(); // Chain decorators
```

---

## 9. Service Descriptor and Replace

```csharp
using Microsoft.Extensions.DependencyInjection.Extensions;

// Replace an existing registration (useful for testing or overriding defaults)
builder.Services.Replace(ServiceDescriptor.Scoped<IEmailSender, MockEmailSender>());

// Remove and re-add
builder.Services.RemoveAll<IEmailSender>();
builder.Services.AddScoped<IEmailSender, CustomEmailSender>();

// Inspect registrations
var descriptor = builder.Services.FirstOrDefault(
    d => d.ServiceType == typeof(IEmailSender));
if (descriptor is not null)
{
    Console.WriteLine($"Implementation: {descriptor.ImplementationType?.Name}");
    Console.WriteLine($"Lifetime: {descriptor.Lifetime}");
}
```

---

## 10. Scope Validation and Common DI Patterns

### Extension Method Registration Pattern

```csharp
// Organize registrations into extension methods
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddApplicationServices(this IServiceCollection services)
    {
        services.AddScoped<IOrderService, OrderService>();
        services.AddScoped<IProductService, ProductService>();
        services.AddScoped<ICustomerService, CustomerService>();
        return services;
    }

    public static IServiceCollection AddInfrastructureServices(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContext<AppDbContext>(options =>
            options.UseNpgsql(configuration.GetConnectionString("Default")));

        services.AddScoped(typeof(IRepository<>), typeof(Repository<>));

        services.AddStackExchangeRedisCache(options =>
        {
            options.Configuration = configuration.GetConnectionString("Redis");
        });

        return services;
    }
}

// Clean Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddApplicationServices();
builder.Services.AddInfrastructureServices(builder.Configuration);
```

### Assembly Scanning Registration

```csharp
// Using reflection to auto-register services
public static IServiceCollection AddServicesFromAssembly(
    this IServiceCollection services,
    Assembly assembly)
{
    var serviceTypes = assembly.GetTypes()
        .Where(t => t is { IsAbstract: false, IsInterface: false })
        .SelectMany(t => t.GetInterfaces()
            .Where(i => i.Name.EndsWith("Service") || i.Name.EndsWith("Repository"))
            .Select(i => new { Interface = i, Implementation = t }));

    foreach (var service in serviceTypes)
    {
        services.AddScoped(service.Interface, service.Implementation);
    }

    return services;
}

builder.Services.AddServicesFromAssembly(typeof(Program).Assembly);
```

---

## Best Practices

1. **Prefer constructor injection** - It makes dependencies explicit and testable.
2. **Use IOptions<T> for configuration** - Strongly-typed configuration over string-keyed access.
3. **Register as the most restrictive lifetime** - Prefer Scoped or Transient unless Singleton is needed.
4. **Use extension methods for registration** - Organize registrations by feature or layer.
5. **Validate options on start** - Use `ValidateOnStart()` to fail fast with misconfiguration.
6. **Use keyed services for strategy pattern** - .NET 8+ keyed services replace manual factory patterns.
7. **Prefer TryAdd for library code** - Allows consumers to override default implementations.
8. **Use open generics** - Reduce repetitive registration for common patterns like repositories.

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---|---|---|
| Captive dependency | Singleton holds reference to a Scoped/Transient service | Never inject shorter-lived services into Singleton |
| Service locator | Injecting `IServiceProvider` directly | Use constructor injection or factory pattern |
| Scope validation disabled | Captive dependencies go undetected | Enable `ValidateScopes` in development |
| Missing registration | `InvalidOperationException` at runtime | Validate service graph at startup |
| Disposable transients | Transient `IDisposable` services not disposed | Prefer Scoped lifetime for disposable services |
| Circular dependencies | A depends on B, B depends on A | Refactor to break the cycle or use `Lazy<T>` |
| Resolving in Configure | Services may not be fully registered yet | Use `app.Services` only after `Build()` |
| Over-using factories | Complex factory lambdas obscure dependencies | Use concrete types or decorator pattern |

### Scope Validation

```csharp
var builder = WebApplication.CreateBuilder(args);

// In development, this is enabled by default
// It detects captive dependencies (singleton -> scoped)
if (builder.Environment.IsDevelopment())
{
    builder.Host.UseDefaultServiceProvider(options =>
    {
        options.ValidateScopes = true;     // Detect captive dependencies
        options.ValidateOnBuild = true;    // Validate all registrations at startup
    });
}
```
