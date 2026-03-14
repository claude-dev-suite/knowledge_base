# ASP.NET Core Middleware

> Official Documentation: https://learn.microsoft.com/aspnet/core/fundamentals/middleware/

## Overview

Middleware are components assembled into the HTTP request pipeline to handle requests and responses. Each middleware component can perform work before and after the next component in the pipeline, and can short-circuit the pipeline by not calling the next delegate. Middleware is configured in `Program.cs` and executes in the order it is registered.

---

## 1. Request Pipeline Concept

The ASP.NET Core request pipeline consists of a sequence of request delegates called one after another. Each delegate can perform operations before and after the next delegate. This creates the "Russian doll" nesting model.

```
Request  -->  Middleware 1  -->  Middleware 2  -->  Middleware 3  -->  Endpoint
Response <--  Middleware 1  <--  Middleware 2  <--  Middleware 3  <--  Endpoint
```

```csharp
var app = builder.Build();

// Each Use() adds middleware to the pipeline
app.Use(async (context, next) =>
{
    // Before the next middleware
    Console.WriteLine($"Request: {context.Request.Method} {context.Request.Path}");

    await next(context); // Call the next middleware

    // After the next middleware (response is coming back)
    Console.WriteLine($"Response: {context.Response.StatusCode}");
});

app.MapGet("/", () => "Hello World");

app.Run();
```

---

## 2. Built-in Middleware Ordering

The order of middleware registration matters significantly. The recommended order for a typical ASP.NET Core application is:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddAuthentication();
builder.Services.AddAuthorization();
builder.Services.AddCors();
builder.Services.AddRateLimiter(options => { /* ... */ });

var app = builder.Build();

// 1. Exception/error handling (outermost - catches everything)
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}
else
{
    app.UseExceptionHandler("/error");
    app.UseHsts();                        // 2. HSTS (HTTP Strict Transport Security)
}

app.UseHttpsRedirection();                // 3. HTTPS redirection

app.UseStaticFiles();                     // 4. Static files (short-circuits for static content)

app.UseRouting();                         // 5. Routing (determines endpoint)

app.UseRequestLocalization();             // 6. Request localization

app.UseCors();                            // 7. CORS (must be after routing, before auth)

app.UseAuthentication();                  // 8. Authentication (who are you?)
app.UseAuthorization();                   // 9. Authorization (what can you do?)

app.UseRateLimiter();                     // 10. Rate limiting

app.UseResponseCaching();                 // 11. Response caching
app.UseResponseCompression();             // 12. Response compression

app.MapControllers();                     // 13. Endpoint execution
app.MapHealthChecks("/health");

app.Run();
```

### Middleware Order Summary

| Order | Middleware | Purpose |
|---|---|---|
| 1 | `UseExceptionHandler` | Global exception handling |
| 2 | `UseHsts` | HTTP Strict Transport Security header |
| 3 | `UseHttpsRedirection` | Redirect HTTP to HTTPS |
| 4 | `UseStaticFiles` | Serve static files (wwwroot) |
| 5 | `UseRouting` | Match request to endpoint |
| 6 | `UseRequestLocalization` | Localize requests |
| 7 | `UseCors` | Cross-origin resource sharing |
| 8 | `UseAuthentication` | Authenticate the user |
| 9 | `UseAuthorization` | Authorize the user |
| 10 | `UseRateLimiter` | Rate limiting |
| 11 | `UseResponseCaching` | Cache responses |
| 12 | `UseResponseCompression` | Compress responses |
| 13 | `MapControllers` / `MapGet` | Execute endpoint |

---

## 3. Custom Middleware (Convention-Based)

Convention-based middleware is a class with an `Invoke` or `InvokeAsync` method and a constructor that accepts `RequestDelegate`.

```csharp
namespace MyApp.Middleware;

