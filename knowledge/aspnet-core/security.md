# ASP.NET Core Security

> Official Documentation: https://learn.microsoft.com/aspnet/core/security/

## Overview

ASP.NET Core provides a comprehensive security framework covering authentication, authorization, data protection, HTTPS enforcement, CORS, rate limiting, and protection against common web vulnerabilities. The security middleware pipeline is designed to be composable, allowing you to combine multiple authentication schemes and authorization policies.

---

## 1. Authentication Schemes

### JWT Bearer Authentication

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using System.Text;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    var jwtConfig = builder.Configuration.GetSection("Jwt");

    options.TokenValidationParameters = new TokenValidationParameters
    {
        // Issuer validation
        ValidateIssuer = true,
        ValidIssuer = jwtConfig["Issuer"],

        // Audience validation
        ValidateAudience = true,
        ValidAudience = jwtConfig["Audience"],

        // Signing key validation
        ValidateIssuerSigningKey = true,
        IssuerSigningKey = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(jwtConfig["Secret"]!)),

        // Token lifetime validation
        ValidateLifetime = true,
        ClockSkew = TimeSpan.FromMinutes(1), // Default is 5 minutes

        // Required claims
        RequireExpirationTime = true,
        RequireSignedTokens = true
    };

    // Handle authentication events
    options.Events = new JwtBearerEvents
    {
        OnTokenValidated = context =>
        {
            var userId = context.Principal?.FindFirst("sub")?.Value;
            // Additional validation (e.g., check user still exists)
            return Task.CompletedTask;
        },
        OnAuthenticationFailed = context =>
        {
            if (context.Exception is SecurityTokenExpiredException)
            {
                context.Response.Headers["X-Token-Expired"] = "true";
            }
            return Task.CompletedTask;
        },
        OnChallenge = context =>
        {
            // Customize 401 response
            context.HandleResponse();
            context.Response.StatusCode = 401;
            context.Response.ContentType = "application/problem+json";
            return context.Response.WriteAsJsonAsync(new ProblemDetails
            {
                Status = 401,
                Title = "Unauthorized",
                Detail = "Valid authentication credentials are required"
            });
        }
    };
});

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
```

### Token Generation Service

```csharp
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Security.Cryptography;

namespace MyApp.Services;

public interface ITokenService
{
    string GenerateAccessToken(User user, IList<string> roles);
    string GenerateRefreshToken();
    ClaimsPrincipal? ValidateExpiredToken(string token);
}

public class TokenService(IOptions<JwtSettings> jwtOptions) : ITokenService
{
    private readonly JwtSettings _jwt = jwtOptions.Value;

    public string GenerateAccessToken(User user, IList<string> roles)
    {
        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, user.Id.ToString()),
            new(JwtRegisteredClaimNames.Email, user.Email),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new(JwtRegisteredClaimNames.Iat,
                DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString(),
                ClaimValueTypes.Integer64),
            new("name", user.FullName),
        };

        // Add role claims
        claims.AddRange(roles.Select(role => new Claim(ClaimTypes.Role, role)));

        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_jwt.Secret));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var token = new JwtSecurityToken(
            issuer: _jwt.Issuer,
            audience: _jwt.Audience,
            claims: claims,
            notBefore: DateTime.UtcNow,
            expires: DateTime.UtcNow.AddMinutes(_jwt.ExpirationMinutes),
            signingCredentials: credentials);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }

    public string GenerateRefreshToken()
    {
        var randomBytes = new byte[64];
        using var rng = RandomNumberGenerator.Create();
        rng.GetBytes(randomBytes);
        return Convert.ToBase64String(randomBytes);
    }

    public ClaimsPrincipal? ValidateExpiredToken(string token)
    {
        var tokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(_jwt.Secret)),
            ValidateIssuer = true,
            ValidIssuer = _jwt.Issuer,
            ValidateAudience = true,
            ValidAudience = _jwt.Audience,
            ValidateLifetime = false // Allow expired tokens for refresh
        };

        var handler = new JwtSecurityTokenHandler();
        var principal = handler.ValidateToken(token, tokenValidationParameters, out var validatedToken);

        if (validatedToken is not JwtSecurityToken jwtToken ||
            !jwtToken.Header.Alg.Equals(SecurityAlgorithms.HmacSha256,
                StringComparison.InvariantCultureIgnoreCase))
        {
            return null;
        }

        return principal;
    }
}
```

### Cookie Authentication

```csharp
builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddCookie(options =>
    {
        options.Cookie.Name = "MyApp.Auth";
        options.Cookie.HttpOnly = true;
        options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
        options.Cookie.SameSite = SameSiteMode.Lax;
        options.ExpireTimeSpan = TimeSpan.FromHours(8);
        options.SlidingExpiration = true;
        options.LoginPath = "/login";
        options.LogoutPath = "/logout";
        options.AccessDeniedPath = "/access-denied";

        options.Events.OnRedirectToLogin = context =>
        {
            // Return 401 for API requests instead of redirecting
            if (context.Request.Path.StartsWithSegments("/api"))
            {
                context.Response.StatusCode = StatusCodes.Status401Unauthorized;
                return Task.CompletedTask;
            }
            context.Response.Redirect(context.RedirectUri);
            return Task.CompletedTask;
        };
    });
