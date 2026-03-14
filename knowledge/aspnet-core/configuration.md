# ASP.NET Core Configuration

> Official Documentation: https://learn.microsoft.com/aspnet/core/fundamentals/configuration/

## Overview

ASP.NET Core provides a flexible configuration system that reads settings from multiple sources with a well-defined precedence order. Configuration values are key-value pairs that can be bound to strongly-typed objects using the Options pattern. The system supports JSON files, environment variables, command-line arguments, user secrets, Azure Key Vault, and custom providers.

---

## 1. Configuration Sources Hierarchy

Configuration sources are read in order, with later sources overriding earlier ones.

| Priority | Source | Typical Use |
|---|---|---|
| 1 (lowest) | `appsettings.json` | Default settings |
| 2 | `appsettings.{Environment}.json` | Environment-specific overrides |
| 3 | User secrets (Development only) | Local secrets (not committed) |
| 4 | Environment variables | Production settings, CI/CD |
| 5 (highest) | Command-line arguments | Runtime overrides |

```csharp
// WebApplication.CreateBuilder sets up the default configuration:
var builder = WebApplication.CreateBuilder(args);

// Default sources (in order):
// 1. appsettings.json
// 2. appsettings.{Environment}.json
// 3. User secrets (when Environment = Development)
// 4. Environment variables
// 5. Command-line arguments

// Access configuration
var connectionString = builder.Configuration.GetConnectionString("Default");
var appName = builder.Configuration["App:Name"];
var port = builder.Configuration.GetValue<int>("App:Port", 5000);
```

### Custom Configuration Sources

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Configuration
    .SetBasePath(Directory.GetCurrentDirectory())
    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
    .AddJsonFile($"appsettings.{builder.Environment.EnvironmentName}.json", optional: true, reloadOnChange: true)
    .AddJsonFile("appsettings.local.json", optional: true, reloadOnChange: true) // Local overrides
    .AddEnvironmentVariables(prefix: "MYAPP_")  // Only vars with MYAPP_ prefix
    .AddCommandLine(args);

// In production, add Azure Key Vault
if (!builder.Environment.IsDevelopment())
{
    builder.Configuration.AddAzureKeyVault(
        new Uri($"https://{builder.Configuration["KeyVault:Name"]}.vault.azure.net/"),
        new DefaultAzureCredential());
}
```

---

## 2. Configuration Files

### appsettings.json Structure

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "ConnectionStrings": {
    "Default": "Host=localhost;Database=myapp;Username=postgres;Password=secret",
    "Redis": "localhost:6379"
  },
  "App": {
    "Name": "My Application",
    "Version": "1.0.0",
    "BaseUrl": "https://localhost:5001"
  },
  "Jwt": {
    "Secret": "OVERRIDE-IN-SECRETS",
    "Issuer": "myapp",
    "Audience": "myapp-api",
    "ExpirationMinutes": 60
  },
  "Email": {
    "SmtpHost": "smtp.example.com",
    "SmtpPort": 587,
    "UseSsl": true,
    "FromAddress": "noreply@example.com",
    "FromName": "My App"
  },
  "Storage": {
    "Provider": "Local",
    "LocalPath": "./uploads",
    "MaxFileSizeMb": 50,
    "AllowedExtensions": [ ".jpg", ".png", ".pdf", ".docx" ]
  },
  "RateLimiting": {
    "PermitLimit": 100,
    "WindowSeconds": 60,
    "QueueLimit": 10
  }
}
```

### Environment-Specific Override (appsettings.Production.json)

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning"
    }
  },
  "App": {
    "BaseUrl": "https://api.myapp.com"
  },
  "Storage": {
    "Provider": "S3",
    "BucketName": "myapp-production-uploads",
    "Region": "us-east-1"
  }
}
```

---

## 3. Reading Configuration Values

### Direct Access with IConfiguration

```csharp
public class StartupService(IConfiguration configuration)
{
    public void PrintConfig()
    {
        // String values
        string appName = configuration["App:Name"]!;
        string connectionString = configuration.GetConnectionString("Default")!;

        // Typed values with defaults
        int port = configuration.GetValue<int>("App:Port", 5000);
        bool useSsl = configuration.GetValue<bool>("Email:UseSsl");

        // Sections
        IConfigurationSection jwtSection = configuration.GetSection("Jwt");
        string issuer = jwtSection["Issuer"]!;

        // Arrays
        var extensions = configuration.GetSection("Storage:AllowedExtensions")
            .Get<string[]>();

        // Check if key exists
        bool hasRedis = configuration.GetSection("ConnectionStrings:Redis").Exists();
    }
}
```

### Binding to Objects

```csharp
public class JwtSettings
{
    public required string Secret { get; set; }
    public required string Issuer { get; set; }
    public required string Audience { get; set; }
    public int ExpirationMinutes { get; set; } = 60;
}