public class RequestTimingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestTimingMiddleware> _logger;

    // Constructor receives the next middleware in the pipeline
    // Note: Constructor-injected services must be Singleton lifetime
    public RequestTimingMiddleware(
        RequestDelegate next,
        ILogger<RequestTimingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    // InvokeAsync can accept Scoped and Transient services as parameters
    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();

        // Add timing header
        context.Response.OnStarting(() =>
        {
            stopwatch.Stop();
            context.Response.Headers["X-Response-Time"] =
                $"{stopwatch.ElapsedMilliseconds}ms";
            return Task.CompletedTask;
        });

        await _next(context);

        _logger.LogInformation(
            "{Method} {Path} responded {StatusCode} in {Elapsed}ms",
            context.Request.Method,
            context.Request.Path,
            context.Response.StatusCode,
            stopwatch.ElapsedMilliseconds);
    }
}

// Extension method for clean registration
public static class RequestTimingMiddlewareExtensions
{
    public static IApplicationBuilder UseRequestTiming(this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<RequestTimingMiddleware>();
    }
}

// Usage in Program.cs
app.UseRequestTiming();
```

### Middleware with Scoped Dependencies

```csharp
public class TenantResolutionMiddleware
{
    private readonly RequestDelegate _next;

    public TenantResolutionMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    // Scoped service (ITenantService) injected via method parameter
    public async Task InvokeAsync(
        HttpContext context,
        ITenantService tenantService,
        ILogger<TenantResolutionMiddleware> logger)
    {
        var tenantId = context.Request.Headers["X-Tenant-Id"].FirstOrDefault();

        if (string.IsNullOrEmpty(tenantId))
        {
            // Try subdomain
            var host = context.Request.Host.Host;
            tenantId = host.Split('.').FirstOrDefault();
        }

        if (string.IsNullOrEmpty(tenantId))
        {
            context.Response.StatusCode = StatusCodes.Status400BadRequest;
            await context.Response.WriteAsJsonAsync(new
            {
                error = "Tenant ID is required"
            });
            return; // Short-circuit -- do not call next
        }

        var tenant = await tenantService.GetByIdAsync(tenantId);
        if (tenant is null)
        {
            context.Response.StatusCode = StatusCodes.Status404NotFound;
            await context.Response.WriteAsJsonAsync(new
            {
                error = $"Tenant '{tenantId}' not found"
            });
            return;
        }

        // Store tenant in HttpContext.Items for downstream access
        context.Items["Tenant"] = tenant;
        logger.LogInformation("Resolved tenant: {TenantId}", tenantId);

        await _next(context);
    }
}
```

---

## 4. Custom Middleware (Factory-Based with IMiddleware)

Factory-based middleware implements `IMiddleware` and is resolved from DI per request. This allows Scoped lifetime for the middleware itself.

```csharp
public class TransactionMiddleware : IMiddleware
{
    private readonly AppDbContext _dbContext;
    private readonly ILogger<TransactionMiddleware> _logger;

    // Constructor injection -- supports Scoped services because
    // IMiddleware is resolved per-request from the DI container
    public TransactionMiddleware(
        AppDbContext dbContext,
        ILogger<TransactionMiddleware> logger)
    {
        _dbContext = dbContext;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        // Only wrap write operations in transactions
        if (HttpMethods.IsGet(context.Request.Method) ||
            HttpMethods.IsHead(context.Request.Method) ||
            HttpMethods.IsOptions(context.Request.Method))
        {
            await next(context);
            return;
        }

        await using var transaction = await _dbContext.Database.BeginTransactionAsync();

        try
        {
            await next(context);

            if (context.Response.StatusCode < 400)
            {
                await transaction.CommitAsync();
                _logger.LogDebug("Transaction committed");
            }
            else
            {
                await transaction.RollbackAsync();
                _logger.LogWarning("Transaction rolled back due to status {Status}",
                    context.Response.StatusCode);
            }
        }
        catch
        {
            await transaction.RollbackAsync();
            _logger.LogError("Transaction rolled back due to exception");
            throw;
        }
    }
}

// IMiddleware MUST be registered in DI
builder.Services.AddScoped<TransactionMiddleware>();