```

### Multiple Authentication Schemes

```csharp
builder.Services.AddAuthentication()
    .AddJwtBearer("Bearer", options => { /* JWT config */ })
    .AddCookie("Cookies", options => { /* Cookie config */ })
    .AddOpenIdConnect("oidc", options =>
    {
        options.Authority = "https://auth.example.com";
        options.ClientId = "my-app";
        options.ClientSecret = builder.Configuration["Oidc:ClientSecret"];
        options.ResponseType = "code";
        options.SaveTokens = true;
        options.Scope.Add("openid");
        options.Scope.Add("profile");
        options.Scope.Add("email");
    });

// Policy with specific scheme
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("ApiPolicy", policy =>
    {
        policy.AuthenticationSchemes.Add(JwtBearerDefaults.AuthenticationScheme);
        policy.RequireAuthenticatedUser();
    });

    options.AddPolicy("WebPolicy", policy =>
    {
        policy.AuthenticationSchemes.Add(CookieAuthenticationDefaults.AuthenticationScheme);
        policy.RequireAuthenticatedUser();
    });
});
```

---

## 2. Authorization

### Policy-Based Authorization

```csharp
builder.Services.AddAuthorization(options =>
{
    // Simple authenticated user policy
    options.AddPolicy("Authenticated", policy =>
        policy.RequireAuthenticatedUser());

    // Role-based
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireRole("Admin"));

    options.AddPolicy("ManagerOrAdmin", policy =>
        policy.RequireRole("Manager", "Admin"));

    // Claims-based
    options.AddPolicy("EmailVerified", policy =>
        policy.RequireClaim("email_verified", "true"));

    options.AddPolicy("PremiumUser", policy =>
        policy.RequireClaim("subscription", "premium", "enterprise"));

    // Combined requirements
    options.AddPolicy("SeniorAdmin", policy =>
    {
        policy.RequireRole("Admin");
        policy.RequireClaim("experience_years");
        policy.RequireAssertion(context =>
        {
            var years = context.User.FindFirst("experience_years")?.Value;
            return int.TryParse(years, out var y) && y >= 5;
        });
    });

    // Resource-based (with custom handler)
    options.AddPolicy("CanEditDocument", policy =>
        policy.Requirements.Add(new DocumentEditRequirement()));

    // Minimum age policy
    options.AddPolicy("Over18", policy =>
        policy.Requirements.Add(new MinimumAgeRequirement(18)));

    // Default policy (applies to [Authorize] without a named policy)
    options.DefaultPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .RequireClaim("email_verified", "true")
        .Build();

    // Fallback policy (applies to endpoints without [Authorize] or [AllowAnonymous])
    options.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
});
```

### Custom Authorization Handlers

```csharp
// Requirement
public class MinimumAgeRequirement : IAuthorizationRequirement
{
    public int MinimumAge { get; }
    public MinimumAgeRequirement(int minimumAge) => MinimumAge = minimumAge;
}