// Method 1: Bind()
var jwtSettings = new JwtSettings();
builder.Configuration.GetSection("Jwt").Bind(jwtSettings);

// Method 2: Get<T>()
var jwtSettings2 = builder.Configuration.GetSection("Jwt").Get<JwtSettings>()!;

// Method 3: Configure<T>() with Options pattern (preferred)
builder.Services.Configure<JwtSettings>(
    builder.Configuration.GetSection("Jwt"));
```

---

## 4. Options Pattern

### Registration Variants

```csharp
// Basic Configure<T> - binds to configuration section
builder.Services.Configure<EmailSettings>(
    builder.Configuration.GetSection("Email"));

// Configure with delegate (for computed values)
builder.Services.Configure<EmailSettings>(options =>
{
    options.SmtpHost = "smtp.example.com";
    options.SmtpPort = 587;
});

// PostConfigure - runs after all Configure calls
builder.Services.PostConfigure<EmailSettings>(options =>
{
    // Ensure FromName always has a value
    if (string.IsNullOrEmpty(options.FromName))
        options.FromName = "Default Sender";
});

// AddOptions fluent API (preferred in .NET 8+)
builder.Services.AddOptions<EmailSettings>()
    .BindConfiguration("Email")             // Bind to section
    .ValidateDataAnnotations()              // Validate [Required], [Range], etc.
    .ValidateOnStart();                     // Fail fast at startup

builder.Services.AddOptions<StorageSettings>()
    .BindConfiguration("Storage")
    .Validate(s => s.MaxFileSizeMb <= 100, "Max file size cannot exceed 100MB")
    .Validate(s => s.AllowedExtensions?.Length > 0, "At least one extension required")
    .ValidateOnStart();
```

### IOptions<T> vs IOptionsSnapshot<T> vs IOptionsMonitor<T>

```csharp
// IOptions<T> - Singleton, reads once at startup
public class EmailService(IOptions<EmailSettings> options)
{
    private readonly EmailSettings _settings = options.Value;

    public Task SendAsync(string to, string subject, string body)
    {
        // _settings never changes during app lifetime
        Console.WriteLine($"Sending via {_settings.SmtpHost}:{_settings.SmtpPort}");
        return Task.CompletedTask;
    }
}

// IOptionsSnapshot<T> - Scoped, re-reads per request
// Use when configuration changes between requests (e.g., after deploy)
public class FeatureFlagService(IOptionsSnapshot<FeatureFlags> optionsSnapshot)
{
    public bool IsEnabled(string feature)
    {
        var flags = optionsSnapshot.Value; // Fresh per request
        return feature switch
        {
            "dark-mode" => flags.DarkMode,
            "beta" => flags.BetaFeatures,
            _ => false
        };
    }
}

// IOptionsMonitor<T> - Singleton, notifies on changes in real-time
public class DynamicConfigService : IDisposable
{
    private readonly IDisposable? _changeListener;
    private RateLimitingSettings _settings;

    public DynamicConfigService(IOptionsMonitor<RateLimitingSettings> monitor)
    {
        _settings = monitor.CurrentValue;
        _changeListener = monitor.OnChange((newSettings, name) =>
        {
            _settings = newSettings;
            Console.WriteLine($"Rate limiting updated: {newSettings.PermitLimit} req/{newSettings.WindowSeconds}s");
        });
    }

    public int GetPermitLimit() => _settings.PermitLimit;

    public void Dispose() => _changeListener?.Dispose();
}
```

### Named Options

```csharp
// appsettings.json
// {
//   "Storage": {
//     "Images": { "Provider": "S3", "BucketName": "images", "MaxFileSizeMb": 10 },
//     "Documents": { "Provider": "Azure", "Container": "docs", "MaxFileSizeMb": 50 }
//   }
// }

builder.Services.Configure<StorageSettings>("Images",
    builder.Configuration.GetSection("Storage:Images"));
builder.Services.Configure<StorageSettings>("Documents",
    builder.Configuration.GetSection("Storage:Documents"));

