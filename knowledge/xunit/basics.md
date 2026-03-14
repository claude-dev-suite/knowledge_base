# xUnit Basics

> Official Documentation: https://xunit.net/docs/getting-started/netcore/cmdline

Comprehensive reference for xUnit.net -- the modern, extensible testing framework for .NET. xUnit is the default testing framework recommended by the .NET team and is used extensively in ASP.NET Core and Entity Framework Core projects. This guide covers xUnit v2 and v3 patterns targeting .NET 8+.

---

## Table of Contents

1. [Project Setup](#project-setup)
2. [Fact Tests](#fact-tests)
3. [Theory Tests](#theory-tests)
4. [InlineData](#inlinedata)
5. [MemberData](#memberdata)
6. [ClassData](#classdata)
7. [Test Lifecycle](#test-lifecycle)
8. [IAsyncLifetime](#iasynclifetime)
9. [Skip and Conditional Tests](#skip-and-conditional-tests)
10. [Test Output](#test-output)
11. [Test Ordering and Parallelism](#test-ordering-and-parallelism)
12. [Trait-Based Categorization](#trait-based-categorization)
13. [dotnet test CLI](#dotnet-test-cli)
14. [Best Practices](#best-practices)
15. [Common Pitfalls](#common-pitfalls)

---

## Project Setup

### Test Project (.csproj)

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <IsPackable>false</IsPackable>
    <IsTestProject>true</IsTestProject>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.11.*" />
    <PackageReference Include="xunit" Version="2.9.*" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.8.*">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
    </PackageReference>
    <PackageReference Include="coverlet.collector" Version="6.0.*">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
    </PackageReference>
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\MyApp\MyApp.csproj" />
  </ItemGroup>
</Project>
```

### Creating a Test Project via CLI

```bash
# Create a new xUnit test project
dotnet new xunit -n MyApp.Tests

# Add reference to the project under test
dotnet add MyApp.Tests reference MyApp

# Add additional packages
dotnet add MyApp.Tests package FluentAssertions
dotnet add MyApp.Tests package NSubstitute
dotnet add MyApp.Tests package Testcontainers
```

### Solution Structure

```
MyApp.sln
├── src/
│   └── MyApp/
│       ├── MyApp.csproj
│       ├── Services/
│       └── Models/
└── tests/
    ├── MyApp.UnitTests/
    │   └── MyApp.UnitTests.csproj
    ├── MyApp.IntegrationTests/
    │   └── MyApp.IntegrationTests.csproj
    └── MyApp.FunctionalTests/
        └── MyApp.FunctionalTests.csproj
```

---

## Fact Tests

`[Fact]` marks a method as a test with no parameters:

```csharp
using Xunit;

public class CalculatorTests
{
    [Fact]
    public void Add_TwoPositiveNumbers_ReturnsSum()
    {
        // Arrange
        var calculator = new Calculator();

        // Act
        var result = calculator.Add(2, 3);

        // Assert
        Assert.Equal(5, result);
    }

    [Fact]
    public void Divide_ByZero_ThrowsDivideByZeroException()
    {
        var calculator = new Calculator();

        Assert.Throws<DivideByZeroException>(() => calculator.Divide(10, 0));
    }

    [Fact]
    public async Task GetUserAsync_ExistingId_ReturnsUser()
    {
        // Arrange
        var service = new UserService(CreateMockRepository());

        // Act
        var user = await service.GetByIdAsync(1);

        // Assert
        Assert.NotNull(user);
        Assert.Equal("Alice", user.Name);
    }
}
```

### Naming Conventions

| Convention | Example |
|-----------|---------|
| `MethodName_Scenario_ExpectedResult` | `Add_TwoPositiveNumbers_ReturnsSum` |
| `Should_ExpectedBehavior_When_Condition` | `Should_ReturnNull_When_UserNotFound` |
| `Given_When_Then` | `GivenValidInput_WhenProcessed_ThenReturnsSuccess` |

---

## Theory Tests

`[Theory]` marks a parameterized test that runs once per data set:

```csharp
public class StringValidatorTests
{
    [Theory]
    [InlineData("hello@example.com", true)]
    [InlineData("invalid-email", false)]
    [InlineData("", false)]
    [InlineData(null, false)]
    [InlineData("user@domain.co.uk", true)]
    public void IsValidEmail_VariousInputs_ReturnsExpected(string? email, bool expected)
    {
        var validator = new StringValidator();

        var result = validator.IsValidEmail(email);

        Assert.Equal(expected, result);
    }
}
```

---

## InlineData

Provide simple inline test parameters directly on the attribute:

```csharp
public class MathTests
{
    [Theory]
    [InlineData(1, 1, 2)]
    [InlineData(0, 0, 0)]
    [InlineData(-1, 1, 0)]
    [InlineData(int.MaxValue, 1, (long)int.MaxValue + 1)]
    public void Add_VariousInputs_ReturnsExpectedSum(int a, int b, long expected)
    {
        var calculator = new Calculator();

        var result = calculator.AddLong(a, b);

        Assert.Equal(expected, result);
    }

    [Theory]
    [InlineData("hello", "HELLO")]
    [InlineData("World", "WORLD")]
    [InlineData("", "")]
    [InlineData("123abc", "123ABC")]
    public void ToUpperCase_VariousStrings_ReturnsUppercased(
        string input, string expected)
    {
        Assert.Equal(expected, input.ToUpper());
    }
}
```

### InlineData Limitations

| Supported Types | Not Supported |
|----------------|---------------|
| Primitive types (int, string, bool, double, etc.) | Complex objects |
| null | DateTime (use MemberData) |
| enum values | Custom classes |
| Type (typeof) | Collections (use MemberData) |

---

## MemberData

Use `[MemberData]` for complex or dynamically generated test data:

### Static Property

```csharp
public class OrderValidatorTests
{
    public static IEnumerable<object[]> ValidOrders =>
    [
        [new Order { Total = 10.00m, Items = 1 }, true],
        [new Order { Total = 0.01m, Items = 1 }, true],
        [new Order { Total = 99999.99m, Items = 100 }, true],
    ];

    public static IEnumerable<object[]> InvalidOrders =>
    [
        [new Order { Total = 0m, Items = 0 }, "Order must have at least one item"],
        [new Order { Total = -1m, Items = 1 }, "Total cannot be negative"],
        [new Order { Total = 100001m, Items = 1 }, "Total exceeds maximum"],
    ];

    [Theory]
    [MemberData(nameof(ValidOrders))]
    public void Validate_ValidOrder_ReturnsTrue(Order order, bool expected)
    {
        var validator = new OrderValidator();

        var result = validator.IsValid(order);

        Assert.Equal(expected, result);
    }

    [Theory]
    [MemberData(nameof(InvalidOrders))]
    public void Validate_InvalidOrder_ReturnsExpectedError(
        Order order, string expectedError)
    {
        var validator = new OrderValidator();

        var result = validator.Validate(order);

        Assert.False(result.IsValid);
        Assert.Contains(expectedError, result.Errors.Select(e => e.ErrorMessage));
    }
}
```

### Static Method with Parameters

```csharp
public class RangeTests
{
    public static IEnumerable<object[]> GetRangeData(int start, int count)
    {
        for (int i = start; i < start + count; i++)
        {
            yield return [i, i * i];
        }
    }

    [Theory]
    [MemberData(nameof(GetRangeData), 1, 5)]
    public void Square_ReturnsCorrectValue(int input, int expected)
    {
        Assert.Equal(expected, input * input);
    }
}
```

### MemberData from External Class

```csharp
public class TestDataSource
{
    public static IEnumerable<object[]> UserData =>
    [
        [new User("Alice", 30), "Alice is 30 years old"],
        [new User("Bob", 25), "Bob is 25 years old"],
    ];
}

public class UserFormatterTests
{
    [Theory]
    [MemberData(nameof(TestDataSource.UserData), MemberType = typeof(TestDataSource))]
    public void Format_ReturnsExpectedString(User user, string expected)
    {
        var formatter = new UserFormatter();

        var result = formatter.Format(user);

        Assert.Equal(expected, result);
    }
}
```

### TheoryData<T> (Strongly Typed)

```csharp
public class DiscountCalculatorTests
{
    public static TheoryData<decimal, CustomerTier, decimal> DiscountData => new()
    {
        { 100m, CustomerTier.Regular, 0m },
        { 100m, CustomerTier.Silver, 5m },
        { 100m, CustomerTier.Gold, 10m },
        { 100m, CustomerTier.Platinum, 15m },
        { 0m, CustomerTier.Platinum, 0m },
    };

    [Theory]
    [MemberData(nameof(DiscountData))]
    public void Calculate_ReturnsExpectedDiscount(
        decimal orderTotal, CustomerTier tier, decimal expectedDiscount)
    {
        var calculator = new DiscountCalculator();

        var discount = calculator.Calculate(orderTotal, tier);

        Assert.Equal(expectedDiscount, discount);
    }
}
```

---

## ClassData

Encapsulate test data in a separate, reusable class:

```csharp
public class PasswordStrengthData : IEnumerable<object[]>
{
    public IEnumerator<object[]> GetEnumerator()
    {
        yield return ["short", PasswordStrength.Weak];
        yield return ["alllowercase", PasswordStrength.Weak];
        yield return ["Medium1pass", PasswordStrength.Medium];
        yield return ["Str0ng!P@ssw0rd", PasswordStrength.Strong];
        yield return ["V3ry$tr0ng!P@ss#2024", PasswordStrength.VeryStrong];
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

public class PasswordValidatorTests
{
    [Theory]
    [ClassData(typeof(PasswordStrengthData))]
    public void AnalyzeStrength_ReturnsExpectedLevel(
        string password, PasswordStrength expected)
    {
        var validator = new PasswordValidator();

        var result = validator.AnalyzeStrength(password);

        Assert.Equal(expected, result);
    }
}
```

### ClassData with TheoryData<T>

```csharp
public class DateRangeTestData : TheoryData<DateTime, DateTime, bool>
{
    public DateRangeTestData()
    {
        var now = new DateTime(2024, 6, 15);
        Add(now, now.AddDays(30), true);          // Valid range
        Add(now, now, false);                      // Same date
        Add(now.AddDays(1), now, false);           // End before start
        Add(now, now.AddYears(2), false);          // Exceeds max range
    }
}

public class DateRangeValidatorTests
{
    [Theory]
    [ClassData(typeof(DateRangeTestData))]
    public void IsValid_ReturnsExpected(DateTime start, DateTime end, bool expected)
    {
        var validator = new DateRangeValidator(maxRangeDays: 365);

        var result = validator.IsValid(start, end);

        Assert.Equal(expected, result);
    }
}
```

---

## Test Lifecycle

### Constructor and Dispose (Per-Test Setup/Teardown)

xUnit creates a **new instance** of the test class for each test method. The constructor acts as setup, `Dispose` acts as teardown:

```csharp
public class DatabaseTests : IDisposable
{
    private readonly SqliteConnection _connection;
    private readonly AppDbContext _context;

    public DatabaseTests()
    {
        // Setup: runs before each test
        _connection = new SqliteConnection("DataSource=:memory:");
        _connection.Open();

        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlite(_connection)
            .Options;

        _context = new AppDbContext(options);
        _context.Database.EnsureCreated();
    }

    [Fact]
    public void AddUser_SavesSuccessfully()
    {
        _context.Users.Add(new User { Name = "Alice", Email = "alice@test.com" });
        _context.SaveChanges();

        Assert.Single(_context.Users);
    }

    [Fact]
    public void GetUser_ReturnsNull_WhenNotFound()
    {
        var user = _context.Users.FirstOrDefault(u => u.Id == 999);

        Assert.Null(user);
    }

    public void Dispose()
    {
        // Teardown: runs after each test
        _context.Dispose();
        _connection.Dispose();
    }
}
```

---

## IAsyncLifetime

For asynchronous setup and teardown, implement `IAsyncLifetime`:

```csharp
public class ApiIntegrationTests : IAsyncLifetime
{
    private readonly HttpClient _client;
    private TestServer _server = null!;
    private string _authToken = string.Empty;

    public ApiIntegrationTests()
    {
        _client = new HttpClient();
    }

    public async Task InitializeAsync()
    {
        // Async setup: runs before each test
        _server = await TestServerBuilder.CreateAsync();
        _client.BaseAddress = _server.BaseAddress;

        var loginResponse = await _client.PostAsJsonAsync("/api/auth/login",
            new { Username = "admin", Password = "test123" });
        var token = await loginResponse.Content.ReadFromJsonAsync<TokenResponse>();
        _authToken = token!.AccessToken;
        _client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", _authToken);
    }

    [Fact]
    public async Task GetProducts_ReturnsOk()
    {
        var response = await _client.GetAsync("/api/products");

        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
    }

    [Fact]
    public async Task CreateProduct_Returns201()
    {
        var product = new { Name = "Widget", Price = 9.99 };

        var response = await _client.PostAsJsonAsync("/api/products", product);

        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
    }

    public async Task DisposeAsync()
    {
        // Async teardown: runs after each test
        _client.Dispose();
        await _server.DisposeAsync();
    }
}
```

---

## Skip and Conditional Tests

### Skip with Reason

```csharp
[Fact(Skip = "Requires external service - disabled for CI")]
public async Task SendEmail_DeliversSuccessfully()
{
    // This test is always skipped
}

[Theory(Skip = "Known bug #1234 - pending fix")]
[InlineData(1)]
[InlineData(2)]
public void BuggyMethod_ShouldWork(int value)
{
    // All data rows skipped
}
```

### Conditional Skip at Runtime

```csharp
public class ConditionalTests
{
    [Fact]
    public void OnlyOnWindows()
    {
        Skip.IfNot(RuntimeInformation.IsOSPlatform(OSPlatform.Windows),
            "This test only runs on Windows");

        // Windows-specific test logic
        var path = Environment.GetFolderPath(Environment.SpecialFolder.ProgramFiles);
        Assert.NotEmpty(path);
    }

    [Fact]
    public void RequiresEnvironmentVariable()
    {
        var connectionString = Environment.GetEnvironmentVariable("TEST_DB_CONNECTION");
        Skip.If(string.IsNullOrEmpty(connectionString),
            "TEST_DB_CONNECTION environment variable not set");

        // Test that requires the connection string
    }
}
```

---

## Test Output

### ITestOutputHelper

xUnit captures test output through `ITestOutputHelper`, not `Console.WriteLine`:

```csharp
public class DiagnosticTests
{
    private readonly ITestOutputHelper _output;

    public DiagnosticTests(ITestOutputHelper output)
    {
        _output = output;
    }

    [Fact]
    public void ProcessData_LogsSteps()
    {
        _output.WriteLine("Starting test at {0}", DateTime.UtcNow);

        var processor = new DataProcessor();
        var data = GenerateTestData(100);

        _output.WriteLine("Processing {0} records", data.Count);
        var result = processor.Process(data);

        _output.WriteLine("Result: {0} processed, {1} errors",
            result.Processed, result.Errors);

        Assert.Equal(0, result.Errors);
    }

    [Theory]
    [InlineData("input1", "expected1")]
    [InlineData("input2", "expected2")]
    public void Transform_LogsInputAndOutput(string input, string expected)
    {
        _output.WriteLine($"Input: '{input}', Expected: '{expected}'");

        var result = Transformer.Transform(input);

        _output.WriteLine($"Actual result: '{result}'");
        Assert.Equal(expected, result);
    }
}
```

---

## Test Ordering and Parallelism

### Default Parallelism

By default, xUnit runs test classes in parallel but methods within a class sequentially.

| Level | Default Behavior |
|-------|-----------------|
| Across assemblies | Parallel |
| Across classes | Parallel |
| Within a class | Sequential |

### Disabling Parallelism

```csharp
// xunit.runner.json (place in test project root)
{
  "$schema": "https://xunit.net/schema/current/xunit.runner.schema.json",
  "parallelizeAssembly": false,
  "parallelizeTestCollections": false,
  "maxParallelThreads": 1
}
```

Ensure it is copied to output:

```xml
<!-- In .csproj -->
<ItemGroup>
  <Content Include="xunit.runner.json" CopyToOutputDirectory="PreserveNewest" />
</ItemGroup>
```

### Test Collections (Shared Sequential Group)

Tests in the same collection run sequentially. Tests in different collections run in parallel:

```csharp
[Collection("Database")]
public class UserRepositoryTests
{
    // Runs sequentially with other tests in "Database" collection
}

[Collection("Database")]
public class OrderRepositoryTests
{
    // Runs sequentially with UserRepositoryTests
}

[Collection("API")]
public class ProductApiTests
{
    // Runs in parallel with "Database" collection
}
```

### Custom Test Ordering

```csharp
[TestCaseOrderer(
    "MyApp.Tests.PriorityOrderer",
    "MyApp.Tests")]
public class OrderedTests
{
    [Fact]
    [TestPriority(1)]
    public void FirstTest() { }

    [Fact]
    [TestPriority(2)]
    public void SecondTest() { }

    [Fact]
    [TestPriority(3)]
    public void ThirdTest() { }
}

// Custom attribute
[AttributeUsage(AttributeTargets.Method)]
public class TestPriorityAttribute(int priority) : Attribute
{
    public int Priority { get; } = priority;
}

// Custom orderer
public class PriorityOrderer : ITestCaseOrderer
{
    public IEnumerable<TTestCase> OrderTestCases<TTestCase>(
        IEnumerable<TTestCase> testCases) where TTestCase : ITestCase
    {
        return testCases.OrderBy(tc =>
        {
            var attr = tc.TestMethod.Method
                .GetCustomAttributes(typeof(TestPriorityAttribute))
                .FirstOrDefault();
            return attr?.GetNamedArgument<int>("Priority") ?? int.MaxValue;
        });
    }
}
```

---

## Trait-Based Categorization

### Built-in Trait Attribute

```csharp
[Trait("Category", "Unit")]
public class UnitTests
{
    [Fact]
    [Trait("Category", "Unit")]
    [Trait("Feature", "Authentication")]
    public void Login_ValidCredentials_ReturnsToken() { }

    [Fact]
    [Trait("Category", "Unit")]
    [Trait("Feature", "Authorization")]
    public void CheckPermission_AdminRole_Granted() { }
}

[Trait("Category", "Integration")]
public class IntegrationTests
{
    [Fact]
    [Trait("Speed", "Slow")]
    public async Task Database_Migration_Succeeds() { }
}
```

### Running Tests by Trait

```bash
# Run only unit tests
dotnet test --filter "Category=Unit"

# Run only integration tests
dotnet test --filter "Category=Integration"

# Run tests for a specific feature
dotnet test --filter "Feature=Authentication"

# Exclude slow tests
dotnet test --filter "Speed!=Slow"

# Combine filters
dotnet test --filter "Category=Unit&Feature=Authentication"
```

---

## dotnet test CLI

### Common Commands

```bash
# Run all tests
dotnet test

# Run specific project
dotnet test tests/MyApp.UnitTests

# Run with verbose output
dotnet test --verbosity detailed

# Run with specific filter
dotnet test --filter "FullyQualifiedName~UserService"
dotnet test --filter "ClassName=UserServiceTests"
dotnet test --filter "DisplayName~Login"

# Run and collect coverage
dotnet test --collect:"XPlat Code Coverage"

# Run with specific logger
dotnet test --logger "console;verbosity=detailed"
dotnet test --logger "trx;LogFileName=results.trx"

# Run with timeout (per test)
dotnet test -- xunit.methodDisplay=method

# Fail fast on first failure
dotnet test -- RunConfiguration.StopOnFirstFailure=true

# Run in parallel with thread limit
dotnet test -- xunit.maxParallelThreads=4
```

### Filter Expressions

| Expression | Example |
|-----------|---------|
| `FullyQualifiedName~` | `--filter "FullyQualifiedName~Calculator"` |
| `ClassName=` | `--filter "ClassName=CalculatorTests"` |
| `Name=` | `--filter "Name=Add_ReturnsSum"` |
| `Category=` | `--filter "Category=Unit"` |
| Logical AND | `--filter "Category=Unit&ClassName=CalcTests"` |
| Logical OR | `--filter "Category=Unit\|Category=Integration"` |
| Logical NOT | `--filter "Category!=Slow"` |

---

## Best Practices

### Test Organization

| Guideline | Description |
|-----------|-------------|
| Mirror source structure | `src/Services/UserService.cs` -> `tests/Services/UserServiceTests.cs` |
| One test class per class under test | `UserService` -> `UserServiceTests` |
| Use descriptive test names | Name should describe scenario and expected outcome |
| Follow AAA pattern | Arrange, Act, Assert in every test |
| Keep tests independent | No test should depend on another test's execution |

### Arrange-Act-Assert Pattern

```csharp
[Fact]
public async Task CreateOrder_ValidItems_ReturnsOrderWithTotal()
{
    // Arrange
    var orderService = new OrderService(
        new FakeInventoryService(),
        new FakePricingService());
    var items = new[]
    {
        new OrderItem("SKU-001", 2),
        new OrderItem("SKU-002", 1),
    };

    // Act
    var order = await orderService.CreateAsync(items);

    // Assert
    Assert.NotNull(order);
    Assert.Equal(2, order.Items.Count);
    Assert.True(order.Total > 0);
    Assert.Equal(OrderStatus.Created, order.Status);
}
```

### Test Data Builders

```csharp
public class UserBuilder
{
    private string _name = "John Doe";
    private string _email = "john@example.com";
    private UserRole _role = UserRole.User;
    private bool _isActive = true;

    public UserBuilder WithName(string name) { _name = name; return this; }
    public UserBuilder WithEmail(string email) { _email = email; return this; }
    public UserBuilder WithRole(UserRole role) { _role = role; return this; }
    public UserBuilder Inactive() { _isActive = false; return this; }

    public User Build() => new()
    {
        Name = _name,
        Email = _email,
        Role = _role,
        IsActive = _isActive,
        CreatedAt = DateTime.UtcNow,
    };
}

// Usage in tests
[Fact]
public void DeactivateUser_SetsInactive()
{
    var user = new UserBuilder()
        .WithName("Alice")
        .WithRole(UserRole.Admin)
        .Build();

    user.Deactivate();

    Assert.False(user.IsActive);
}
```

---

## Common Pitfalls

### 1. Sharing State Between Tests

```csharp
// WRONG: Static or shared mutable state
public class BadTests
{
    private static List<string> _items = []; // Shared across tests!

    [Fact]
    public void Test1()
    {
        _items.Add("item1");
        Assert.Single(_items); // May fail if Test2 runs first
    }

    [Fact]
    public void Test2()
    {
        _items.Add("item2");
        Assert.Single(_items); // May fail if Test1 runs first
    }
}

// CORRECT: Each test gets its own instance
public class GoodTests
{
    private readonly List<string> _items = [];

    [Fact]
    public void Test1()
    {
        _items.Add("item1");
        Assert.Single(_items); // Always passes
    }

    [Fact]
    public void Test2()
    {
        _items.Add("item2");
        Assert.Single(_items); // Always passes
    }
}
```

### 2. Using Console.WriteLine for Output

```csharp
// WRONG: Output is swallowed by xUnit
[Fact]
public void MyTest()
{
    Console.WriteLine("Debug info"); // Not captured in test output
}

// CORRECT: Use ITestOutputHelper
public class MyTests(ITestOutputHelper output)
{
    [Fact]
    public void MyTest()
    {
        output.WriteLine("Debug info"); // Visible in test results
    }
}
```

### 3. Not Disposing Resources

```csharp
// WRONG: Resources leak between tests
public class LeakyTests
{
    [Fact]
    public void Test()
    {
        var connection = new SqlConnection(connString);
        connection.Open();
        // connection never closed/disposed
    }
}

// CORRECT: Implement IDisposable
public class ProperTests : IDisposable
{
    private readonly SqlConnection _connection;

    public ProperTests()
    {
        _connection = new SqlConnection(connString);
        _connection.Open();
    }

    [Fact]
    public void Test()
    {
        // Use _connection
    }

    public void Dispose()
    {
        _connection.Dispose();
    }
}
```

### 4. Relying on Test Execution Order

```csharp
// WRONG: Test2 depends on Test1 creating data
[Fact]
public void Test1_CreateUser() { /* creates user in DB */ }

[Fact]
public void Test2_GetUser() { /* expects user from Test1 */ }

// CORRECT: Each test is self-contained
[Fact]
public async Task GetUser_ExistingUser_ReturnsUser()
{
    // Arrange -- create the user this test needs
    await _repository.AddAsync(new User { Id = 1, Name = "Alice" });

    // Act
    var user = await _repository.GetByIdAsync(1);

    // Assert
    Assert.NotNull(user);
}
```

### 5. Async Void Tests

```csharp
// WRONG: async void -- xUnit cannot await completion
[Fact]
public async void WrongAsyncTest()
{
    var result = await service.GetAsync();
    Assert.NotNull(result); // May not execute before test ends
}

// CORRECT: async Task
[Fact]
public async Task CorrectAsyncTest()
{
    var result = await service.GetAsync();
    Assert.NotNull(result);
}
```