// Handler
public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        MinimumAgeRequirement requirement)
    {
        var dateOfBirthClaim = context.User.FindFirst("date_of_birth");

        if (dateOfBirthClaim is null)
        {
            return Task.CompletedTask; // Requirement not met (no claim)
        }

        if (DateTime.TryParse(dateOfBirthClaim.Value, out var dateOfBirth))
        {
            var age = DateTime.Today.Year - dateOfBirth.Year;
            if (dateOfBirth > DateTime.Today.AddYears(-age)) age--;

            if (age >= requirement.MinimumAge)
            {
                context.Succeed(requirement);
            }
        }

        return Task.CompletedTask;
    }
}

// Resource-based authorization
public class DocumentEditRequirement : IAuthorizationRequirement { }

public class DocumentEditHandler : AuthorizationHandler<DocumentEditRequirement, Document>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        DocumentEditRequirement requirement,
        Document resource)
    {
        var userId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;

        // Owner can always edit
        if (resource.OwnerId.ToString() == userId)
        {
            context.Succeed(requirement);
            return Task.CompletedTask;
        }

        // Admin can edit any document
        if (context.User.IsInRole("Admin"))
        {
            context.Succeed(requirement);
            return Task.CompletedTask;
        }

        // Collaborators can edit
        if (resource.Collaborators.Any(c => c.UserId.ToString() == userId))
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}

// Register handlers
builder.Services.AddSingleton<IAuthorizationHandler, MinimumAgeHandler>();
builder.Services.AddSingleton<IAuthorizationHandler, DocumentEditHandler>();

// Use resource-based authorization in a controller
[ApiController]
[Route("api/[controller]")]
public class DocumentsController : ControllerBase
{
    private readonly IAuthorizationService _authorizationService;
    private readonly IDocumentService _documentService;

    public DocumentsController(
        IAuthorizationService authorizationService,
        IDocumentService documentService)
    {
        _authorizationService = authorizationService;
        _documentService = documentService;
    }

    [HttpPut("{id}")]
    public async Task<IActionResult> Update(int id, UpdateDocumentDto dto)
    {
        var document = await _documentService.GetByIdAsync(id);
        if (document is null) return NotFound();

        var authResult = await _authorizationService.AuthorizeAsync(
            User, document, "CanEditDocument");

        if (!authResult.Succeeded)
            return Forbid();

        await _documentService.UpdateAsync(id, dto);
        return NoContent();
    }
}
```

### Applying Authorization

```csharp
// Controller-level
[Authorize]
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    [AllowAnonymous]  // Override controller-level auth
    public ActionResult<IEnumerable<ProductDto>> GetAll() => Ok();

    [HttpPost]
    [Authorize(Roles = "Admin,Manager")]
    public ActionResult<ProductDto> Create(CreateProductDto dto) => Ok();

    [HttpDelete("{id}")]
    [Authorize(Policy = "AdminOnly")]
    public IActionResult Delete(int id) => NoContent();
}

// Minimal API
app.MapGet("/products", GetAllProducts).AllowAnonymous();
app.MapPost("/products", CreateProduct).RequireAuthorization("AdminOnly");
app.MapDelete("/products/{id}", DeleteProduct).RequireAuthorization("AdminOnly");
```

---

## 3. CORS Configuration

```csharp
builder.Services.AddCors(options =>
{
    // Named policy for specific origins
    options.AddPolicy("Production", policy =>
    {
        policy.WithOrigins(
                "https://myapp.com",
                "https://admin.myapp.com")
            .WithMethods("GET", "POST", "PUT", "DELETE", "PATCH")
            .WithHeaders("Content-Type", "Authorization", "X-Correlation-Id")
            .AllowCredentials()
            .SetPreflightMaxAge(TimeSpan.FromMinutes(10));
    });

    // Permissive policy for development
    options.AddPolicy("Development", policy =>
    {
        policy.AllowAnyOrigin()
            .AllowAnyMethod()
            .AllowAnyHeader();
    });

    // Default policy
    options.AddDefaultPolicy(policy =>
    {
        policy.WithOrigins("https://myapp.com")
            .AllowAnyMethod()
            .AllowAnyHeader()
            .AllowCredentials();
    });
});