// Access named options
public class FileUploadService(IOptionsSnapshot<StorageSettings> optionsSnapshot)
{
    public async Task<string> UploadImageAsync(Stream file, string fileName)
    {
        var settings = optionsSnapshot.Get("Images");
        // Use Images storage settings
        return $"Uploaded to {settings.Provider}/{settings.BucketName}/{fileName}";
    }

    public async Task<string> UploadDocumentAsync(Stream file, string fileName)
    {
        var settings = optionsSnapshot.Get("Documents");
        // Use Documents storage settings
        return $"Uploaded to {settings.Provider}/{settings.Container}/{fileName}";
    }
}
```

---

## 5. Options Validation

### Data Annotations Validation

```csharp
using System.ComponentModel.DataAnnotations;

public class JwtSettings
{
    public const string SectionName = "Jwt";

    [Required(ErrorMessage = "JWT Secret is required")]
    [MinLength(32, ErrorMessage = "JWT Secret must be at least 32 characters")]
    public required string Secret { get; set; }

    [Required]
    public required string Issuer { get; set; }

    [Required]
    public required string Audience { get; set; }

    [Range(5, 1440, ErrorMessage = "Expiration must be between 5 and 1440 minutes")]
    public int ExpirationMinutes { get; set; } = 60;

    [Range(1, 43200, ErrorMessage = "Refresh expiration must be between 1 and 43200 minutes")]
    public int RefreshExpirationMinutes { get; set; } = 10080;
}

builder.Services.AddOptions<JwtSettings>()
    .BindConfiguration(JwtSettings.SectionName)
    .ValidateDataAnnotations()
    .ValidateOnStart(); // App will not start if validation fails
```

### Custom IValidateOptions<T>

```csharp
public class DatabaseSettingsValidator : IValidateOptions<DatabaseSettings>
{
    public ValidateOptionsResult Validate(string? name, DatabaseSettings options)
    {
        var failures = new List<string>();

        if (string.IsNullOrWhiteSpace(options.ConnectionString))
            failures.Add("ConnectionString is required");

        if (options.MaxRetryCount < 0 || options.MaxRetryCount > 10)
            failures.Add("MaxRetryCount must be between 0 and 10");

        if (options.CommandTimeoutSeconds < 1 || options.CommandTimeoutSeconds > 300)
            failures.Add("CommandTimeoutSeconds must be between 1 and 300");

        if (options.EnableSensitiveDataLogging && options.Environment == "Production")
            failures.Add("SensitiveDataLogging must not be enabled in Production");

        return failures.Count > 0
            ? ValidateOptionsResult.Fail(failures)
            : ValidateOptionsResult.Success;
    }
}

builder.Services.AddSingleton<IValidateOptions<DatabaseSettings>, DatabaseSettingsValidator>();

builder.Services.AddOptions<DatabaseSettings>()
    .BindConfiguration("Database")
    .ValidateOnStart();
```

---

## 6. User Secrets (Secret Manager)

User secrets store sensitive configuration outside the project directory during development. Secrets are stored in `%APPDATA%\Microsoft\UserSecrets\<user_secrets_id>\secrets.json` on Windows.

```bash
# Initialize user secrets for a project
dotnet user-secrets init

# Set secrets
dotnet user-secrets set "Jwt:Secret" "my-super-secret-key-that-is-long-enough-for-jwt"
dotnet user-secrets set "ConnectionStrings:Default" "Host=localhost;Database=myapp;Username=admin;Password=P@ssw0rd!"
dotnet user-secrets set "Email:ApiKey" "SG.xxxxxxxxxxxx"

# Set hierarchical values
dotnet user-secrets set "ExternalApis:GitHub:ClientId" "abc123"
dotnet user-secrets set "ExternalApis:GitHub:ClientSecret" "secret456"

# List all secrets
dotnet user-secrets list

# Remove a secret
dotnet user-secrets remove "Email:ApiKey"

# Clear all secrets
dotnet user-secrets clear
```

```xml
<!-- .csproj - UserSecretsId is added automatically by dotnet user-secrets init -->
<PropertyGroup>
  <TargetFramework>net8.0</TargetFramework>
  <UserSecretsId>a1b2c3d4-e5f6-7890-abcd-ef1234567890</UserSecretsId>
</PropertyGroup>
```

```csharp
// User secrets are automatically loaded in Development environment
var builder = WebApplication.CreateBuilder(args);

// The following is done automatically by CreateBuilder:
// if (builder.Environment.IsDevelopment())
//     builder.Configuration.AddUserSecrets<Program>();
```

---

## 7. Environment Variables

```csharp
// Environment variables use __ (double underscore) or : as section separator
// MYAPP__Jwt__Secret maps to configuration["Jwt:Secret"]