// Usage
app.UseMiddleware<TransactionMiddleware>();
```

### Convention-Based vs Factory-Based Comparison

| Feature | Convention-Based | Factory-Based (IMiddleware) |
|---|---|---|
| Registration | `UseMiddleware<T>()` | DI + `UseMiddleware<T>()` |
| Middleware Lifetime | Singleton (created once) | Per-request (resolved from DI) |
| Constructor DI | Singleton services only | Any lifetime |
| Method DI | Via `InvokeAsync` parameters | Not available |
| Activation | By convention (reflection) | By DI container |

---

## 5. Exception Handling Middleware

```csharp
public class GlobalExceptionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<GlobalExceptionMiddleware> _logger;
    private readonly IHostEnvironment _env;

    public GlobalExceptionMiddleware(
        RequestDelegate next,
        ILogger<GlobalExceptionMiddleware> logger,
        IHostEnvironment env)
    {
        _next = next;
        _logger = logger;
        _env = env;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            await HandleExceptionAsync(context, ex);
        }
    }

    private async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        _logger.LogError(exception, "Unhandled exception: {Message}", exception.Message);

        var (statusCode, title) = exception switch
        {
            NotFoundException => (StatusCodes.Status404NotFound, "Not Found"),
            ValidationException => (StatusCodes.Status400BadRequest, "Validation Error"),
            UnauthorizedAccessException => (StatusCodes.Status401Unauthorized, "Unauthorized"),
            ForbiddenException => (StatusCodes.Status403Forbidden, "Forbidden"),
            ConflictException => (StatusCodes.Status409Conflict, "Conflict"),
            _ => (StatusCodes.Status500InternalServerError, "Internal Server Error")
        };

        var problemDetails = new ProblemDetails
        {
            Status = statusCode,
            Title = title,
            Detail = _env.IsDevelopment() ? exception.Message : null,
            Instance = context.Request.Path,
            Type = $"https://httpstatuses.com/{statusCode}"
        };

        if (exception is ValidationException validationEx)
        {
            problemDetails.Extensions["errors"] = validationEx.Errors;
        }

        if (_env.IsDevelopment())
        {
            problemDetails.Extensions["stackTrace"] = exception.StackTrace;
        }

        problemDetails.Extensions["traceId"] = context.TraceIdentifier;

        context.Response.StatusCode = statusCode;
        context.Response.ContentType = "application/problem+json";
        await context.Response.WriteAsJsonAsync(problemDetails);
    }
}

// Custom exception types
public class NotFoundException : Exception
{
    public NotFoundException(string entity, object id)
        : base($"{entity} with id '{id}' was not found.") { }
}

public class ValidationException : Exception
{
    public IDictionary<string, string[]> Errors { get; }

    public ValidationException(IDictionary<string, string[]> errors)
        : base("One or more validation errors occurred.")
    {
        Errors = errors;
    }
}

public class ConflictException : Exception
{
    public ConflictException(string message) : base(message) { }
}

public class ForbiddenException : Exception
{
    public ForbiddenException(string message = "Access denied") : base(message) { }
}

// Register as first middleware
app.UseMiddleware<GlobalExceptionMiddleware>();
```

### .NET 8+ IExceptionHandler

```csharp
// .NET 8 introduces IExceptionHandler for structured exception handling
public class NotFoundExceptionHandler : IExceptionHandler
{
    private readonly ILogger<NotFoundExceptionHandler> _logger;

    public NotFoundExceptionHandler(ILogger<NotFoundExceptionHandler> logger)
    {
        _logger = logger;
    }

    public async ValueTask<bool> TryHandleAsync(
        HttpContext context,
        Exception exception,
        CancellationToken ct)
    {
        if (exception is not NotFoundException notFoundEx)
            return false; // Not handled, pass to next handler

        _logger.LogWarning("Not found: {Message}", notFoundEx.Message);

        context.Response.StatusCode = StatusCodes.Status404NotFound;
        await context.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = StatusCodes.Status404NotFound,
            Title = "Not Found",
            Detail = notFoundEx.Message,
            Type = "https://httpstatuses.com/404"
        }, ct);

        return true; // Handled
    }
}

public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;

    public GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger)
    {
        _logger = logger;
    }

    public async ValueTask<bool> TryHandleAsync(
        HttpContext context,
        Exception exception,
        CancellationToken ct)
    {
        _logger.LogError(exception, "Unhandled exception");

        context.Response.StatusCode = StatusCodes.Status500InternalServerError;
        await context.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = StatusCodes.Status500InternalServerError,
            Title = "Internal Server Error",
            Type = "https://httpstatuses.com/500"
        }, ct);

        return true;
    }
}

