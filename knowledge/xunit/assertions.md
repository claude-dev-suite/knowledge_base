# xUnit Assertions

> Official Documentation: https://xunit.net/docs/assertions

Comprehensive reference for xUnit.net assertion methods -- the built-in verification API for validating test outcomes. This guide covers all assertion categories including equality, boolean, null, type, exception, string, collection, range, and custom assertions targeting .NET 8+ with xUnit v2/v3.

---

## Table of Contents

1. [Equality Assertions](#equality-assertions)
2. [Boolean Assertions](#boolean-assertions)
3. [Null Assertions](#null-assertions)
4. [String Assertions](#string-assertions)
5. [Collection Assertions](#collection-assertions)
6. [Type Assertions](#type-assertions)
7. [Exception Assertions](#exception-assertions)
8. [Range Assertions](#range-assertions)
9. [Structural Comparison](#structural-comparison)
10. [Grouped Assertions](#grouped-assertions)
11. [Custom Assertions](#custom-assertions)
12. [FluentAssertions Integration](#fluentassertions-integration)
13. [Best Practices](#best-practices)
14. [Common Pitfalls](#common-pitfalls)

---

## Equality Assertions

### Assert.Equal and Assert.NotEqual

```csharp
// Value types
Assert.Equal(42, calculator.Add(40, 2));
Assert.NotEqual(0, result);

// Strings (ordinal comparison)
Assert.Equal("Hello, World!", greeting);
Assert.Equal("hello, world!", greeting, ignoreCase: true);

// String with custom comparer
Assert.Equal("Strasse", "Straße",
    StringComparer.Create(CultureInfo.GetCultureInfo("de-DE"), CompareOptions.None));

// Floating point with precision
Assert.Equal(3.14159, Math.PI, precision: 5);    // 5 decimal places
Assert.Equal(3.14, Math.PI, tolerance: 0.01);    // Within tolerance

// Decimal
Assert.Equal(10.50m, invoice.Total);

// DateTime
Assert.Equal(
    new DateTime(2024, 6, 15, 10, 30, 0),
    result.CreatedAt);

// Collections (element-by-element comparison)
Assert.Equal(
    new[] { 1, 2, 3 },
    actualList);

Assert.Equal(
    new[] { "apple", "banana", "cherry" },
    actualFruits);

// With custom comparer
Assert.Equal(expected, actual,
    new UserEqualityComparer());
```

### Custom Equality Comparer

```csharp
public class UserEqualityComparer : IEqualityComparer<User>
{
    public bool Equals(User? x, User? y)
    {
        if (ReferenceEquals(x, y)) return true;
        if (x is null || y is null) return false;
        return x.Id == y.Id && x.Name == y.Name && x.Email == y.Email;
    }

    public int GetHashCode(User obj) =>
        HashCode.Combine(obj.Id, obj.Name, obj.Email);
}

[Fact]
public void GetUser_ReturnsExpectedUser()
{
    var expected = new User { Id = 1, Name = "Alice", Email = "alice@test.com" };

    var actual = service.GetById(1);

    Assert.Equal(expected, actual, new UserEqualityComparer());
}
```

---

## Boolean Assertions

### Assert.True and Assert.False

```csharp
[Fact]
public void IsAdult_Age18_ReturnsTrue()
{
    var person = new Person { Age = 18 };

    Assert.True(person.IsAdult());
    Assert.True(person.IsAdult(), "Person aged 18 should be considered an adult");
}

[Fact]
public void IsExpired_FutureDate_ReturnsFalse()
{
    var coupon = new Coupon { ExpiresAt = DateTime.UtcNow.AddDays(30) };

    Assert.False(coupon.IsExpired());
    Assert.False(coupon.IsExpired(), "Coupon with future expiry should not be expired");
}
```

---

## Null Assertions

### Assert.Null and Assert.NotNull

```csharp
[Fact]
public void FindUser_NonExistentId_ReturnsNull()
{
    var user = repository.FindById(999);

    Assert.Null(user);
}

[Fact]
public void CreateUser_ReturnsNonNullUser()
{
    var user = service.Create("Alice", "alice@test.com");

    Assert.NotNull(user);
    Assert.NotNull(user.Id);         // ID should be assigned
    Assert.NotNull(user.CreatedAt);  // Timestamp should be set
}
```

---

## String Assertions

### Contains, StartsWith, EndsWith

```csharp
[Fact]
public void ErrorMessage_ContainsFieldName()
{
    var result = validator.Validate(new User { Name = "" });

    Assert.Contains("Name", result.ErrorMessage);
    Assert.Contains("required", result.ErrorMessage, StringComparison.OrdinalIgnoreCase);
}

[Fact]
public void GenerateSlug_StartsWithPrefix()
{
    var slug = SlugGenerator.Generate("Hello World");

    Assert.StartsWith("hello", slug);
    Assert.EndsWith("world", slug);
}

[Fact]
public void DoesNotContain_SensitiveData()
{
    var log = logger.GetLastEntry();

    Assert.DoesNotContain("password", log, StringComparison.OrdinalIgnoreCase);
    Assert.DoesNotContain("secret", log, StringComparison.OrdinalIgnoreCase);
}
```

### Assert.Matches (Regex)

```csharp
[Fact]
public void GenerateId_MatchesUuidFormat()
{
    var id = IdGenerator.NewId();

    Assert.Matches(
        @"^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$",
        id);
}

[Fact]
public void FormatPhoneNumber_MatchesPattern()
{
    var formatted = PhoneFormatter.Format("5551234567");

    Assert.Matches(@"^\(\d{3}\) \d{3}-\d{4}$", formatted);
}
```

### Assert.Empty for Strings

```csharp
[Fact]
public void Trim_WhitespaceOnly_ReturnsEmpty()
{
    var result = "   ".Trim();

    Assert.Empty(result);
    Assert.NotEmpty("hello");
}
```

---

## Collection Assertions

### Assert.Empty and Assert.NotEmpty

```csharp
[Fact]
public void GetProducts_NoMatch_ReturnsEmptyList()
{
    var products = repository.Search("nonexistent");

    Assert.Empty(products);
}

[Fact]
public void GetAllUsers_ReturnsNonEmptyList()
{
    SeedDatabase();

    var users = repository.GetAll();

    Assert.NotEmpty(users);
}
```

### Assert.Single

```csharp
[Fact]
public void FindByEmail_UniqueEmail_ReturnsSingleUser()
{
    var users = repository.FindByEmail("alice@test.com");

    var user = Assert.Single(users);
    Assert.Equal("Alice", user.Name);
}

[Fact]
public void GetErrors_SingleValidationError_ReturnsSingle()
{
    var result = validator.Validate(new Order { Total = -1 });

    var error = Assert.Single(result.Errors);
    Assert.Equal("Total", error.PropertyName);
}
```

### Assert.Contains and Assert.DoesNotContain (Collections)

```csharp
[Fact]
public void GetRoles_AdminUser_ContainsAdminRole()
{
    var roles = userService.GetRoles("admin-user-id");

    Assert.Contains("Admin", roles);
    Assert.Contains(roles, r => r.StartsWith("Admin"));
    Assert.DoesNotContain("SuperUser", roles);
}

[Fact]
public void GetProducts_ContainsExpectedProduct()
{
    var products = catalog.GetAll();

    Assert.Contains(products, p => p.Name == "Widget" && p.Price == 9.99m);
    Assert.DoesNotContain(products, p => p.IsDiscontinued);
}
```

### Assert.All

Verify a condition holds for every element in a collection:

```csharp
[Fact]
public void GetActiveUsers_AllAreActive()
{
    var users = repository.GetActiveUsers();

    Assert.All(users, user =>
    {
        Assert.True(user.IsActive);
        Assert.NotNull(user.Email);
        Assert.NotEmpty(user.Name);
    });
}

[Fact]
public void GetPrices_AllArePositive()
{
    var prices = catalog.GetAllPrices();

    Assert.All(prices, price => Assert.True(price > 0, $"Price {price} should be positive"));
}
```

### Assert.Collection

Verify each element in a collection matches a specific inspector, in order:

```csharp
[Fact]
public void GetTopThreeProducts_ReturnsInOrder()
{
    var products = catalog.GetTopThree();

    Assert.Collection(products,
        first =>
        {
            Assert.Equal("Product A", first.Name);
            Assert.Equal(4.9, first.Rating, precision: 1);
        },
        second =>
        {
            Assert.Equal("Product B", second.Name);
            Assert.Equal(4.7, second.Rating, precision: 1);
        },
        third =>
        {
            Assert.Equal("Product C", third.Name);
            Assert.Equal(4.5, third.Rating, precision: 1);
        }
    );
}
```

### Assert.Distinct

```csharp
[Fact]
public void GenerateIds_AreAllUnique()
{
    var ids = Enumerable.Range(0, 100)
        .Select(_ => IdGenerator.NewId())
        .ToList();

    Assert.Distinct(ids);
}
```

### Assert.Subset and Assert.Superset

```csharp
[Fact]
public void UserPermissions_SubsetOfAllPermissions()
{
    var allPermissions = new HashSet<string> { "read", "write", "delete", "admin" };
    var userPermissions = new HashSet<string> { "read", "write" };

    Assert.Subset(allPermissions, userPermissions);      // user is subset of all
    Assert.Superset(userPermissions, allPermissions);    // all is superset of user
    Assert.ProperSubset(allPermissions, userPermissions); // proper subset (not equal)
}
```

---

## Type Assertions

### Assert.IsType and Assert.IsNotType

```csharp
[Fact]
public void CreateShape_Circle_ReturnsCircleType()
{
    var shape = ShapeFactory.Create("circle", radius: 5);

    var circle = Assert.IsType<Circle>(shape);  // Returns the typed object
    Assert.Equal(5, circle.Radius);
}

[Fact]
public void CreateShape_Circle_IsNotRectangle()
{
    var shape = ShapeFactory.Create("circle", radius: 5);

    Assert.IsNotType<Rectangle>(shape);
}
```

### Assert.IsAssignableFrom

Checks that the object is of the specified type or a derived type:

```csharp
[Fact]
public void CreateAnimal_Dog_IsAssignableToAnimal()
{
    var animal = AnimalFactory.Create("dog");

    Assert.IsAssignableFrom<Animal>(animal);
    Assert.IsAssignableFrom<ICanBark>(animal);
}

[Fact]
public void GetResult_ReturnsIActionResult()
{
    var controller = new ProductsController(mockService);

    var result = controller.GetAll();

    Assert.IsAssignableFrom<IActionResult>(result);
}
```

---

## Exception Assertions

### Assert.Throws<T>

```csharp
[Fact]
public void Withdraw_InsufficientFunds_ThrowsInsufficientFundsException()
{
    var account = new BankAccount(balance: 100m);

    var exception = Assert.Throws<InsufficientFundsException>(
        () => account.Withdraw(200m));

    Assert.Equal(100m, exception.CurrentBalance);
    Assert.Equal(200m, exception.RequestedAmount);
    Assert.Contains("insufficient", exception.Message, StringComparison.OrdinalIgnoreCase);
}

[Fact]
public void Constructor_NullName_ThrowsArgumentNullException()
{
    var exception = Assert.Throws<ArgumentNullException>(
        () => new User(name: null!, email: "test@test.com"));

    Assert.Equal("name", exception.ParamName);
}

[Fact]
public void SetAge_NegativeValue_ThrowsArgumentOutOfRangeException()
{
    var person = new Person();

    var exception = Assert.Throws<ArgumentOutOfRangeException>(
        () => person.Age = -1);

    Assert.Equal("value", exception.ParamName);
}
```

### Assert.ThrowsAsync<T>

```csharp
[Fact]
public async Task GetUser_NotFound_ThrowsNotFoundException()
{
    var service = new UserService(mockRepo);

    var exception = await Assert.ThrowsAsync<NotFoundException>(
        () => service.GetByIdAsync(999));

    Assert.Equal("User", exception.EntityName);
    Assert.Equal(999, exception.EntityId);
}

[Fact]
public async Task CreateOrder_DuplicateId_ThrowsConflictException()
{
    var service = CreateOrderService();

    await Assert.ThrowsAsync<ConflictException>(
        async () => await service.CreateAsync(new Order { Id = "existing-id" }));
}
```

### Assert.ThrowsAny<T> (Includes Derived Types)

```csharp
[Fact]
public void FailingOperation_ThrowsHttpException_OrDerived()
{
    // Catches HttpRequestException or any derived exception
    Assert.ThrowsAny<HttpRequestException>(
        () => client.SendRequest());
}

[Fact]
public async Task AsyncOperation_ThrowsAnyException()
{
    await Assert.ThrowsAnyAsync<Exception>(
        () => service.ProcessAsync());
}
```

### Verifying No Exception Is Thrown

```csharp
[Fact]
public void ValidOperation_DoesNotThrow()
{
    var account = new BankAccount(balance: 100m);

    // Implicitly verifies no exception -- if it throws, the test fails
    var exception = Record.Exception(() => account.Withdraw(50m));

    Assert.Null(exception);
}

[Fact]
public async Task ValidAsyncOperation_DoesNotThrow()
{
    var exception = await Record.ExceptionAsync(
        () => service.ProcessAsync(validInput));

    Assert.Null(exception);
}
```

---

## Range Assertions

### Assert.InRange and Assert.NotInRange

```csharp
[Fact]
public void GenerateAge_WithinExpectedRange()
{
    var age = personGenerator.GenerateAge();

    Assert.InRange(age, 0, 150);
}

[Fact]
public void CalculateDiscount_NotExceedingMaximum()
{
    var discount = calculator.CalculateDiscount(1000m, CustomerTier.Gold);

    Assert.InRange(discount, 0m, 200m);
    Assert.NotInRange(discount, 201m, decimal.MaxValue);
}

[Fact]
public void GenerateTimestamp_WithinLastMinute()
{
    var before = DateTime.UtcNow;
    var timestamp = TimestampGenerator.Now();
    var after = DateTime.UtcNow;

    Assert.InRange(timestamp, before, after);
}

[Fact]
public void ProcessingTime_UnderThreshold()
{
    var stopwatch = Stopwatch.StartNew();
    service.Process(testData);
    stopwatch.Stop();

    Assert.InRange(stopwatch.ElapsedMilliseconds, 0, 5000);
}
```

---

## Structural Comparison

### Assert.Equivalent

`Assert.Equivalent` performs a deep structural comparison, ignoring reference equality. It does not require the types to match exactly and works across anonymous types, DTOs, and collections:

```csharp
[Fact]
public void MapToDto_ProducesEquivalentObject()
{
    var entity = new UserEntity
    {
        Id = 1,
        Name = "Alice",
        Email = "alice@test.com",
        Roles = ["Admin", "User"],
    };

    var dto = mapper.MapToDto(entity);

    // Structurally compares property values, not reference equality
    Assert.Equivalent(
        new { Id = 1, Name = "Alice", Email = "alice@test.com" },
        dto);
}

[Fact]
public void GetProducts_EquivalentToExpected()
{
    var expected = new[]
    {
        new { Name = "Widget", Price = 9.99m },
        new { Name = "Gadget", Price = 19.99m },
    };

    var actual = service.GetAll()
        .Select(p => new { p.Name, p.Price })
        .ToArray();

    Assert.Equivalent(expected, actual);
}

[Fact]
public void CollectionEquivalence_IgnoresOrder()
{
    var expected = new[] { 3, 1, 2 };
    var actual = new[] { 1, 2, 3 };

    // By default, strict: false means order does not matter
    Assert.Equivalent(expected, actual, strict: false);
}

[Fact]
public void StrictEquivalence_EnforcesExactProperties()
{
    var expected = new { Name = "Alice", Age = 30 };
    var actual = new { Name = "Alice", Age = 30, Email = "alice@test.com" };

    // strict: true fails because actual has extra property
    // Assert.Equivalent(expected, actual, strict: true); // Would fail!

    // strict: false (default) allows extra properties on actual
    Assert.Equivalent(expected, actual, strict: false);
}
```

---

## Grouped Assertions

### Assert.Multiple

Run multiple assertions and report all failures at once rather than stopping at the first failure:

```csharp
[Fact]
public void CreateUser_SetsAllProperties()
{
    var user = userService.Create("Alice", "alice@test.com", UserRole.Admin);

    Assert.Multiple(
        () => Assert.Equal("Alice", user.Name),
        () => Assert.Equal("alice@test.com", user.Email),
        () => Assert.Equal(UserRole.Admin, user.Role),
        () => Assert.True(user.IsActive),
        () => Assert.NotNull(user.CreatedAt),
        () => Assert.Null(user.DeletedAt)
    );
}

[Fact]
public void OrderSummary_AllFieldsCorrect()
{
    var order = orderService.GetSummary("ORDER-001");

    Assert.Multiple(
        () => Assert.Equal("ORDER-001", order.Id),
        () => Assert.Equal(3, order.ItemCount),
        () => Assert.Equal(149.97m, order.Subtotal),
        () => Assert.Equal(12.00m, order.Tax),
        () => Assert.Equal(161.97m, order.Total),
        () => Assert.Equal(OrderStatus.Completed, order.Status)
    );
}
```

---

## Custom Assertions

### Extension Method Pattern

```csharp
public static class CustomAssert
{
    public static void IsValidEmail(string email)
    {
        Assert.NotNull(email);
        Assert.NotEmpty(email);
        Assert.Matches(@"^[^@\s]+@[^@\s]+\.[^@\s]+$", email);
    }

    public static void IsSuccessStatusCode(HttpResponseMessage response)
    {
        Assert.True(
            response.IsSuccessStatusCode,
            $"Expected success status code, but got {(int)response.StatusCode} " +
            $"({response.StatusCode}). Response body: " +
            $"{response.Content.ReadAsStringAsync().GetAwaiter().GetResult()}");
    }

    public static void HasValidationError(
        ValidationResult result, string propertyName, string expectedMessage)
    {
        Assert.False(result.IsValid, "Expected validation to fail but it succeeded");
        var error = Assert.Single(result.Errors, e => e.PropertyName == propertyName);
        Assert.Contains(expectedMessage, error.ErrorMessage);
    }

    public static void IsWithinTimeRange(
        DateTime actual, DateTime expected, TimeSpan tolerance)
    {
        var diff = Math.Abs((actual - expected).TotalMilliseconds);
        Assert.True(
            diff <= tolerance.TotalMilliseconds,
            $"Expected {actual:O} to be within {tolerance} of {expected:O}, " +
            $"but the difference was {TimeSpan.FromMilliseconds(diff)}");
    }
}

// Usage:
[Fact]
public void CreateUser_HasValidEmail()
{
    var user = service.Create("Alice", "alice@test.com");

    CustomAssert.IsValidEmail(user.Email);
}

[Fact]
public async Task GetProducts_ReturnsSuccess()
{
    var response = await client.GetAsync("/api/products");

    CustomAssert.IsSuccessStatusCode(response);
}
```

---

## FluentAssertions Integration

FluentAssertions provides a more readable assertion syntax and better error messages:

### Installation

```bash
dotnet add package FluentAssertions
```

### Basic Usage

```csharp
using FluentAssertions;

[Fact]
public void Calculate_ReturnsExpectedResult()
{
    var result = calculator.Add(2, 3);

    result.Should().Be(5);
    result.Should().BePositive();
    result.Should().BeGreaterThan(0);
    result.Should().BeInRange(1, 10);
}

[Fact]
public void GetUser_ReturnsExpectedUser()
{
    var user = service.GetById(1);

    user.Should().NotBeNull();
    user!.Name.Should().Be("Alice");
    user.Email.Should().Contain("@").And.EndWith(".com");
    user.Age.Should().BeGreaterThanOrEqualTo(18);
    user.Roles.Should().Contain("Admin").And.HaveCount(2);
}
```

### Object Comparison

```csharp
[Fact]
public void MapToDto_ProducesEquivalentObject()
{
    var expected = new UserDto { Id = 1, Name = "Alice", Email = "alice@test.com" };

    var actual = mapper.MapToDto(entity);

    actual.Should().BeEquivalentTo(expected);

    // Excluding specific properties
    actual.Should().BeEquivalentTo(expected, options => options
        .Excluding(u => u.CreatedAt)
        .Excluding(u => u.UpdatedAt));

    // With custom comparison
    actual.Should().BeEquivalentTo(expected, options => options
        .Using<DateTime>(ctx => ctx.Subject.Should().BeCloseTo(ctx.Expectation, TimeSpan.FromSeconds(1)))
        .WhenTypeIs<DateTime>());
}
```

### Collection Assertions

```csharp
[Fact]
public void GetProducts_ReturnsExpectedProducts()
{
    var products = service.GetAll();

    products.Should().HaveCount(3);
    products.Should().ContainSingle(p => p.Name == "Widget");
    products.Should().OnlyContain(p => p.Price > 0);
    products.Should().BeInAscendingOrder(p => p.Name);
    products.Should().NotContainNulls();
    products.Should().AllSatisfy(p =>
    {
        p.Id.Should().NotBeEmpty();
        p.Name.Should().NotBeNullOrWhiteSpace();
        p.Price.Should().BePositive();
    });
}
```

### Exception Assertions

```csharp
[Fact]
public void InvalidOperation_ThrowsWithMessage()
{
    var account = new BankAccount(100m);

    var act = () => account.Withdraw(200m);

    act.Should().Throw<InsufficientFundsException>()
        .WithMessage("*insufficient*")
        .Where(ex => ex.CurrentBalance == 100m);
}

[Fact]
public async Task AsyncOperation_ThrowsExpectedException()
{
    var act = () => service.ProcessAsync(invalidInput);

    await act.Should().ThrowAsync<ValidationException>()
        .WithMessage("*validation*failed*");
}
```

---

## Best Practices

### Assertion Guidelines

| Guideline | Description |
|-----------|-------------|
| One logical assertion per test | Multiple `Assert` calls are fine if they verify one logical outcome |
| Use the most specific assertion | `Assert.Equal` over `Assert.True(a == b)` |
| Include failure messages for `Assert.True/False` | Helps diagnose failures |
| Prefer `Assert.Single` over `Assert.Equal(1, count)` | Returns the single element for further assertions |
| Use `Assert.Collection` for ordered verification | Verifies both count and individual elements |
| Use `Assert.Equivalent` for DTOs | Deep structural comparison without custom comparers |

### Assertion Selection Guide

| Scenario | Recommended Assertion |
|----------|----------------------|
| Exact value match | `Assert.Equal(expected, actual)` |
| Object structural match | `Assert.Equivalent(expected, actual)` |
| Collection has exact items in order | `Assert.Collection(list, ...)` |
| All items satisfy condition | `Assert.All(list, ...)` |
| Collection contains specific item | `Assert.Contains(list, predicate)` |
| Exactly one item | `Assert.Single(list)` |
| Exception thrown | `Assert.Throws<T>(action)` |
| No exception thrown | `Assert.Null(Record.Exception(...))` |
| Type checking | `Assert.IsType<T>(obj)` |
| Range checking | `Assert.InRange(val, low, high)` |
| Multiple independent checks | `Assert.Multiple(...)` |

---

## Common Pitfalls

### 1. Comparing Floating Point with Assert.Equal

```csharp
// WRONG: May fail due to floating-point precision
Assert.Equal(0.1 + 0.2, 0.3); // Fails! 0.30000000000000004 != 0.3

// CORRECT: Use precision parameter
Assert.Equal(0.3, 0.1 + 0.2, precision: 10);

// CORRECT: Use tolerance
Assert.Equal(0.3, 0.1 + 0.2, tolerance: 1e-15);
```

### 2. Wrong Argument Order in Assert.Equal

```csharp
// WRONG: Arguments reversed -- error message will be confusing
Assert.Equal(actual, expected);
// "Expected: [actual value], Actual: [expected value]" -- backwards!

// CORRECT: Expected first, then actual
Assert.Equal(expected, actual);
```

### 3. Using Assert.True for Equality

```csharp
// WRONG: Poor error message on failure
Assert.True(user.Name == "Alice");
// Message: "Assert.True() Failure"

// CORRECT: Clear error message
Assert.Equal("Alice", user.Name);
// Message: "Assert.Equal() Failure: Expected: Alice, Actual: Bob"
```

### 4. Not Capturing Return from Assert.Throws

```csharp
// MISSED OPPORTUNITY: Not verifying exception properties
Assert.Throws<ArgumentException>(() => DoSomething(null));

// BETTER: Capture and verify the exception
var exception = Assert.Throws<ArgumentException>(() => DoSomething(null));
Assert.Equal("param", exception.ParamName);
Assert.Contains("cannot be null", exception.Message);
```

### 5. Collection Equality Pitfalls

```csharp
// WRONG: Comparing different collection types may have unexpected behavior
var expected = new HashSet<int> { 1, 2, 3 };
var actual = new List<int> { 3, 2, 1 };
Assert.Equal(expected, actual); // Fails! Order matters for Assert.Equal

// CORRECT for order-independent comparison:
Assert.Equivalent(expected, actual);

// CORRECT if you want ordered comparison:
Assert.Equal(expected.OrderBy(x => x), actual.OrderBy(x => x));
```

### 6. Assert.Throws Does Not Catch Inner Async Exceptions

```csharp
// WRONG: Does not catch exceptions from async methods
Assert.Throws<InvalidOperationException>(() => service.ProcessAsync());
// This catches AggregateException, not the inner exception!

// CORRECT: Use ThrowsAsync for async methods
await Assert.ThrowsAsync<InvalidOperationException>(
    () => service.ProcessAsync());
```