var app = builder.Build();
app.UseRouting();

// Apply CORS globally
if (app.Environment.IsDevelopment())
    app.UseCors("Development");
else
    app.UseCors("Production");

app.UseAuthentication();
app.UseAuthorization();

// Or per-endpoint
app.MapGet("/api/public", () => "Public").RequireCors("Production");
```

---

## 4. HTTPS and HSTS

```csharp
var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    // HSTS - tells browsers to only connect via HTTPS
    app.UseHsts();
}

// Redirect HTTP to HTTPS
app.UseHttpsRedirection();

// Configure HSTS options
builder.Services.AddHsts(options =>
{
    options.MaxAge = TimeSpan.FromDays(365);
    options.IncludeSubDomains = true;
    options.Preload = true;
    options.ExcludedHosts.Add("localhost");
});

// Configure HTTPS redirection
builder.Services.AddHttpsRedirection(options =>
{
    options.RedirectStatusCode = StatusCodes.Status308PermanentRedirect;
    options.HttpsPort = 443;
});
```

---

## 5. Rate Limiting (.NET 8+)

```csharp
using System.Threading.RateLimiting;

builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;

    // Global rate limit handler
    options.OnRejected = async (context, ct) =>
    {
        context.HttpContext.Response.ContentType = "application/problem+json";
        await context.HttpContext.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = 429,
            Title = "Too Many Requests",
            Detail = "Rate limit exceeded. Please try again later.",
            Extensions = { ["retryAfter"] = context.Lease.TryGetMetadata(
                MetadataName.RetryAfter, out var retryAfter) ? retryAfter.TotalSeconds : 60 }
        }, ct);
    };

    // Fixed window - N requests per time window
    options.AddFixedWindowLimiter("fixed", opt =>
    {
        opt.PermitLimit = 100;
        opt.Window = TimeSpan.FromMinutes(1);
        opt.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        opt.QueueLimit = 10;
    });

    // Sliding window - smoother rate limiting
    options.AddSlidingWindowLimiter("sliding", opt =>
    {
        opt.PermitLimit = 100;
        opt.Window = TimeSpan.FromMinutes(1);
        opt.SegmentsPerWindow = 6; // 10-second segments
        opt.QueueLimit = 5;
    });

    // Token bucket - allows burst traffic
    options.AddTokenBucketLimiter("token-bucket", opt =>
    {
        opt.TokenLimit = 100;           // Max tokens (burst capacity)
        opt.ReplenishmentPeriod = TimeSpan.FromSeconds(10);
        opt.TokensPerPeriod = 20;       // Tokens added each period
        opt.AutoReplenishment = true;
        opt.QueueLimit = 10;
    });

    // Concurrency limiter - max simultaneous requests
    options.AddConcurrencyLimiter("concurrent", opt =>
    {
        opt.PermitLimit = 50;           // Max concurrent requests
        opt.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        opt.QueueLimit = 25;
    });

    // Per-user rate limiting using a partitioned limiter
    options.AddPolicy("per-user", context =>
    {
        var userId = context.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value
            ?? context.Connection.RemoteIpAddress?.ToString()
            ?? "anonymous";

        return RateLimitPartition.GetFixedWindowLimiter(userId, _ =>
            new FixedWindowRateLimiterOptions
            {
                PermitLimit = 60,
                Window = TimeSpan.FromMinutes(1)
            });
    });
});

var app = builder.Build();
app.UseRateLimiter();

// Apply rate limiting to endpoints
app.MapGet("/api/products", GetProducts).RequireRateLimiting("fixed");
app.MapPost("/api/auth/login", Login).RequireRateLimiting("per-user");
app.MapPost("/api/upload", Upload).RequireRateLimiting("concurrent");