// Only load vars with a prefix
builder.Configuration.AddEnvironmentVariables(prefix: "MYAPP_");
// MYAPP_Jwt__Secret maps to configuration["Jwt:Secret"]
// The prefix is stripped from the key
```

### Common Environment Variable Patterns

| Environment Variable | Configuration Key | Value |
|---|---|---|
| `ConnectionStrings__Default` | `ConnectionStrings:Default` | Connection string |
| `Jwt__Secret` | `Jwt:Secret` | JWT signing key |
| `ASPNETCORE_ENVIRONMENT` | `Environment` | Development, Staging, Production |
| `ASPNETCORE_URLS` | `Urls` | `https://+:5001;http://+:5000` |
| `Logging__LogLevel__Default` | `Logging:LogLevel:Default` | Information, Warning, Error |

```bash
# Docker / docker-compose.yml
services:
  api:
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ConnectionStrings__Default=Host=db;Database=myapp;Username=postgres;Password=secret
      - Jwt__Secret=production-secret-key-that-is-very-long
      - ASPNETCORE_URLS=http://+:8080

# Linux/macOS shell
export ConnectionStrings__Default="Host=localhost;Database=myapp"
export Jwt__Secret="my-secret-key"

# Windows PowerShell
$env:ConnectionStrings__Default="Host=localhost;Database=myapp"
$env:Jwt__Secret="my-secret-key"
```

---

## 8. Key-Per-File Configuration

Useful in Kubernetes where secrets are mounted as files in a directory.

```csharp
// Each file in the directory becomes a configuration key
// File name = key, file content = value
builder.Configuration.AddKeyPerFile("/run/secrets", optional: true);

// /run/secrets/
//   ConnectionStrings__Default    -> contains: "Host=db;Database=myapp"
//   Jwt__Secret                   -> contains: "super-secret-key"
//   Email__ApiKey                 -> contains: "SG.xxx"
```

---

## 9. Azure Key Vault Integration

```csharp
// Install: Azure.Extensions.AspNetCore.Configuration.Secrets
//          Azure.Identity

var builder = WebApplication.CreateBuilder(args);

if (!builder.Environment.IsDevelopment())
{
    var keyVaultUrl = builder.Configuration["KeyVault:Url"]
        ?? throw new InvalidOperationException("KeyVault:Url is not configured");

    builder.Configuration.AddAzureKeyVault(
        new Uri(keyVaultUrl),
        new DefaultAzureCredential(),
        new AzureKeyVaultConfigurationOptions
        {
            ReloadInterval = TimeSpan.FromMinutes(5) // Poll for changes
        });
}

// Key Vault secret names use -- instead of : for hierarchy
// Secret name: "Jwt--Secret" maps to configuration["Jwt:Secret"]
// Secret name: "ConnectionStrings--Default" maps to connectionStrings["Default"]

// Custom key mapping
builder.Configuration.AddAzureKeyVault(
    new Uri(keyVaultUrl),
    new DefaultAzureCredential(),
    new PrefixKeyVaultSecretManager("MyApp"));

public class PrefixKeyVaultSecretManager : KeyVaultSecretManager
{
    private readonly string _prefix;

    public PrefixKeyVaultSecretManager(string prefix)
    {
        _prefix = $"{prefix}-";
    }

    public override bool Load(SecretProperties properties)
    {
        return properties.Name.StartsWith(_prefix, StringComparison.OrdinalIgnoreCase);
    }

    public override string GetKey(KeyVaultSecret secret)
    {
        return secret.Name[_prefix.Length..]
            .Replace("--", ConfigurationPath.KeyDelimiter);
    }
}
```

---

## 10. Custom Configuration Provider

