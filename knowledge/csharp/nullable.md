# C# Nullable Reference Types and Null Safety

> Official Documentation: https://learn.microsoft.com/dotnet/csharp/nullable-references

## Table of Contents

1. [Nullable Reference Types Feature](#nullable-reference-types-feature)
2. [Enabling NRTs](#enabling-nrts)
3. [Nullable Annotations](#nullable-annotations)
4. [Null-Conditional Operators](#null-conditional-operators)
5. [Null-Coalescing Operators](#null-coalescing-operators)
6. [Null-Forgiving Operator](#null-forgiving-operator)
7. [MemberNotNull and MemberNotNullWhen](#membernotnull-and-membernotnullwhen)
8. [Flow Analysis Attributes](#flow-analysis-attributes)
9. [AllowNull and DisallowNull](#allownull-and-disallownull)
10. [Guard Clause Patterns](#guard-clause-patterns)
11. [Required Members](#required-members)
12. [Nullable Value Types](#nullable-value-types)
13. [Nullable in Generics](#nullable-in-generics)
14. [Pattern Matching for Null Checks](#pattern-matching-for-null-checks)
15. [Migration Strategies](#migration-strategies)
16. [Best Practices and Common Pitfalls](#best-practices-and-common-pitfalls)

---

## Nullable Reference Types Feature

Nullable Reference Types (NRTs) add static flow analysis to detect potential null reference exceptions at compile time. When enabled, all reference types are non-nullable by default.

```csharp
#nullable enable

string name = "John";    // Non-nullable: cannot be null
string? nickname = null;  // Nullable: explicitly allows null

// Compiler warning: possible null reference
Console.WriteLine(nickname.Length); // Warning CS8602

// Safe: null check first
if (nickname is not null)
{
    Console.WriteLine(nickname.Length); // No warning
}
```

### Without NRTs (Default before .NET 6)

```csharp
#nullable disable

string name = null;      // No warning - any reference can be null
string nickname = null;   // No warning - null is always allowed
```

---

## Enabling NRTs

### Project-Level (Recommended)

```xml
<!-- In .csproj file -->
<PropertyGroup>
    <Nullable>enable</Nullable>
</PropertyGroup>
```

### File-Level Directives

```csharp
// Enable for this file
#nullable enable

// Disable for this file
#nullable disable

// Restore to project setting
#nullable restore

// Enable only annotations (no warnings)
#nullable enable annotations

// Enable only warnings (no annotations)
#nullable enable warnings
```

### .NET 6+ Default

Starting with .NET 6, new project templates have `<Nullable>enable</Nullable>` by default.

| .NET Version | NRT Default | Notes |
|-------------|-------------|-------|
| .NET 5 and earlier | Disabled | Must opt-in explicitly |
| .NET 6+ | Enabled | Default in new templates |
| .NET 8+ | Enabled | Strongly recommended |

---

## Nullable Annotations

### The `?` Annotation

```csharp
// Non-nullable (default when NRTs enabled)
public class UserService
{
    private readonly ILogger _logger;      // Must never be null
    private readonly string _connectionString; // Must never be null

    // Nullable parameters and returns
    public User? FindByEmail(string email)  // May return null
    {
        // ...
    }

    // Nullable properties
    public string? MiddleName { get; set; } // Can be null

    // Non-nullable collection with nullable elements
    public List<string?> Tags { get; set; } = new();

    // Nullable collection of non-nullable elements
    public List<string>? OptionalTags { get; set; }
}
```

### Nullable in Method Signatures

```csharp
public class ProductService
{
    // Non-nullable parameter, nullable return
    public async Task<Product?> GetByIdAsync(int id)
    {
        return await context.Products.FindAsync(id); // May be null
    }

    // Nullable parameter
    public async Task<List<Product>> SearchAsync(
        string? name = null,
        decimal? minPrice = null,
        decimal? maxPrice = null)
    {
        IQueryable<Product> query = context.Products;

        if (name is not null)
            query = query.Where(p => p.Name.Contains(name));

        if (minPrice.HasValue)
            query = query.Where(p => p.Price >= minPrice.Value);

        if (maxPrice.HasValue)
            query = query.Where(p => p.Price <= maxPrice.Value);

        return await query.ToListAsync();
    }
}
```

---

## Null-Conditional Operators

### `?.` (Member Access)

```csharp
string? name = GetName();

// Without null-conditional
int length = name != null ? name.Length : 0;

// With null-conditional
int? length = name?.Length;          // Returns int? (null if name is null)
int safeLength = name?.Length ?? 0;  // Coalesced to int

// Chained null-conditional
string? city = customer?.Address?.City;

// Method invocation
customer?.Orders?.Clear();

// Delegate invocation
EventHandler? handler = OnDataReceived;
handler?.Invoke(this, EventArgs.Empty);
```

### `?[]` (Element Access)

```csharp
int[]? numbers = GetNumbers();

// Safe element access
int? first = numbers?[0];

// Chained
string? firstChar = names?[0]?.ToUpper();

// Dictionary
Dictionary<string, string>? headers = GetHeaders();
string? contentType = headers?["Content-Type"];
```

### Chaining Examples

```csharp
// Complex chain
var displayName = user?.Profile?.DisplayName?.Trim();

// With method calls
var upperCity = order?.Customer?.Address?.City?.ToUpper();

// Safe LINQ
var firstOrder = customer?.Orders?.FirstOrDefault();
var total = customer?.Orders?.Sum(o => o.Total);

// Event invocation pattern
public event EventHandler<DataEventArgs>? DataReceived;

protected virtual void OnDataReceived(DataEventArgs e)
{
    DataReceived?.Invoke(this, e);
}
```

---

## Null-Coalescing Operators

### `??` (Null-Coalescing)

```csharp
// Provide default when null
string name = GetName() ?? "Anonymous";
int count = GetNullableCount() ?? 0;
List<string> items = GetItems() ?? [];

// Chain multiple fallbacks
string config = Environment.GetEnvironmentVariable("MY_CONFIG")
    ?? configuration["MyConfig"]
    ?? "default-value";

// With throw
string requiredValue = GetValue()
    ?? throw new InvalidOperationException("Value is required");

// Lazy evaluation (right side only evaluated if left is null)
User user = cachedUser ?? await LoadUserAsync(userId);
```

### `??=` (Null-Coalescing Assignment)

```csharp
// Assign only if null
string? name = null;
name ??= "Default";       // name is now "Default"
name ??= "Other";         // name is still "Default" (not null)

// Lazy initialization
private List<string>? _items;
public List<string> Items => _items ??= new List<string>();

// In methods
public void AddTag(string? tag)
{
    tag ??= "untagged";
    _tags.Add(tag);
}

// Complex lazy init
private Dictionary<string, IService>? _services;

public IService GetService(string name)
{
    _services ??= LoadServices();
    return _services[name];
}
```

---

## Null-Forgiving Operator

The `!` (null-forgiving / null-suppression) operator suppresses nullable warnings. Use sparingly.

```csharp
// Suppress warning when you know better than the compiler
string? nullable = GetValue();

// You've verified externally that this won't be null
string nonNull = nullable!; // No warning

// Common legitimate uses:

// 1. After FindAsync in EF Core (when you've already checked)
var product = await context.Products.FindAsync(id);
if (product is null) return NotFound();
ProcessProduct(product); // product is not null here (compiler knows)

// 2. Deserialization where you know the structure
var config = JsonSerializer.Deserialize<Config>(json)!;

// 3. Test assertions (you're about to assert anyway)
var result = await service.GetAsync(id);
Assert.NotNull(result);
Assert.Equal("Expected", result!.Name);

// 4. DI-injected properties (null-forgiving on DbSet)
public DbSet<Product> Products { get; set; } = null!;
```

### When NOT to Use `!`

```csharp
// BAD: hiding a real bug
string? name = GetName();
Console.WriteLine(name!.Length); // May throw NullReferenceException!

// GOOD: handle the null
string? name = GetName();
if (name is not null)
{
    Console.WriteLine(name.Length);
}

// GOOD: provide default
Console.WriteLine((name ?? "Unknown").Length);
```

---

## MemberNotNull and MemberNotNullWhen

These attributes tell the compiler that a member is guaranteed non-null after a method call.

```csharp
using System.Diagnostics.CodeAnalysis;

public class UserSession
{
    public string? Username { get; private set; }
    public string? Token { get; private set; }

    [MemberNotNull(nameof(Username), nameof(Token))]
    public void Initialize(string username, string token)
    {
        Username = username ?? throw new ArgumentNullException(nameof(username));
        Token = token ?? throw new ArgumentNullException(nameof(token));
    }

    // After calling Initialize(), Username and Token are guaranteed non-null
    public void DoWork()
    {
        Initialize("admin", "abc123");
        Console.WriteLine(Username.Length);  // No warning
        Console.WriteLine(Token.Length);     // No warning
    }
}

// MemberNotNullWhen - conditional guarantee
public class Connection
{
    public string? ConnectionString { get; private set; }
    public string? Server { get; private set; }

    [MemberNotNullWhen(true, nameof(ConnectionString), nameof(Server))]
    public bool IsConnected { get; private set; }

    public void UseConnection()
    {
        if (IsConnected)
        {
            Console.WriteLine(ConnectionString.Length); // No warning
            Console.WriteLine(Server.Length);           // No warning
        }
    }
}
```

---

## Flow Analysis Attributes

### NotNull and NotNullWhen

```csharp
using System.Diagnostics.CodeAnalysis;

// NotNull: parameter is guaranteed non-null after the method
public static void ThrowIfNull([NotNull] object? value, string paramName)
{
    if (value is null)
        throw new ArgumentNullException(paramName);
    // After this method, caller knows 'value' is not null
}

// NotNullWhen: parameter is non-null when method returns specified bool
public static bool TryParse(
    string? input,
    [NotNullWhen(true)] out string? result)
{
    if (input is not null && input.Length > 0)
    {
        result = input.Trim();
        return true;
    }

    result = null;
    return false;
}

// Usage
if (TryParse(input, out var parsed))
{
    Console.WriteLine(parsed.Length); // No warning - 'parsed' is non-null when true
}
```

### MaybeNull and MaybeNullWhen

```csharp
// MaybeNull: return may be null even if type is non-nullable
[return: MaybeNull]
public T FindOrDefault<T>(string key)
{
    if (_cache.TryGetValue(key, out var value))
        return (T)value;
    return default; // May be null for reference types
}

// MaybeNullWhen: output parameter may be null when method returns specified bool
public bool TryGet<TValue>(
    string key,
    [MaybeNullWhen(false)] out TValue value)
{
    if (_dict.TryGetValue(key, out var obj) && obj is TValue typed)
    {
        value = typed;
        return true;
    }

    value = default;
    return false;
}
```

---

## AllowNull and DisallowNull

```csharp
using System.Diagnostics.CodeAnalysis;

public class Settings
{
    private string _theme = "default";

    // Property is non-nullable (getter never returns null)
    // but setter accepts null (treats it as "default")
    [AllowNull]
    public string Theme
    {
        get => _theme;
        set => _theme = value ?? "default";
    }
}

// Usage
var settings = new Settings();
settings.Theme = null;                   // No warning (AllowNull)
string theme = settings.Theme;           // Guaranteed non-null
Console.WriteLine(theme.Length);          // Safe

// DisallowNull: nullable property that should not receive null
public class Config
{
    [DisallowNull]
    public string? CachedValue
    {
        get => _cache;  // May return null (first access)
        set => _cache = value ?? throw new ArgumentNullException(nameof(value));
    }
    private string? _cache;
}
```

---

## Guard Clause Patterns

### ArgumentNullException.ThrowIfNull (.NET 6+)

```csharp
public class OrderService
{
    private readonly IOrderRepository _repository;
    private readonly ILogger<OrderService> _logger;

    public OrderService(IOrderRepository repository, ILogger<OrderService> logger)
    {
        // .NET 6+ guard clauses
        ArgumentNullException.ThrowIfNull(repository);
        ArgumentNullException.ThrowIfNull(logger);

        _repository = repository;
        _logger = logger;
    }

    public async Task<Order> CreateAsync(CreateOrderRequest request)
    {
        ArgumentNullException.ThrowIfNull(request);
        ArgumentException.ThrowIfNullOrEmpty(request.CustomerId);
        ArgumentException.ThrowIfNullOrWhiteSpace(request.ShippingAddress);
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(request.Quantity);
        ArgumentOutOfRangeException.ThrowIfGreaterThan(request.Quantity, 1000);

        // ... create order
    }
}
```

### Custom Guard Clauses

```csharp
public static class Guard
{
    public static T NotNull<T>([NotNull] T? value, [CallerArgumentExpression(nameof(value))] string? paramName = null)
        where T : class
    {
        ArgumentNullException.ThrowIfNull(value, paramName);
        return value;
    }

    public static string NotNullOrEmpty([NotNull] string? value, [CallerArgumentExpression(nameof(value))] string? paramName = null)
    {
        ArgumentException.ThrowIfNullOrEmpty(value, paramName);
        return value;
    }

    public static T InRange<T>(T value, T min, T max, [CallerArgumentExpression(nameof(value))] string? paramName = null)
        where T : IComparable<T>
    {
        if (value.CompareTo(min) < 0 || value.CompareTo(max) > 0)
            throw new ArgumentOutOfRangeException(paramName, value, $"Must be between {min} and {max}");
        return value;
    }
}

// Usage with primary constructor
public class ProductService(
    IProductRepository repository,
    ILogger<ProductService> logger)
{
    private readonly IProductRepository _repository = Guard.NotNull(repository);
    private readonly ILogger<ProductService> _logger = Guard.NotNull(logger);
}
```

---

## Required Members

The `required` keyword (C# 11+) enforces that properties must be set during initialization.

```csharp
public class User
{
    public required string Email { get; init; }
    public required string Name { get; init; }
    public string? Bio { get; init; }
    public int Age { get; init; }
}

// Must provide required properties
var user = new User
{
    Email = "john@example.com",
    Name = "John Doe"
    // Bio and Age are optional
};

// Compile error: missing required members
// var invalid = new User { Email = "john@example.com" }; // Error: Name is required
```

### Required with Records

```csharp
public record ProductRequest
{
    public required string Name { get; init; }
    public required decimal Price { get; init; }
    public string? Description { get; init; }
}

// Positional records already enforce via constructor
public record Product(string Name, decimal Price, string? Description = null);
```

### SetsRequiredMembers Attribute

```csharp
using System.Diagnostics.CodeAnalysis;

public class Configuration
{
    public required string ConnectionString { get; init; }
    public required string AppName { get; init; }

    [SetsRequiredMembers]
    public Configuration(string connectionString, string appName)
    {
        ConnectionString = connectionString;
        AppName = appName;
    }

    public Configuration() { } // Still requires setting required members
}
```

---

## Nullable Value Types

Nullable value types (`T?` where `T` is a struct) have existed since C# 2.

```csharp
// Nullable value types
int? nullableInt = null;
double? nullableDouble = 3.14;
bool? nullableBool = null;
DateTime? nullableDate = null;

// HasValue and Value
if (nullableInt.HasValue)
{
    int value = nullableInt.Value;
}

// GetValueOrDefault
int withDefault = nullableInt.GetValueOrDefault(0);
int withDefault2 = nullableInt ?? 0; // Same effect

// Nullable arithmetic
int? a = 5;
int? b = null;
int? sum = a + b;  // null (any operation with null is null)

// Comparison
bool isGreater = a > 3;     // true
bool isNull = b > 3;        // false (null comparisons are false)
bool isNullEq = b == null;  // true

// Conversion
int nonNull = 42;
int? nullable = nonNull;     // Implicit conversion (int -> int?)
int back = (int)nullable;    // Explicit cast (int? -> int, throws if null)
int safe = nullable ?? 0;    // Safe conversion
```

### In Entity Models

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }

    // Nullable value types map to nullable columns
    public decimal? DiscountPrice { get; set; }
    public DateTime? DiscontinuedAt { get; set; }
    public int? MinimumAge { get; set; }
}
```

---

## Nullable in Generics

### Constraining to Non-Null

```csharp
// where T : notnull - T cannot be a nullable type
public class Cache<TKey, TValue>
    where TKey : notnull
    where TValue : notnull
{
    private readonly Dictionary<TKey, TValue> _store = new();

    public void Set(TKey key, TValue value) => _store[key] = value;

    public TValue? Get(TKey key) =>
        _store.TryGetValue(key, out var value) ? value : default;
}

// where T : class - reference type (nullable with T?)
public interface IRepository<T> where T : class
{
    Task<T?> FindAsync(int id);           // May return null
    Task<T> GetAsync(int id);             // Never returns null (throws)
    Task<List<T>> GetAllAsync();           // Never returns null
}

// where T : struct - value type
public static T? FirstOrNull<T>(this IEnumerable<T> source, Func<T, bool> predicate)
    where T : struct
{
    foreach (var item in source)
    {
        if (predicate(item))
            return item;
    }
    return null;
}
```

### Default and Nullable in Generics

```csharp
// Using default with nullable
public T? GetOrDefault<T>(string key)
{
    if (_dict.TryGetValue(key, out var value) && value is T typed)
        return typed;
    return default; // null for reference types, 0/false for value types
}

// Explicit nullable constraint (.NET 7+)
public class Result<T> where T : notnull
{
    public T? Value { get; }
    public string? Error { get; }
    public bool IsSuccess => Error is null;

    private Result(T? value, string? error)
    {
        Value = value;
        Error = error;
    }

    public static Result<T> Success(T value) => new(value, null);
    public static Result<T> Failure(string error) => new(default, error);
}
```

---

## Pattern Matching for Null Checks

```csharp
// Recommended null check patterns
if (value is null) { /* handle null */ }
if (value is not null) { /* use value safely */ }

// Type pattern with null check
if (obj is string text)
{
    // text is guaranteed non-null here
    Console.WriteLine(text.Length);
}

// Switch expression with null
string Display(object? value) => value switch
{
    null => "null",
    string s => $"String: {s}",
    int i => $"Int: {i}",
    _ => value.ToString() ?? "unknown"
};

// Property pattern null check
if (customer is { Address: not null, Address.City: not null })
{
    Console.WriteLine(customer.Address.City);
}

// List pattern with null elements
string[] names = ["Alice", null!, "Carol"];
if (names is [var first, null, var last])
{
    Console.WriteLine($"First: {first}, Last: {last}");
}

// Negated null in switch
public string Process(string? input) => input switch
{
    null or "" => "Empty",
    { Length: > 100 } => "Too long",
    _ => $"Processing: {input}"
};
```

---

## Migration Strategies

### Gradual Migration Approach

```xml
<!-- Step 1: Enable annotations only (see warnings without enforcement) -->
<PropertyGroup>
    <Nullable>annotations</Nullable>
</PropertyGroup>

<!-- Step 2: Enable warnings (see all issues) -->
<PropertyGroup>
    <Nullable>enable</Nullable>
    <!-- Optionally treat warnings as errors gradually -->
    <WarningsAsErrors>CS8600;CS8601;CS8602;CS8603</WarningsAsErrors>
</PropertyGroup>
```

### File-by-File Migration

```csharp
// Old file - disable NRTs temporarily
#nullable disable

public class LegacyService
{
    // No nullable warnings in this file
    public string GetName() => null;
}

// New file - NRTs enabled
#nullable enable

public class ModernService
{
    public string? GetName() => null; // Properly annotated
}
```

### Common Warning Codes

| Code | Description | Fix |
|------|-------------|-----|
| CS8600 | Converting null to non-nullable | Add `?` or null check |
| CS8601 | Possible null reference assignment | Add `?` to target type |
| CS8602 | Dereference of possibly null reference | Add null check before access |
| CS8603 | Possible null reference return | Add `?` to return type |
| CS8604 | Possible null reference argument | Add null check before passing |
| CS8618 | Non-nullable property not initialized | Initialize or add `required` |
| CS8625 | Cannot convert null to non-nullable | Add `?` or provide non-null value |

### Migration Checklist

```
1. Enable <Nullable>enable</Nullable> in .csproj
2. Fix constructor initialization (CS8618)
   - Add 'required' keyword
   - Initialize in constructor
   - Add '= string.Empty' or '= null!'
3. Fix method signatures
   - Add '?' to parameters that accept null
   - Add '?' to return types that may return null
4. Fix null dereferences (CS8602)
   - Add null checks
   - Use null-conditional operators
   - Use null-coalescing operators
5. Fix assignments (CS8600, CS8601)
   - Add proper null handling
6. Add flow analysis attributes where needed
7. Run tests to verify no regressions
```

---

## Best Practices and Common Pitfalls

### Best Practices

| Practice | Description |
|----------|-------------|
| Enable NRTs in all new projects | Default since .NET 6 - use it |
| Use `is not null` over `!= null` | Clearer intent, pattern-matching compatible |
| Use `??` for defaults | `name ?? "Unknown"` is concise and clear |
| Use `ArgumentNullException.ThrowIfNull` | Built-in guard clause (.NET 6+) |
| Use `required` for mandatory properties | Compiler-enforced initialization |
| Annotate public API surfaces accurately | Nullable/non-nullable should match actual behavior |
| Use flow analysis attributes | `NotNull`, `NotNullWhen`, etc. help the compiler |
| Return empty collections, not null | `Array.Empty<T>()` or `Enumerable.Empty<T>()` |

### Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Overusing `!` (null-forgiving) | Hides real bugs | Fix the null flow instead |
| `= null!` on every property | Defeats the purpose of NRTs | Use `required` or initialize properly |
| Not checking `FindAsync` returns | May be null, causing NRE | Always null-check `Find` results |
| Ignoring warnings with `#pragma` | Technical debt accumulates | Fix warnings properly |
| `string.Empty` vs `null` ambiguity | Different semantics | Be consistent: `null` = absent, `""` = present but empty |
| Nullable in DTOs without validation | Null passed to non-nullable service | Validate at API boundary |
| Missing `?` on collection elements | `List<string>` vs `List<string?>` | Consider if elements can be null |
| `default` returns null for reference types | `default(T)` where T is a class returns null | Use `default!` or provide actual default |