// Apply to route group
var api = app.MapGroup("/api").RequireRateLimiting("sliding");

// Disable for specific endpoint
api.MapGet("/health", () => "OK").DisableRateLimiting();
```

---

## 6. Anti-Forgery Tokens

```csharp
builder.Services.AddAntiforgery(options =>
{
    options.HeaderName = "X-XSRF-TOKEN"; // Header for SPA applications
    options.Cookie.Name = "XSRF-TOKEN";
    options.Cookie.HttpOnly = false; // Must be readable by JavaScript
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    options.Cookie.SameSite = SameSiteMode.Strict;
});

// Endpoint to get token (for SPA)
app.MapGet("/api/antiforgery/token", (IAntiforgery antiforgery, HttpContext context) =>
{
    var tokens = antiforgery.GetAndStoreTokens(context);
    return Results.Ok(new { token = tokens.RequestToken });
}).AllowAnonymous();

// Apply to all mutating endpoints
app.MapPost("/api/orders", CreateOrder); // Antiforgery validated by default in .NET 8

// Skip antiforgery for specific endpoints
app.MapPost("/api/webhooks", HandleWebhook).DisableAntiforgery();
```

---

## 7. Data Protection API

```csharp
using Microsoft.AspNetCore.DataProtection;

builder.Services.AddDataProtection()
    .PersistKeysToFileSystem(new DirectoryInfo("/keys"))
    .SetApplicationName("MyApp")
    .SetDefaultKeyLifetime(TimeSpan.FromDays(90));

// For distributed apps, persist to shared storage
builder.Services.AddDataProtection()
    .PersistKeysToDbContext<DataProtectionKeyContext>()
    .ProtectKeysWithCertificate(certificate);

// Usage in services
public class SensitiveDataService(IDataProtectionProvider dataProtectionProvider)
{
    private readonly IDataProtector _protector =
        dataProtectionProvider.CreateProtector("SensitiveData.v1");

    public string Protect(string plaintext) => _protector.Protect(plaintext);

    public string Unprotect(string protectedData) => _protector.Unprotect(protectedData);

    // Time-limited protection
    public string ProtectWithExpiry(string plaintext, TimeSpan lifetime)
    {
        var timeLimitedProtector = _protector.ToTimeLimitedDataProtector();
        return timeLimitedProtector.Protect(plaintext, lifetime);
    }

    public string? UnprotectWithExpiry(string protectedData)
    {
        var timeLimitedProtector = _protector.ToTimeLimitedDataProtector();
        try
        {
            return timeLimitedProtector.Unprotect(protectedData, out var expiry);
        }
        catch (CryptographicException)
        {
            return null; // Expired or tampered
        }
    }
}
```

---

## 8. Security Headers

```csharp
public class SecurityHeadersMiddleware
{
    private readonly RequestDelegate _next;

    public SecurityHeadersMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        // Prevent MIME type sniffing
        context.Response.Headers["X-Content-Type-Options"] = "nosniff";

        // Prevent clickjacking
        context.Response.Headers["X-Frame-Options"] = "DENY";

        // XSS protection (legacy browsers)
        context.Response.Headers["X-XSS-Protection"] = "1; mode=block";

        // Referrer policy
        context.Response.Headers["Referrer-Policy"] = "strict-origin-when-cross-origin";

        // Permissions policy
        context.Response.Headers["Permissions-Policy"] =
            "camera=(), microphone=(), geolocation=(), payment=()";

        // Content Security Policy
        context.Response.Headers.ContentSecurityPolicy =
            "default-src 'self'; " +
            "script-src 'self' 'nonce-{NONCE}'; " +
            "style-src 'self' 'unsafe-inline'; " +
            "img-src 'self' data: https:; " +
            "font-src 'self'; " +
            "connect-src 'self' https://api.myapp.com; " +
            "frame-ancestors 'none'; " +
            "base-uri 'self'; " +
            "form-action 'self'";