```csharp
// Custom provider that reads from a database
public class DatabaseConfigurationSource : IConfigurationSource
{
    public string ConnectionString { get; set; } = default!;
    public TimeSpan ReloadInterval { get; set; } = TimeSpan.FromMinutes(5);

    public IConfigurationProvider Build(IConfigurationBuilder builder)
    {
        return new DatabaseConfigurationProvider(this);
    }
}

public class DatabaseConfigurationProvider : ConfigurationProvider, IDisposable
{
    private readonly DatabaseConfigurationSource _source;
    private readonly Timer _reloadTimer;

    public DatabaseConfigurationProvider(DatabaseConfigurationSource source)
    {
        _source = source;
        _reloadTimer = new Timer(
            _ => LoadAsync().GetAwaiter().GetResult(),
            null,
            source.ReloadInterval,
            source.ReloadInterval);
    }

    public override void Load()
    {
        LoadAsync().GetAwaiter().GetResult();
    }

    private async Task LoadAsync()
    {
        await using var connection = new NpgsqlConnection(_source.ConnectionString);
        await connection.OpenAsync();

        await using var command = new NpgsqlCommand(
            "SELECT key, value FROM app_configuration WHERE is_active = true",
            connection);

        await using var reader = await command.ExecuteReaderAsync();
        var data = new Dictionary<string, string?>(StringComparer.OrdinalIgnoreCase);

        while (await reader.ReadAsync())
        {
            data[reader.GetString(0)] = reader.GetString(1);
        }

        Data = data;
        OnReload(); // Notify IOptionsMonitor subscribers
    }

    public void Dispose() => _reloadTimer?.Dispose();
}

// Extension method
public static class DatabaseConfigurationExtensions
{
    public static IConfigurationBuilder AddDatabaseConfiguration(
        this IConfigurationBuilder builder,
        string connectionString,
        TimeSpan? reloadInterval = null)
    {
        builder.Add(new DatabaseConfigurationSource
        {
            ConnectionString = connectionString,
            ReloadInterval = reloadInterval ?? TimeSpan.FromMinutes(5)
        });

        return builder;
    }
}

// Usage
builder.Configuration.AddDatabaseConfiguration(
    builder.Configuration.GetConnectionString("Config")!,
    reloadInterval: TimeSpan.FromMinutes(2));
```

---

## 11. Configuration in Program.cs

```csharp
var builder = WebApplication.CreateBuilder(args);

// Configure services using configuration
var connectionString = builder.Configuration.GetConnectionString("Default")
    ?? throw new InvalidOperationException("Connection string 'Default' not found");

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(connectionString));

// Register options
builder.Services.AddOptions<JwtSettings>()
    .BindConfiguration("Jwt")
    .ValidateDataAnnotations()
    .ValidateOnStart();

builder.Services.AddOptions<EmailSettings>()
    .BindConfiguration("Email")
    .ValidateDataAnnotations()
    .ValidateOnStart();

// Use configuration directly for one-time setup
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        var jwtConfig = builder.Configuration.GetSection("Jwt");
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = jwtConfig["Issuer"],
            ValidateAudience = true,
            ValidAudience = jwtConfig["Audience"],
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(jwtConfig["Secret"]!))
        };
    });

var app = builder.Build();

// Access configuration in middleware pipeline
if (app.Configuration.GetValue<bool>("Features:EnableSwagger"))
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.Run();
```

---

## Best Practices

1. **Use the Options pattern** - Always bind configuration to strongly-typed classes rather than reading raw strings.
2. **Validate on start** - Call `ValidateOnStart()` to catch misconfigurations before the app accepts traffic.
3. **Never store secrets in appsettings.json** - Use User Secrets in development, environment variables or Key Vault in production.
4. **Use environment-specific files** - Override only what differs per environment in `appsettings.{Environment}.json`.
5. **Prefix environment variables** - Use `AddEnvironmentVariables(prefix: "MYAPP_")` to avoid conflicts with system variables.
6. **Use `GetConnectionString()`** - It is a shorthand for `GetSection("ConnectionStrings")[name]`.
7. **Fail fast on missing required config** - Use the null-forgiving operator with a throw or `?? throw new InvalidOperationException()`.
8. **Organize by feature** - Group related settings under a common section key.

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---|---|---|
| Secrets in source control | `appsettings.json` with real passwords committed | Use User Secrets and environment variables |
| Missing `ValidateOnStart` | Bad config not detected until first use | Always call `ValidateOnStart()` |
| Wrong section name | `Configure<T>` binds empty object silently | Double-check section names match JSON keys exactly |
| IOptionsSnapshot in Singleton | Cannot inject scoped into singleton | Use `IOptionsMonitor<T>` for singletons |
| Environment variable separator | Using `:` on Linux (not supported) | Use `__` (double underscore) which works everywhere |
| Reload not working | File changes not detected | Ensure `reloadOnChange: true` is set |
| Missing `optional: true` | App crashes when file does not exist | Use `optional: true` for non-essential config files |
| Array binding issues | Arrays in environment variables | Use indexed format: `Storage__AllowedExtensions__0=.jpg` |