// Registration -- handlers are tried in order
builder.Services.AddExceptionHandler<NotFoundExceptionHandler>();
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();

app.UseExceptionHandler();
```

---

## 6. Request/Response Manipulation

```csharp
// Correlation ID middleware
public class CorrelationIdMiddleware
{
    private const string CorrelationIdHeader = "X-Correlation-Id";
    private readonly RequestDelegate _next;

    public CorrelationIdMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // Read or generate correlation ID
        var correlationId = context.Request.Headers[CorrelationIdHeader].FirstOrDefault()
            ?? Guid.NewGuid().ToString("N");

        // Make it available throughout the request
        context.Items["CorrelationId"] = correlationId;

        // Add to response headers
        context.Response.OnStarting(() =>
        {
            context.Response.Headers[CorrelationIdHeader] = correlationId;
            return Task.CompletedTask;
        });

        // Add to logging scope
        using (context.RequestServices.GetRequiredService<ILogger<CorrelationIdMiddleware>>()
            .BeginScope(new Dictionary<string, object>
            {
                ["CorrelationId"] = correlationId
            }))
        {
            await _next(context);
        }
    }
}

// Request body buffering (for logging or multiple reads)
public class RequestBufferingMiddleware
{
    private readonly RequestDelegate _next;

    public RequestBufferingMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        // Enable buffering so the body can be read multiple times
        context.Request.EnableBuffering();
        await _next(context);
    }
}

// Security headers middleware
public class SecurityHeadersMiddleware
{
    private readonly RequestDelegate _next;

    public SecurityHeadersMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        context.Response.OnStarting(() =>
        {
            var headers = context.Response.Headers;
            headers["X-Content-Type-Options"] = "nosniff";
            headers["X-Frame-Options"] = "DENY";
            headers["X-XSS-Protection"] = "1; mode=block";
            headers["Referrer-Policy"] = "strict-origin-when-cross-origin";
            headers["Permissions-Policy"] = "camera=(), microphone=(), geolocation=()";
            headers.ContentSecurityPolicy =
                "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'";
            return Task.CompletedTask;
        });

        await _next(context);
    }
}
```

---

## 7. Branching and Terminal Middleware

### Map, MapWhen, UseWhen

```csharp
var app = builder.Build();

// Map - branch based on path prefix
// Creates a separate pipeline for /api paths
app.Map("/api", apiApp =>
{
    apiApp.UseAuthentication();
    apiApp.UseAuthorization();
    apiApp.UseMiddleware<ApiRateLimitingMiddleware>();
    apiApp.UseRouting();
    apiApp.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
});

// MapWhen - branch based on a predicate
app.MapWhen(
    context => context.Request.Headers.ContainsKey("X-Admin-Key"),
    adminApp =>
    {
        adminApp.UseMiddleware<AdminAuthMiddleware>();
        adminApp.Run(async context =>
        {
            await context.Response.WriteAsync("Admin endpoint");
        });
    });

// UseWhen - conditional middleware that rejoins the main pipeline
app.UseWhen(
    context => context.Request.Path.StartsWithSegments("/api"),
    apiApp =>
    {
        apiApp.UseMiddleware<ApiLoggingMiddleware>();
        // No terminal middleware -- request continues in main pipeline
    });

// UseWhen with content type check
app.UseWhen(
    context => context.Request.ContentType?.Contains("application/json") == true,
    jsonApp =>
    {
        jsonApp.UseMiddleware<JsonValidationMiddleware>();
    });
```

### Terminal Middleware (Run)

```csharp
// Run() adds terminal middleware - does NOT call next()
app.Run(async context =>
{
    await context.Response.WriteAsync("Terminal - pipeline ends here");
});