        // Remove server identification headers
        context.Response.Headers.Remove("Server");
        context.Response.Headers.Remove("X-Powered-By");

        await _next(context);
    }
}

app.UseMiddleware<SecurityHeadersMiddleware>();
```

---

## 9. Secret Management Patterns

```csharp
// Development: User Secrets
// dotnet user-secrets set "Jwt:Secret" "my-dev-secret-key-32-chars-long!"

// Production: Environment Variables
// JWT__Secret=production-secret-key

// Production: Azure Key Vault
builder.Configuration.AddAzureKeyVault(
    new Uri("https://my-vault.vault.azure.net/"),
    new DefaultAzureCredential());

// Never log sensitive configuration
builder.Services.AddOptions<JwtSettings>()
    .BindConfiguration("Jwt")
    .ValidateDataAnnotations()
    .ValidateOnStart();

// Validate secrets at startup
var jwtSecret = builder.Configuration["Jwt:Secret"]
    ?? throw new InvalidOperationException("JWT Secret is not configured");

if (jwtSecret.Length < 32)
    throw new InvalidOperationException("JWT Secret must be at least 32 characters");

// Use data protection for encrypting stored data
builder.Services.AddDataProtection()
    .PersistKeysToFileSystem(new DirectoryInfo("/keys"))
    .SetApplicationName("MyApp");
```

---

## 10. OWASP Top 10 Mitigations

| OWASP Risk | ASP.NET Core Mitigation |
|---|---|
| **A01: Broken Access Control** | `[Authorize]`, policy-based auth, resource-based auth |
| **A02: Cryptographic Failures** | Data Protection API, HTTPS enforcement, secure cookie options |
| **A03: Injection** | Parameterized queries (EF Core), input validation, output encoding |
| **A04: Insecure Design** | Rate limiting, CORS policies, threat modeling |
| **A05: Security Misconfiguration** | HSTS, security headers, `ValidateOnStart()` |
| **A06: Vulnerable Components** | `dotnet list package --vulnerable`, Dependabot |
| **A07: Auth Failures** | JWT best practices, account lockout, 2FA |
| **A08: Data Integrity** | Anti-forgery tokens, signed JWTs, SRI for scripts |
| **A09: Logging Failures** | Structured logging, security event logging |
| **A10: SSRF** | Input validation, URL allowlisting, `HttpClient` factory |

### Injection Prevention

```csharp
// NEVER: String concatenation for queries
// var sql = $"SELECT * FROM Users WHERE Id = {id}"; // SQL injection!

// ALWAYS: Parameterized queries with EF Core
var user = await _context.Users
    .Where(u => u.Id == id)
    .FirstOrDefaultAsync();

// Raw SQL with parameters
var users = await _context.Users
    .FromSqlInterpolated($"SELECT * FROM Users WHERE Email = {email}")
    .ToListAsync();

// Input validation
public class CreateUserDto
{
    [Required]
    [StringLength(100)]
    [RegularExpression(@"^[a-zA-Z0-9\s\-\.]+$")]
    public required string Name { get; init; }

    [Required]
    [EmailAddress]
    public required string Email { get; init; }
}
```

### Security Event Logging

```csharp
public class SecurityEventLogger(ILogger<SecurityEventLogger> logger)
{
    public void LogAuthenticationSuccess(string userId, string ipAddress)
    {
        logger.LogInformation(
            "Authentication succeeded for user {UserId} from {IpAddress}",
            userId, ipAddress);
    }

    public void LogAuthenticationFailure(string email, string ipAddress, string reason)
    {
        logger.LogWarning(
            "Authentication failed for {Email} from {IpAddress}: {Reason}",
            email, ipAddress, reason);
    }

    public void LogAuthorizationFailure(string userId, string resource, string policy)
    {
        logger.LogWarning(
            "Authorization denied for user {UserId} on {Resource} with policy {Policy}",
            userId, resource, policy);
    }

