# C# Pattern Matching

> Official Documentation: https://learn.microsoft.com/dotnet/csharp/fundamentals/functional/pattern-matching

## Table of Contents

1. [Type Patterns](#type-patterns)
2. [Declaration Patterns](#declaration-patterns)
3. [Constant Patterns](#constant-patterns)
4. [Property Patterns](#property-patterns)
5. [Positional Patterns](#positional-patterns)
6. [Relational Patterns](#relational-patterns)
7. [Logical Patterns](#logical-patterns)
8. [List Patterns (.NET 7+)](#list-patterns)
9. [Switch Expressions](#switch-expressions)
10. [Switch Statements with Patterns](#switch-statements-with-patterns)
11. [Var Pattern](#var-pattern)
12. [Discard Pattern](#discard-pattern)
13. [Nested Patterns](#nested-patterns)
14. [Pattern Matching with Tuples](#pattern-matching-with-tuples)
15. [Guard Clauses](#guard-clauses)
16. [Combining Patterns for Complex Matching](#combining-patterns-for-complex-matching)
17. [Best Practices and Common Pitfalls](#best-practices-and-common-pitfalls)

---

## Type Patterns

The `is` keyword checks if an expression is of a given type.

```csharp
object value = "Hello, World!";

// Type check only
if (value is string)
{
    Console.WriteLine("It's a string");
}

// Type check with variable declaration
if (value is string text)
{
    Console.WriteLine($"String with length {text.Length}");
}

// Negation
if (value is not string)
{
    Console.WriteLine("Not a string");
}

// In method parameters
public double CalculateArea(object shape) => shape switch
{
    Circle c => Math.PI * c.Radius * c.Radius,
    Rectangle r => r.Width * r.Height,
    Triangle t => 0.5 * t.Base * t.Height,
    _ => throw new ArgumentException($"Unknown shape: {shape.GetType().Name}")
};
```

### Type Pattern with Interfaces

```csharp
public interface IShape { double Area { get; } }
public record Circle(double Radius) : IShape
{
    public double Area => Math.PI * Radius * Radius;
}
public record Rectangle(double Width, double Height) : IShape
{
    public double Area => Width * Height;
}

public string Describe(IShape shape) => shape switch
{
    Circle { Radius: > 10 } => "Large circle",
    Circle => "Small circle",
    Rectangle { Width: var w, Height: var h } when w == h => "Square",
    Rectangle => "Rectangle",
    _ => "Unknown shape"
};
```

---

## Declaration Patterns

Declaration patterns combine a type check with a variable declaration.

```csharp
// In if statements
public void Process(object input)
{
    if (input is int number)
    {
        Console.WriteLine($"Integer: {number * 2}");
    }
    else if (input is string { Length: > 0 } text)
    {
        Console.WriteLine($"Non-empty string: {text}");
    }
    else if (input is IEnumerable<int> numbers)
    {
        Console.WriteLine($"Int collection with {numbers.Count()} items");
    }
}

// Scope of pattern variable
public bool TryParse(object input, out int result)
{
    if (input is int directInt)
    {
        result = directInt;
        return true;
    }

    if (input is string str && int.TryParse(str, out result))
    {
        return true;
    }

    result = 0;
    return false;
}
```

---

## Constant Patterns

Match against constant values including `null`.

```csharp
// Null check
if (value is null)
{
    Console.WriteLine("Value is null");
}

if (value is not null)
{
    Console.WriteLine("Value is not null");
}

// Constant matching
public string GetDayType(DayOfWeek day) => day switch
{
    DayOfWeek.Saturday => "Weekend",
    DayOfWeek.Sunday => "Weekend",
    _ => "Weekday"
};

// Numeric constants
public string Classify(int score) => score switch
{
    100 => "Perfect",
    0 => "Zero",
    _ => $"Score: {score}"
};

// String constants
public decimal GetDiscount(string code) => code switch
{
    "SAVE10" => 0.10m,
    "SAVE20" => 0.20m,
    "HALFOFF" => 0.50m,
    "" or null => 0m,
    _ => 0m
};
```

---

## Property Patterns

Match based on property values of an object.

```csharp
public record Address(string City, string State, string ZipCode, string Country);
public record Customer(string Name, Address Address, decimal Balance);

// Basic property pattern
public decimal GetTaxRate(Address address) => address switch
{
    { State: "WA" } => 0.065m,
    { State: "OR" } => 0.0m,
    { State: "CA" } => 0.0725m,
    { Country: "US" } => 0.05m,
    { Country: "CA" } => 0.13m,
    _ => 0.0m
};

// Nested property pattern
public string ClassifyCustomer(Customer customer) => customer switch
{
    { Balance: > 10000, Address.Country: "US" } => "VIP Domestic",
    { Balance: > 10000 } => "VIP International",
    { Balance: > 1000 } => "Premium",
    { Balance: > 0 } => "Standard",
    { Balance: <= 0 } => "Inactive",
    _ => "Unknown"
};

// Property pattern with variable capture
public string Describe(Customer customer) => customer switch
{
    { Name: var name, Balance: > 5000 } => $"{name} is a high-value customer",
    { Name: var name, Address.City: var city } => $"{name} from {city}",
    _ => "Unknown customer"
};

// Multiple property conditions
if (customer is { Name: not null, Balance: > 0, Address.State: "WA" or "OR" })
{
    Console.WriteLine("Active Pacific Northwest customer");
}
```

---

## Positional Patterns

Match based on deconstructed values. Works with records, tuples, and types that implement `Deconstruct`.

```csharp
public record Point(int X, int Y);

public string GetQuadrant(Point point) => point switch
{
    (0, 0) => "Origin",
    (> 0, > 0) => "Quadrant 1",
    (< 0, > 0) => "Quadrant 2",
    (< 0, < 0) => "Quadrant 3",
    (> 0, < 0) => "Quadrant 4",
    (0, _) => "Y-axis",
    (_, 0) => "X-axis",
};

// With Deconstruct method
public class Range
{
    public int Start { get; }
    public int End { get; }
    public Range(int start, int end) => (Start, End) = (start, end);
    public void Deconstruct(out int start, out int end) => (start, end) = (Start, End);
}

public string Describe(Range range) => range switch
{
    (0, 0) => "Empty range",
    (var s, var e) when s == e => $"Single point: {s}",
    (var s, var e) when s > e => "Invalid (reversed)",
    (var s, var e) => $"Range from {s} to {e} (size: {e - s})"
};
```

---

## Relational Patterns

Compare values using relational operators (C# 9+).

```csharp
// Basic relational
public string ClassifyTemperature(double tempC) => tempC switch
{
    < -40 => "Extreme cold",
    < 0 => "Freezing",
    < 10 => "Cold",
    < 20 => "Cool",
    < 30 => "Warm",
    < 40 => "Hot",
    _ => "Extreme heat"
};

// Combined with and/or
public string ClassifyAge(int age) => age switch
{
    < 0 => throw new ArgumentOutOfRangeException(nameof(age)),
    0 => "Newborn",
    >= 1 and <= 12 => "Child",
    >= 13 and <= 17 => "Teenager",
    >= 18 and <= 64 => "Adult",
    >= 65 => "Senior",
};

// HTTP status code classification
public string ClassifyStatusCode(int statusCode) => statusCode switch
{
    >= 200 and < 300 => "Success",
    >= 300 and < 400 => "Redirect",
    >= 400 and < 500 => "Client Error",
    >= 500 and < 600 => "Server Error",
    _ => "Unknown"
};

// In if statements
if (temperature is >= -10 and <= 40)
{
    Console.WriteLine("Normal operating temperature");
}
```

---

## Logical Patterns

Combine patterns with `and`, `or`, and `not` (C# 9+).

```csharp
// or pattern
public bool IsWeekend(DayOfWeek day) => day is DayOfWeek.Saturday or DayOfWeek.Sunday;

// and pattern
public bool IsWorkingHour(int hour) => hour is >= 9 and <= 17;

// not pattern
public bool IsNotNull(object? obj) => obj is not null;

// Complex combinations
public string ClassifyChar(char c) => c switch
{
    >= 'a' and <= 'z' => "Lowercase letter",
    >= 'A' and <= 'Z' => "Uppercase letter",
    >= '0' and <= '9' => "Digit",
    ' ' or '\t' or '\n' or '\r' => "Whitespace",
    '.' or ',' or ';' or ':' or '!' or '?' => "Punctuation",
    _ => "Other"
};

// not pattern in if
if (input is not (null or ""))
{
    Process(input);
}

// Negated type pattern
if (shape is not Circle)
{
    Console.WriteLine("Not a circle");
}

// Complex logical with property patterns
public decimal CalculateShipping(Order order) => order switch
{
    { Weight: <= 1, Destination.Country: "US" } => 5.99m,
    { Weight: <= 5 and > 1, Destination.Country: "US" } => 9.99m,
    { Weight: > 5, Destination.Country: "US" } => 14.99m,
    { Destination.Country: not "US" } => 29.99m,
    _ => 19.99m
};
```

---

## List Patterns

Match against array and list elements (.NET 7+ / C# 11).

```csharp
// Exact match
int[] numbers = [1, 2, 3];

if (numbers is [1, 2, 3])
    Console.WriteLine("Exact match: [1, 2, 3]");

// Discard individual elements
if (numbers is [1, _, 3])
    Console.WriteLine("Starts with 1, ends with 3");

// Slice pattern (..)
if (numbers is [1, ..])
    Console.WriteLine("Starts with 1");

if (numbers is [.., 3])
    Console.WriteLine("Ends with 3");

if (numbers is [1, .. var middle, 3])
    Console.WriteLine($"Middle elements: [{string.Join(", ", middle)}]");

// Capture elements
if (numbers is [var first, .. , var last])
    Console.WriteLine($"First: {first}, Last: {last}");

// In switch expressions
string Describe(int[] arr) => arr switch
{
    [] => "Empty",
    [var single] => $"Single element: {single}",
    [var first, var second] => $"Pair: ({first}, {second})",
    [var first, .., var last] => $"Many elements, first={first}, last={last}",
};

// With relational patterns
string DescribeScores(int[] scores) => scores switch
{
    [> 90, > 90, > 90] => "All excellent",
    [>= 60, >= 60, >= 60] => "All passing",
    [< 60, ..] => "First score failing",
    _ => "Mixed results"
};

// Nested list patterns
string AnalyzeMatrix(int[][] matrix) => matrix switch
{
    [[1, 0], [0, 1]] => "Identity matrix",
    [[0, ..], ..] => "First row starts with zero",
    [var firstRow, ..] when firstRow.All(x => x == 0) => "First row is all zeros",
    _ => "Other matrix"
};
```

### List Patterns with Span

```csharp
// Works with Span<T> and ReadOnlySpan<T>
ReadOnlySpan<char> span = "Hello".AsSpan();

if (span is ['H', ..])
    Console.WriteLine("Starts with H");

// Parse command
ReadOnlySpan<char> command = "GET /api/products".AsSpan();
// (Note: list patterns on spans work element-by-element)
```

---

## Switch Expressions

Compact pattern matching that returns a value (C# 8+).

```csharp
// Basic switch expression
public string GetDescription(Shape shape) => shape switch
{
    Circle c => $"Circle with radius {c.Radius}",
    Rectangle r => $"Rectangle {r.Width}x{r.Height}",
    Triangle t => $"Triangle with base {t.Base}",
    _ => "Unknown shape"
};

// Multi-pattern with when
public decimal CalculateDiscount(Customer customer, Order order) => (customer, order) switch
{
    ({ IsPremium: true }, { Total: > 1000 }) => 0.20m,
    ({ IsPremium: true }, _) => 0.10m,
    (_, { Total: > 500 }) => 0.05m,
    (_, { Total: > 100 }) => 0.02m,
    _ => 0m
};

// Chained switch expression
public string FormatValue(object value) => value switch
{
    null => "null",
    int i => i.ToString("N0"),
    double d => d.ToString("F2"),
    decimal m => m.ToString("C"),
    DateTime dt => dt.ToString("yyyy-MM-dd"),
    string s when s.Length > 50 => $"{s[..50]}...",
    string s => s,
    IEnumerable<object> list => $"[{string.Join(", ", list)}]",
    _ => value.ToString() ?? "unknown"
};
```

### Expression-Bodied Members with Switch

```csharp
public record Order(decimal Subtotal, string ShipMethod, bool IsRush);

public static class ShippingCalculator
{
    public static decimal CalculateShipping(Order order) => order.ShipMethod switch
    {
        "standard" => order.Subtotal > 50 ? 0m : 5.99m,
        "express" => 12.99m,
        "overnight" => 24.99m,
        _ => throw new ArgumentException($"Unknown shipping method: {order.ShipMethod}")
    };

    public static decimal CalculateTotal(Order order) =>
        order.Subtotal + CalculateShipping(order) + (order.IsRush ? 10m : 0m);
}
```

---

## Switch Statements with Patterns

Traditional switch statements enhanced with pattern matching.

```csharp
public void ProcessCommand(object command)
{
    switch (command)
    {
        case string s when s.StartsWith('/'):
            HandleSlashCommand(s);
            break;

        case string { Length: 0 }:
            Console.WriteLine("Empty string command");
            break;

        case string s:
            HandleTextCommand(s);
            break;

        case int code when code >= 100 and code < 200:
            HandleInformational(code);
            break;

        case int code when code >= 200 and code < 300:
            HandleSuccess(code);
            break;

        case null:
            Console.WriteLine("Null command received");
            break;

        default:
            Console.WriteLine($"Unknown command type: {command.GetType()}");
            break;
    }
}
```

---

## Var Pattern

The `var` pattern always matches and captures the value into a variable.

```csharp
// Capture intermediate values in complex patterns
public string AnalyzePoint(Point p) => p switch
{
    var (x, y) when x == y => $"On diagonal at ({x}, {y})",
    var (x, y) when Math.Abs(x) == Math.Abs(y) => $"On anti-diagonal at ({x}, {y})",
    var (x, y) => $"Point at ({x}, {y})"
};

// Capture for logging/debugging
if (GetResult() is var result && result > 0)
{
    Console.WriteLine($"Positive result: {result}");
}

// In property patterns
public string Describe(Customer c) => c switch
{
    { Balance: var b } when b > 10000 => $"VIP with balance {b:C}",
    { Name: var name, Address: { City: var city } } => $"{name} from {city}",
    _ => "Unknown"
};
```

---

## Discard Pattern

The `_` pattern matches anything and is used when the value is not needed.

```csharp
// Catch-all in switch
public string Classify(object obj) => obj switch
{
    int => "integer",
    string => "string",
    _ => "other"       // Discard: matches anything else
};

// In positional patterns
if (point is (_, 0))
    Console.WriteLine("On the X-axis (any X value)");

// In list patterns
if (array is [_, _, var third, ..])
    Console.WriteLine($"Third element: {third}");

// Multiple discards
var (_, _, zip) = address;  // Only need zip code from deconstruction
```

---

## Nested Patterns

Combine multiple pattern types for deep matching.

```csharp
public record Address(string City, string State, string Country);
public record Customer(string Name, int Age, Address Address, decimal Balance);

public string Classify(Customer customer) => customer switch
{
    // Nested: type + property + relational + logical
    {
        Age: >= 18 and <= 65,
        Balance: > 0,
        Address: { Country: "US", State: "WA" or "OR" or "CA" }
    } => "Active West Coast US customer",

    {
        Age: < 18,
        Address.Country: "US"
    } => "Minor (US)",

    {
        Balance: <= 0,
        Name: not (null or "")
    } => $"Inactive customer: {customer.Name}",

    _ => "Other"
};

// Deep nesting with records
public record OrderLine(Product Product, int Quantity);
public record Product(string Name, decimal Price, string Category);

public decimal CalculateLineDiscount(OrderLine line) => line switch
{
    { Product: { Category: "Electronics", Price: > 500 }, Quantity: >= 2 } => 0.15m,
    { Product: { Category: "Electronics" }, Quantity: >= 5 } => 0.10m,
    { Product.Price: > 100, Quantity: >= 10 } => 0.08m, // Shorthand nested access
    { Quantity: >= 20 } => 0.05m,
    _ => 0m
};
```

---

## Pattern Matching with Tuples

```csharp
// State machine
public string GetNextState(string currentState, string action) =>
    (currentState, action) switch
    {
        ("idle", "start") => "running",
        ("running", "pause") => "paused",
        ("running", "stop") => "stopped",
        ("paused", "resume") => "running",
        ("paused", "stop") => "stopped",
        ("stopped", "reset") => "idle",
        (_, "emergency_stop") => "stopped",
        _ => throw new InvalidOperationException(
            $"Invalid transition: {currentState} + {action}")
    };

// Rock-Paper-Scissors
public string DetermineWinner(string player1, string player2) =>
    (player1, player2) switch
    {
        ("rock", "scissors") or ("scissors", "paper") or ("paper", "rock") => "Player 1 wins",
        ("scissors", "rock") or ("paper", "scissors") or ("rock", "paper") => "Player 2 wins",
        _ when player1 == player2 => "Tie",
        _ => throw new ArgumentException("Invalid move")
    };

// Multi-value classification
public decimal GetRate(string customerType, string product, int quantity) =>
    (customerType, product, quantity) switch
    {
        ("wholesale", _, >= 1000) => 0.30m,
        ("wholesale", _, >= 100) => 0.20m,
        ("wholesale", _, _) => 0.10m,
        ("retail", "premium", >= 10) => 0.05m,
        ("retail", _, _) => 0m,
        _ => 0m
    };
```

---

## Guard Clauses

The `when` keyword adds additional conditions to patterns.

```csharp
// when clause in switch expression
public string Classify(int[] data) => data switch
{
    [] => "Empty",
    [var single] when single > 0 => $"Single positive: {single}",
    [var single] => $"Single non-positive: {single}",
    { Length: var len } when len > 100 => "Large dataset",
    [var first, .., var last] when first == last => "Starts and ends with same value",
    _ => $"Dataset with {data.Length} elements"
};

// Guard with external method call
public decimal CalculatePrice(Order order) => order switch
{
    { CustomerId: var id } when IsVipCustomer(id) => order.Subtotal * 0.8m,
    { Subtotal: > 1000 } when IsHolidaySeason() => order.Subtotal * 0.85m,
    { Subtotal: > 500 } => order.Subtotal * 0.95m,
    _ => order.Subtotal
};

// when with pattern variables
public string DescribeNumbers(int a, int b) => (a, b) switch
{
    var (x, y) when x == y => $"Equal: {x}",
    var (x, y) when x + y == 0 => $"Opposites: {x} and {y}",
    var (x, y) when x > y => $"{x} is greater",
    var (x, y) => $"{y} is greater or equal"
};
```

---

## Combining Patterns for Complex Matching

### Validation Example

```csharp
public record RegistrationRequest(
    string? Email,
    string? Password,
    int Age,
    string? Country);

public static class Validator
{
    public static string? Validate(RegistrationRequest request) => request switch
    {
        { Email: null or "" } => "Email is required",
        { Email: var e } when !e.Contains('@') => "Invalid email format",
        { Password: null or { Length: < 8 } } => "Password must be at least 8 characters",
        { Age: < 13 } => "Must be at least 13 years old",
        { Age: > 120 } => "Invalid age",
        { Country: null or "" } => "Country is required",
        _ => null // Valid
    };
}

// Usage
var request = new RegistrationRequest("user@example.com", "password123", 25, "US");
var error = Validator.Validate(request);
if (error is not null)
{
    Console.WriteLine($"Validation error: {error}");
}
```

### HTTP Response Handling

```csharp
public record ApiResponse(int StatusCode, string? Body, string? ErrorMessage);

public async Task<T?> HandleResponse<T>(ApiResponse response) where T : class => response switch
{
    { StatusCode: 200, Body: not null } =>
        JsonSerializer.Deserialize<T>(response.Body),

    { StatusCode: 204 } => null, // No content

    { StatusCode: >= 400 and < 500, ErrorMessage: var msg } =>
        throw new HttpRequestException($"Client error: {msg ?? "Unknown"}"),

    { StatusCode: >= 500, ErrorMessage: var msg } =>
        throw new HttpRequestException($"Server error: {msg ?? "Unknown"}"),

    { StatusCode: 301 or 302 } =>
        throw new HttpRequestException("Redirect not followed"),

    _ => throw new HttpRequestException($"Unexpected status: {response.StatusCode}")
};
```

---

## Best Practices and Common Pitfalls

### Best Practices

| Practice | Description |
|----------|-------------|
| Use switch expressions for type dispatch | Cleaner than if/else chains |
| Use `is not null` instead of `!= null` | Pattern matching is more idiomatic |
| Prefer property patterns for validation | Readable, declarative style |
| Use list patterns for fixed-format data | Command parsing, protocol handling |
| Combine patterns for complex logic | `and`, `or`, `not` compose well |
| Always include a discard `_` arm | Ensures exhaustive matching |
| Use `when` for conditions that cannot be expressed as patterns | External method calls, computed conditions |
| Keep pattern arms ordered specific-to-general | First match wins; avoid unreachable patterns |

### Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Missing discard arm `_` | Runtime exception if no match | Always include catch-all |
| Overlapping patterns | Earlier arm matches, later arm unreachable | Order from most specific to most general |
| Complex nested patterns | Hard to read and maintain | Extract to helper methods |
| Forgetting `not` precedence | `not A or B` means `(not A) or B` | Use parentheses: `not (A or B)` |
| Mutable state in guard clauses | `when` evaluated at match time only | Keep guards side-effect-free |
| Pattern matching vs polymorphism | Adding new types requires modifying switch | Use pattern matching for closed type sets, polymorphism for open ones |
| `is var x` always matches | Can be confusing | Use `is` with type or condition for clarity |
| List patterns on non-indexable types | Only works with arrays, lists, spans | Convert to array first if needed |
