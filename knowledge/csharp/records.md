# C# Records

> Official Documentation: https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/record

## Table of Contents

1. [Record Class (Reference Type)](#record-class-reference-type)
2. [Record Struct (Value Type)](#record-struct-value-type)
3. [Positional Records](#positional-records)
4. [With-Expressions](#with-expressions)
5. [Value-Based Equality](#value-based-equality)
6. [ToString Auto-Implementation](#tostring-auto-implementation)
7. [Deconstruction](#deconstruction)
8. [Inheritance with Records](#inheritance-with-records)
9. [Abstract and Sealed Records](#abstract-and-sealed-records)
10. [Records vs Classes vs Structs](#records-vs-classes-vs-structs)
11. [DTOs and API Models](#dtos-and-api-models)
12. [Immutability Patterns](#immutability-patterns)
13. [JSON Serialization](#json-serialization)
14. [Best Practices and Common Pitfalls](#best-practices-and-common-pitfalls)

---

## Record Class (Reference Type)

A `record` (or `record class`) is a reference type with value-based equality semantics and built-in immutability support.

```csharp
// Basic record class (immutable by default with init properties)
public record Person
{
    public required string FirstName { get; init; }
    public required string LastName { get; init; }
    public int Age { get; init; }
}

// Usage
var person = new Person
{
    FirstName = "John",
    LastName = "Doe",
    Age = 30
};

// Cannot modify after construction
// person.FirstName = "Jane"; // Compile error: init-only property
```

### Record with Mutable Properties

```csharp
// Mutable record (use cautiously)
public record MutablePerson
{
    public required string FirstName { get; set; }
    public required string LastName { get; set; }
    public int Age { get; set; }
}
```

---

## Record Struct (Value Type)

`record struct` provides value-based equality with value type semantics. Available since C# 10.

```csharp
// Mutable by default (record struct)
public record struct Point(double X, double Y);

var p = new Point(1.0, 2.0);
p.X = 3.0; // Allowed - record struct properties are mutable by default

// Immutable record struct
public readonly record struct ImmutablePoint(double X, double Y);

var ip = new ImmutablePoint(1.0, 2.0);
// ip.X = 3.0; // Compile error: readonly
```

### Record Struct with Body

```csharp
public readonly record struct Temperature
{
    public double Celsius { get; init; }
    public double Fahrenheit => Celsius * 9.0 / 5.0 + 32.0;
    public double Kelvin => Celsius + 273.15;

    public Temperature(double celsius) => Celsius = celsius;

    public static Temperature FromFahrenheit(double f) =>
        new((f - 32.0) * 5.0 / 9.0);
}

var temp = new Temperature(100);
Console.WriteLine($"{temp.Celsius}C = {temp.Fahrenheit}F = {temp.Kelvin}K");
// 100C = 212F = 373.15K
```

---

## Positional Records

Positional records use primary constructor syntax for concise definition. The compiler generates `init` properties, a constructor, and `Deconstruct`.

```csharp
// Positional record class (most common form)
public record Person(string FirstName, string LastName, int Age);

// Equivalent hand-written code:
public record Person
{
    public string FirstName { get; init; }
    public string LastName { get; init; }
    public int Age { get; init; }

    public Person(string firstName, string lastName, int age)
    {
        FirstName = firstName;
        LastName = lastName;
        Age = age;
    }

    public void Deconstruct(out string firstName, out string lastName, out int age)
    {
        firstName = FirstName;
        lastName = LastName;
        age = Age;
    }
}

// Usage
var person = new Person("John", "Doe", 30);
Console.WriteLine(person.FirstName); // John
```

### Positional Record with Additional Members

```csharp
public record Product(string Name, decimal Price, string Category)
{
    // Additional computed property
    public string DisplayPrice => $"${Price:F2}";

    // Additional method
    public Product ApplyDiscount(decimal percentage) =>
        this with { Price = Price * (1 - percentage / 100) };

    // Override generated property (make it mutable)
    public decimal Price { get; set; } = Price;
}
```

### Positional Record Struct

```csharp
public readonly record struct Coordinate(double Latitude, double Longitude)
{
    public double DistanceTo(Coordinate other)
    {
        var dLat = (other.Latitude - Latitude) * Math.PI / 180;
        var dLon = (other.Longitude - Longitude) * Math.PI / 180;
        var a = Math.Sin(dLat / 2) * Math.Sin(dLat / 2) +
                Math.Cos(Latitude * Math.PI / 180) * Math.Cos(other.Latitude * Math.PI / 180) *
                Math.Sin(dLon / 2) * Math.Sin(dLon / 2);
        return 6371 * 2 * Math.Atan2(Math.Sqrt(a), Math.Sqrt(1 - a)); // km
    }
}
```

---

## With-Expressions

The `with` expression creates a copy of a record with specified properties changed (non-destructive mutation).

```csharp
public record Person(string FirstName, string LastName, int Age);

var john = new Person("John", "Doe", 30);

// Create a modified copy
var jane = john with { FirstName = "Jane" };
// jane = Person { FirstName = "Jane", LastName = "Doe", Age = 30 }

// Multiple property changes
var olderJohn = john with { Age = 31, LastName = "Smith" };

// Copy with no changes (shallow clone)
var clone = john with { };
Console.WriteLine(ReferenceEquals(john, clone)); // false
Console.WriteLine(john == clone);                // true (value equality)
```

### With-Expressions and Nested Records

```csharp
public record Address(string Street, string City, string ZipCode);
public record Customer(string Name, Address Address);

var customer = new Customer("John", new Address("123 Main", "Seattle", "98101"));

// Replace the entire nested record
var moved = customer with
{
    Address = customer.Address with { City = "Portland", ZipCode = "97201" }
};
```

### With-Expressions on Record Structs

```csharp
public readonly record struct Money(decimal Amount, string Currency);

var price = new Money(29.99m, "USD");
var eurPrice = price with { Amount = 27.50m, Currency = "EUR" };
```

---

## Value-Based Equality

Records implement value-based equality: two records are equal if all their properties are equal.

```csharp
public record Person(string FirstName, string LastName, int Age);

var p1 = new Person("John", "Doe", 30);
var p2 = new Person("John", "Doe", 30);
var p3 = new Person("Jane", "Doe", 25);

Console.WriteLine(p1 == p2);             // true (value equality)
Console.WriteLine(p1 != p3);             // true
Console.WriteLine(p1.Equals(p2));        // true
Console.WriteLine(p1.GetHashCode() == p2.GetHashCode()); // true
Console.WriteLine(ReferenceEquals(p1, p2)); // false (different instances)
```

### Custom Equality

```csharp
public record Person(string FirstName, string LastName, int Age)
{
    // Override equality to ignore case
    public virtual bool Equals(Person? other)
    {
        if (other is null) return false;
        return string.Equals(FirstName, other.FirstName, StringComparison.OrdinalIgnoreCase)
            && string.Equals(LastName, other.LastName, StringComparison.OrdinalIgnoreCase)
            && Age == other.Age;
    }

    public override int GetHashCode()
    {
        return HashCode.Combine(
            FirstName.ToUpperInvariant(),
            LastName.ToUpperInvariant(),
            Age);
    }
}
```

### Equality with Collections

```csharp
// Records with collection properties compare by REFERENCE, not by contents
public record Team(string Name, List<string> Members);

var t1 = new Team("Alpha", ["Alice", "Bob"]);
var t2 = new Team("Alpha", ["Alice", "Bob"]);
Console.WriteLine(t1 == t2); // false! Different List instances

// Solution: override equality or use immutable collections
public record Team(string Name, ImmutableArray<string> Members);
```

---

## ToString Auto-Implementation

Records generate a `ToString()` that includes all properties in a readable format.

```csharp
public record Person(string FirstName, string LastName, int Age);

var person = new Person("John", "Doe", 30);
Console.WriteLine(person);
// Person { FirstName = John, LastName = Doe, Age = 30 }
```

### Override PrintMembers

```csharp
public record SensitiveUser(string Name, string Email, string SocialSecurityNumber)
{
    // Customize what ToString() outputs
    protected virtual bool PrintMembers(System.Text.StringBuilder builder)
    {
        builder.Append($"Name = {Name}, Email = {Email}, SSN = ***-**-****");
        return true;
    }
}

var user = new SensitiveUser("John", "john@example.com", "123-45-6789");
Console.WriteLine(user);
// SensitiveUser { Name = John, Email = john@example.com, SSN = ***-**-**** }
```

---

## Deconstruction

Positional records automatically generate a `Deconstruct` method.

```csharp
public record Person(string FirstName, string LastName, int Age);

var person = new Person("John", "Doe", 30);

// Deconstruct into variables
var (first, last, age) = person;
Console.WriteLine($"{first} {last}, age {age}");

// Discard unused values
var (name, _, _) = person;

// Use in pattern matching
if (person is ("John", _, > 18))
{
    Console.WriteLine("Adult John");
}

// Use in switch
string greeting = person switch
{
    ("John", _, _) => "Hey John!",
    (_, _, < 18) => "Hello, young person!",
    _ => $"Hello, {person.FirstName}!"
};
```

---

## Inheritance with Records

Records support inheritance (only `record class`, not `record struct`).

```csharp
public record Person(string FirstName, string LastName);

public record Employee(string FirstName, string LastName, string Department, decimal Salary)
    : Person(FirstName, LastName);

public record Manager(string FirstName, string LastName, string Department, decimal Salary, int TeamSize)
    : Employee(FirstName, LastName, Department, Salary);

// Usage
var manager = new Manager("John", "Doe", "Engineering", 150_000m, 10);

// Equality works across hierarchy
Person p1 = new Employee("John", "Doe", "Engineering", 100_000m);
Person p2 = new Employee("John", "Doe", "Engineering", 100_000m);
Console.WriteLine(p1 == p2); // true

Person p3 = new Manager("John", "Doe", "Engineering", 100_000m, 5);
Console.WriteLine(p1 == p3); // false (different runtime types)

// with-expression preserves runtime type
Employee emp = new Manager("John", "Doe", "Engineering", 100_000m, 5);
var promoted = emp with { Salary = 120_000m };
Console.WriteLine(promoted.GetType()); // Manager (not Employee!)
```

---

## Abstract and Sealed Records

```csharp
// Abstract record
public abstract record Shape(string Color)
{
    public abstract double Area { get; }
}

public record Circle(string Color, double Radius) : Shape(Color)
{
    public override double Area => Math.PI * Radius * Radius;
}

public record Rectangle(string Color, double Width, double Height) : Shape(Color)
{
    public override double Area => Width * Height;
}

// Sealed record (cannot be inherited)
public sealed record Configuration(string Key, string Value, DateTime CreatedAt);
```

---

## Records vs Classes vs Structs

| Feature | `record class` | `record struct` | `class` | `struct` |
|---------|---------------|----------------|---------|----------|
| Type | Reference | Value | Reference | Value |
| Equality | Value-based | Value-based | Reference-based | Value-based |
| `with` expression | Yes | Yes | No | No |
| Deconstruction | Positional only | Positional only | Manual | Manual |
| ToString | Auto-generated | Auto-generated | Object.ToString | Object.ToString |
| Inheritance | Yes | No | Yes | No |
| Nullable | Reference null | Not nullable | Reference null | Not nullable |
| Default immutability | `init` props | Mutable (use `readonly`) | Mutable | Mutable |
| Heap/Stack | Heap | Stack (usually) | Heap | Stack (usually) |

### Decision Guide

```
Use record class when:
  - Immutable data model (DTOs, events, messages)
  - Value-based equality needed
  - Inheritance required
  - Data will be shared across threads

Use record struct when:
  - Small data (under ~64 bytes)
  - High-performance / low-allocation scenarios
  - No inheritance needed
  - Stack allocation preferred

Use class when:
  - Mutable state with identity
  - Complex behavior (services, controllers)
  - Entity Framework entities (need change tracking)
  - Long-lived objects

Use struct when:
  - Small value types (Point, Color, DateTime)
  - Performance-critical inner loops
  - Interop with native code
```

---

## DTOs and API Models

Records are ideal for API request/response models.

```csharp
// API request
public record CreateProductRequest(
    string Name,
    decimal Price,
    string Category,
    string? Description = null);

// API response
public record ProductResponse(
    int Id,
    string Name,
    decimal Price,
    string Category,
    string? Description,
    DateTime CreatedAt);

// Pagination response
public record PagedResponse<T>(
    List<T> Items,
    int TotalCount,
    int Page,
    int PageSize)
{
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasNext => Page < TotalPages;
    public bool HasPrevious => Page > 1;
}

// Error response
public record ApiError(string Code, string Message, Dictionary<string, string[]>? Errors = null);

// Controller usage
[ApiController]
[Route("api/[controller]")]
public class ProductsController(AppDbContext context) : ControllerBase
{
    [HttpPost]
    public async Task<ActionResult<ProductResponse>> Create(CreateProductRequest request)
    {
        var product = new Product
        {
            Name = request.Name,
            Price = request.Price,
            Category = request.Category,
            Description = request.Description,
            CreatedAt = DateTime.UtcNow
        };

        context.Products.Add(product);
        await context.SaveChangesAsync();

        var response = new ProductResponse(
            product.Id, product.Name, product.Price,
            product.Category, product.Description, product.CreatedAt);

        return CreatedAtAction(nameof(GetById), new { id = product.Id }, response);
    }

    [HttpGet("{id:int}")]
    public async Task<ActionResult<ProductResponse>> GetById(int id)
    {
        var product = await context.Products.FindAsync(id);
        if (product is null) return NotFound();

        return new ProductResponse(
            product.Id, product.Name, product.Price,
            product.Category, product.Description, product.CreatedAt);
    }
}
```

---

## Immutability Patterns

### Immutable Builder with Records

```csharp
public record EmailMessage(
    string From,
    IReadOnlyList<string> To,
    string Subject,
    string Body,
    IReadOnlyList<string>? Cc = null,
    IReadOnlyList<string>? Bcc = null,
    bool IsHtml = false)
{
    public static EmailMessageBuilder Builder() => new();
}

public class EmailMessageBuilder
{
    private string _from = "";
    private readonly List<string> _to = [];
    private string _subject = "";
    private string _body = "";
    private readonly List<string> _cc = [];
    private readonly List<string> _bcc = [];
    private bool _isHtml;

    public EmailMessageBuilder From(string from) { _from = from; return this; }
    public EmailMessageBuilder To(string to) { _to.Add(to); return this; }
    public EmailMessageBuilder Subject(string subject) { _subject = subject; return this; }
    public EmailMessageBuilder Body(string body) { _body = body; return this; }
    public EmailMessageBuilder Cc(string cc) { _cc.Add(cc); return this; }
    public EmailMessageBuilder Html() { _isHtml = true; return this; }

    public EmailMessage Build() => new(
        _from, _to.AsReadOnly(), _subject, _body,
        _cc.Count > 0 ? _cc.AsReadOnly() : null,
        _bcc.Count > 0 ? _bcc.AsReadOnly() : null,
        _isHtml);
}

// Usage
var email = EmailMessage.Builder()
    .From("sender@example.com")
    .To("recipient@example.com")
    .To("another@example.com")
    .Subject("Hello")
    .Body("<h1>Hi!</h1>")
    .Html()
    .Build();
```

---

## JSON Serialization

### System.Text.Json

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

public record Product(
    int Id,
    string Name,
    decimal Price,
    [property: JsonPropertyName("category_name")]
    string Category,
    [property: JsonIgnore]
    string? InternalNote = null);

// Serialize
var product = new Product(1, "Laptop", 999.99m, "Electronics");
var json = JsonSerializer.Serialize(product, new JsonSerializerOptions
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    WriteIndented = true
});
// { "id": 1, "name": "Laptop", "price": 999.99, "category_name": "Electronics" }

// Deserialize
var deserialized = JsonSerializer.Deserialize<Product>(json, new JsonSerializerOptions
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase
});

// Records with required properties
public record CreateUserRequest(
    [property: JsonRequired] string Email,
    [property: JsonRequired] string Password,
    string? DisplayName = null);
```

### ASP.NET Core Configuration

```csharp
builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
    options.SerializerOptions.DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull;
    options.SerializerOptions.Converters.Add(new JsonStringEnumConverter());
});
```

---

## Best Practices and Common Pitfalls

### Best Practices

| Practice | Description |
|----------|-------------|
| Use positional records for simple DTOs | Concise syntax: `record Person(string Name, int Age)` |
| Prefer `record class` over `record struct` for DTOs | Reference semantics work better with serialization |
| Use `readonly record struct` for small value types | Guarantees immutability for value types |
| Use `with` for non-destructive mutation | Creates copies instead of modifying state |
| Use records for domain events | Immutable, value-equality, self-documenting |
| Keep records focused and small | One responsibility per record type |
| Use `required` for mandatory properties | Compiler-enforced initialization |
| Use nullable parameters with defaults | `string? Note = null` for optional fields |

### Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Records with mutable collections | `List<T>` in record breaks value equality | Use `IReadOnlyList<T>` or `ImmutableArray<T>` |
| Record as EF entity | EF needs mutable properties and identity | Use `class` for EF entities, `record` for DTOs |
| `record struct` is mutable by default | Properties have `get; set;` | Use `readonly record struct` |
| Positional record with many params | Confusing parameter order | Use named properties or split into smaller records |
| Deep hierarchy with records | `with` creates copies at declared type | Limit inheritance depth to 1-2 levels |
| Equality with floating-point | IEEE 754 quirks with NaN | Override equality for float/double records |
| Reference equality assumptions | Records use value equality | Use `ReferenceEquals` if needed |
| Thread-safety with mutable records | Shared mutable state is unsafe | Use `init` properties for thread-safe immutability |