    public void LogSuspiciousActivity(string userId, string activity, string details)
    {
        logger.LogError(
            "Suspicious activity by user {UserId}: {Activity} - {Details}",
            userId, activity, details);
    }
}
```

---

## 11. Complete Security Configuration

```csharp
var builder = WebApplication.CreateBuilder(args);

// Authentication
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        var jwt = builder.Configuration.GetSection("Jwt");
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = jwt["Issuer"],
            ValidateAudience = true,
            ValidAudience = jwt["Audience"],
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(jwt["Secret"]!)),
            ValidateLifetime = true,
            ClockSkew = TimeSpan.FromMinutes(1)
        };
    });

// Authorization
builder.Services.AddAuthorizationBuilder()
    .AddPolicy("Admin", policy => policy.RequireRole("Admin"))
    .AddPolicy("Manager", policy => policy.RequireRole("Admin", "Manager"))
    .AddPolicy("EmailVerified", policy => policy.RequireClaim("email_verified", "true"));

// CORS
builder.Services.AddCors(options =>
{
    options.AddPolicy("Api", policy =>
    {
        policy.WithOrigins("https://myapp.com")
            .AllowAnyMethod()
            .AllowAnyHeader()
            .AllowCredentials();
    });
});

// Rate limiting
builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = 429;
    options.AddFixedWindowLimiter("api", opt =>
    {
        opt.PermitLimit = 100;
        opt.Window = TimeSpan.FromMinutes(1);
    });
    options.AddPolicy("per-user", context =>
        RateLimitPartition.GetFixedWindowLimiter(
            context.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value ?? "anon",
            _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 30,
                Window = TimeSpan.FromMinutes(1)
            }));
});

// HSTS
builder.Services.AddHsts(options =>
{
    options.MaxAge = TimeSpan.FromDays(365);
    options.IncludeSubDomains = true;
    options.Preload = true;
});

var app = builder.Build();

// Middleware pipeline (order matters)
app.UseMiddleware<SecurityHeadersMiddleware>();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseRouting();
app.UseCors("Api");
app.UseAuthentication();
app.UseAuthorization();
app.UseRateLimiter();

app.MapControllers();
app.Run();
```

---

## Best Practices

1. **Always validate JWTs fully** - Validate issuer, audience, signing key, and lifetime.
2. **Use policy-based authorization** - Prefer policies over raw role checks for flexibility.
3. **Configure CORS restrictively** - Never use `AllowAnyOrigin` with `AllowCredentials` in production.
4. **Enable HSTS in production** - Use `UseHsts()` with `IncludeSubDomains` and `Preload`.
5. **Apply rate limiting per user** - Use partitioned rate limiters keyed on user identity or IP.
6. **Set security headers** - Use middleware to add CSP, X-Frame-Options, and other security headers.
7. **Log security events** - Log authentication failures, authorization denials, and suspicious activity.
8. **Rotate secrets** - Use Azure Key Vault or AWS Secrets Manager with regular rotation.
9. **Validate on startup** - Use `ValidateOnStart()` for security configuration to fail fast.
10. **Use `ClockSkew` wisely** - Reduce from default 5 minutes to 1 minute or less.

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---|---|---|
| Default `ClockSkew` of 5 min | Expired tokens accepted for too long | Set `ClockSkew = TimeSpan.FromMinutes(1)` |
| `AllowAnyOrigin` + `AllowCredentials` | CORS spec violation, browsers reject | Use specific origins with credentials |
| Secrets in `appsettings.json` | Secrets committed to source control | Use User Secrets, env vars, or Key Vault |
| Missing `UseAuthentication` | `[Authorize]` always returns 401 | Add `UseAuthentication()` before `UseAuthorization()` |
| Wrong middleware order | Auth bypassed or CORS rejected | Follow documented middleware order exactly |
| Missing `HttpOnly` on auth cookies | XSS can steal authentication cookies | Set `Cookie.HttpOnly = true` |
| No rate limiting on auth endpoints | Brute force attacks possible | Apply strict rate limits to login/register |
| Logging sensitive data | Tokens or passwords in logs | Never log credentials, tokens, or PII |