// Health check as terminal middleware
app.Map("/health", healthApp =>
{
    healthApp.Run(async context =>
    {
        context.Response.ContentType = "application/json";
        await context.Response.WriteAsJsonAsync(new
        {
            status = "healthy",
            timestamp = DateTime.UtcNow,
            version = Assembly.GetExecutingAssembly().GetName().Version?.ToString()
        });
    });
});
```

---

## 8. Middleware vs Filters Comparison

| Feature | Middleware | Filters |
|---|---|---|
| Scope | Entire HTTP pipeline | MVC/Minimal API actions only |
| Access to | HttpContext | ActionContext, endpoint metadata |
| Ordering | Registration order | Filter pipeline order |
| DI support | Constructor + method (convention) | Constructor (via [ServiceFilter]) |
| Short-circuit | Skip calling `next` | Set `context.Result` |
| Use for | Cross-cutting (logging, auth, CORS) | Action-specific (validation, caching) |
| Runs for | All requests | Only matched endpoints |

### When to Use Middleware vs Filters

| Scenario | Use |
|---|---|
| Logging all HTTP requests | Middleware |
| CORS, HTTPS redirect | Middleware |
| Authentication/Authorization | Middleware (built-in) |
| Request validation for specific controllers | Filter |
| Action-level caching | Filter |
| Exception handling | Either (middleware for global, filter for MVC-specific) |
| Response transformation for APIs | Filter (IResultFilter) |
| Security headers | Middleware |
| Rate limiting | Middleware |
| Tenant resolution | Middleware |

---

## 9. Complete Middleware Pipeline Example

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer();
builder.Services.AddAuthorization();
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowFrontend", policy =>
    {
        policy.WithOrigins("https://myapp.com")
            .AllowAnyMethod()
            .AllowAnyHeader()
            .AllowCredentials();
    });
});
builder.Services.AddHealthChecks()
    .AddNpgSql(builder.Configuration.GetConnectionString("Default")!);
builder.Services.AddResponseCompression();
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();

var app = builder.Build();

// Pipeline order matters
app.UseExceptionHandler();                   // 1. Catch all exceptions
app.UseMiddleware<CorrelationIdMiddleware>(); // 2. Correlation ID tracking
app.UseMiddleware<RequestTimingMiddleware>(); // 3. Request timing

if (!app.Environment.IsDevelopment())
{
    app.UseHsts();                           // 4. HSTS
}

app.UseHttpsRedirection();                   // 5. HTTPS redirect
app.UseResponseCompression();                // 6. Response compression
app.UseStaticFiles();                        // 7. Static files
app.UseMiddleware<SecurityHeadersMiddleware>(); // 8. Security headers
app.UseRouting();                            // 9. Routing
app.UseCors("AllowFrontend");                // 10. CORS
app.UseAuthentication();                     // 11. Authentication
app.UseAuthorization();                      // 12. Authorization
app.MapControllers();                        // 13. Endpoints
app.MapHealthChecks("/health");              // 14. Health checks

app.Run();
```

---

## Best Practices

1. **Respect middleware ordering** - Exception handling must be first; auth must come before authorization; CORS must be between routing and auth.
2. **Use extension methods** - Create `UseMyMiddleware()` extension methods for clean `Program.cs` registration.
3. **Prefer IMiddleware for scoped dependencies** - Convention-based middleware is singleton; use `IMiddleware` when you need scoped services as fields.
4. **Keep middleware focused** - Each middleware should have a single responsibility.
5. **Short-circuit when appropriate** - Return early for invalid requests instead of passing them through the entire pipeline.
6. **Use `OnStarting` for response headers** - Headers must be set before the response body starts writing.
7. **Enable request buffering explicitly** - Call `EnableBuffering()` if you need to read the body multiple times.
8. **Use `IExceptionHandler` in .NET 8+** - Prefer the structured exception handler interface over custom exception middleware.

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---|---|---|
| Wrong middleware order | Auth before routing, CORS after auth | Follow the documented ordering |
| Modifying response after headers sent | `InvalidOperationException` | Use `OnStarting` callback for headers |
| Convention middleware with scoped DI | Singleton middleware holds scoped service | Use `IMiddleware` or method injection |
| Forgetting `await next()` | Pipeline short-circuited unintentionally | Always call `next()` unless short-circuiting intentionally |
| Reading body without buffering | Body stream consumed, cannot re-read | Call `EnableBuffering()` first |
| Middleware after `Run()` | Never executes | `Run()` is terminal; place it last |
| Exception in `OnStarting` | Unhandled, crashes the response | Wrap in try-catch |
| Heavy sync work in middleware | Blocks the thread pool | Use `async` throughout |
