# xUnit Fixtures

> Official Documentation: https://xunit.net/docs/shared-context

Comprehensive reference for xUnit.net fixtures and shared context -- mechanisms for sharing setup and teardown logic across tests. Fixtures let you run expensive initialization once (database seeding, container startup, HTTP server creation) and share the result across multiple test methods or even multiple test classes, while maintaining test isolation.

---

## Table of Contents

1. [Constructor Injection (Per-Test)](#constructor-injection-per-test)
2. [IClassFixture (Class-Level)](#iclassfixture-class-level)
3. [ICollectionFixture (Cross-Class)](#icollectionfixture-cross-class)
4. [IAsyncLifetime for Fixtures](#iasynclifetime-for-fixtures)
5. [IDisposable and IAsyncDisposable Cleanup](#idisposable-and-iasyncdisposable-cleanup)
6. [Database Fixtures](#database-fixtures)
7. [WebApplicationFactory for Integration Tests](#webapplicationfactory-for-integration-tests)
8. [HttpClient Fixture Patterns](#httpclient-fixture-patterns)
9. [Fixture Dependencies](#fixture-dependencies)
10. [TestContainers Patterns](#testcontainers-patterns)
11. [Best Practices](#best-practices)
12. [Common Pitfalls](#common-pitfalls)

---

## Constructor Injection (Per-Test)

xUnit creates a new instance of the test class for every test method. The constructor runs before each test, and `Dispose` runs after:

```csharp
public class UserServiceTests : IDisposable
{
    private readonly UserService _service;
    private readonly Mock<IUserRepository> _mockRepo;
    private readonly Mock<IEmailService> _mockEmail;

    public UserServiceTests()
    {
        // Runs before EACH test
        _mockRepo = new Mock<IUserRepository>();
        _mockEmail = new Mock<IEmailService>();
        _service = new UserService(_mockRepo.Object, _mockEmail.Object);
    }

    [Fact]
    public async Task GetById_ExistingUser_ReturnsUser()
    {
        _mockRepo.Setup(r => r.GetByIdAsync(1))
            .ReturnsAsync(new User { Id = 1, Name = "Alice" });

        var user = await _service.GetByIdAsync(1);

        Assert.NotNull(user);
        Assert.Equal("Alice", user.Name);
    }

    [Fact]
    public async Task Create_SendsWelcomeEmail()
    {
        _mockRepo.Setup(r => r.AddAsync(It.IsAny<User>()))
            .ReturnsAsync((User u) => u);

        await _service.CreateAsync("Bob", "bob@test.com");

        _mockEmail.Verify(e => e.SendWelcomeEmailAsync("bob@test.com"), Times.Once);
    }

    public void Dispose()
    {
        // Runs after EACH test
        // Cleanup if needed
    }
}
```

### When to Use Constructor Injection

| Scenario | Appropriate |
|----------|-------------|
| Setting up mock objects | Yes |
| Creating lightweight in-memory instances | Yes |
| Seeding an in-memory database | Yes |
| Starting a Docker container | No (use IClassFixture) |
| Creating an HTTP server | No (use IClassFixture) |

---

## IClassFixture (Class-Level)

`IClassFixture<T>` creates a single fixture instance shared across all tests in **one test class**. The fixture is created before any test in the class runs and disposed after all tests complete.

### Defining a Fixture

```csharp
public class DatabaseFixture : IAsyncLifetime
{
    public AppDbContext DbContext { get; private set; } = null!;
    private SqliteConnection _connection = null!;

    public async Task InitializeAsync()
    {
        _connection = new SqliteConnection("DataSource=:memory:");
        await _connection.OpenAsync();

        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlite(_connection)
            .Options;

        DbContext = new AppDbContext(options);
        await DbContext.Database.EnsureCreatedAsync();

        // Seed shared test data
        DbContext.Users.AddRange(
            new User { Id = 1, Name = "Alice", Email = "alice@test.com", IsActive = true },
            new User { Id = 2, Name = "Bob", Email = "bob@test.com", IsActive = true },
            new User { Id = 3, Name = "Charlie", Email = "charlie@test.com", IsActive = false }
        );
        await DbContext.SaveChangesAsync();
    }

    public async Task DisposeAsync()
    {
        await DbContext.DisposeAsync();
        await _connection.DisposeAsync();
    }
}
```

### Using the Fixture

```csharp
public class UserRepositoryTests : IClassFixture<DatabaseFixture>
{
    private readonly DatabaseFixture _fixture;
    private readonly UserRepository _repository;

    public UserRepositoryTests(DatabaseFixture fixture)
    {
        _fixture = fixture;
        _repository = new UserRepository(fixture.DbContext);
    }

    [Fact]
    public async Task GetAll_ReturnsAllUsers()
    {
        var users = await _repository.GetAllAsync();

        Assert.Equal(3, users.Count);
    }

    [Fact]
    public async Task GetActive_ReturnsOnlyActiveUsers()
    {
        var users = await _repository.GetActiveAsync();

        Assert.Equal(2, users.Count);
        Assert.All(users, u => Assert.True(u.IsActive));
    }

    [Fact]
    public async Task GetById_ExistingUser_ReturnsUser()
    {
        var user = await _repository.GetByIdAsync(1);

        Assert.NotNull(user);
        Assert.Equal("Alice", user.Name);
    }
}
```

### Fixture Lifecycle

```
DatabaseFixture.InitializeAsync()     -- runs once before all tests in class
  |
  |-- UserRepositoryTests.ctor()       -- new instance per test
  |-- Test1()
  |-- UserRepositoryTests.Dispose()
  |
  |-- UserRepositoryTests.ctor()       -- new instance per test
  |-- Test2()
  |-- UserRepositoryTests.Dispose()
  |
  |-- UserRepositoryTests.ctor()       -- new instance per test
  |-- Test3()
  |-- UserRepositoryTests.Dispose()
  |
DatabaseFixture.DisposeAsync()         -- runs once after all tests in class
```

---

## ICollectionFixture (Cross-Class)

`ICollectionFixture<T>` shares a single fixture instance across **multiple test classes**. This is ideal for expensive resources like database containers or HTTP servers.

### Defining a Collection

```csharp
// 1. Define the collection (no test methods here)
[CollectionDefinition("Integration")]
public class IntegrationTestCollection : ICollectionFixture<IntegrationTestFixture>
{
    // This class has no code -- it only serves as the collection anchor
}

// 2. Define the fixture
public class IntegrationTestFixture : IAsyncLifetime
{
    public WebApplicationFactory<Program> Factory { get; private set; } = null!;
    public HttpClient Client { get; private set; } = null!;
    public string ConnectionString { get; private set; } = string.Empty;

    public async Task InitializeAsync()
    {
        // Start a Testcontainers PostgreSQL instance
        var postgres = new PostgreSqlBuilder()
            .WithImage("postgres:16-alpine")
            .Build();
        await postgres.StartAsync();
        ConnectionString = postgres.GetConnectionString();

        // Create the WebApplicationFactory
        Factory = new WebApplicationFactory<Program>()
            .WithWebHostBuilder(builder =>
            {
                builder.ConfigureServices(services =>
                {
                    // Replace the database with the test container
                    services.RemoveAll<DbContextOptions<AppDbContext>>();
                    services.AddDbContext<AppDbContext>(options =>
                        options.UseNpgsql(ConnectionString));
                });
            });

        Client = Factory.CreateClient();

        // Run migrations and seed
        using var scope = Factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.MigrateAsync();
        await SeedDataAsync(db);
    }

    private async Task SeedDataAsync(AppDbContext db)
    {
        db.Users.Add(new User { Name = "Admin", Email = "admin@test.com", Role = "Admin" });
        await db.SaveChangesAsync();
    }

    public async Task DisposeAsync()
    {
        Client.Dispose();
        await Factory.DisposeAsync();
    }
}
```

### Using the Collection in Test Classes

```csharp
// 3. Use [Collection] attribute to reference the shared fixture
[Collection("Integration")]
public class UserApiTests
{
    private readonly IntegrationTestFixture _fixture;

    public UserApiTests(IntegrationTestFixture fixture)
    {
        _fixture = fixture;
    }

    [Fact]
    public async Task GetUsers_ReturnsOk()
    {
        var response = await _fixture.Client.GetAsync("/api/users");

        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
    }

    [Fact]
    public async Task CreateUser_Returns201()
    {
        var payload = new { Name = "New User", Email = "new@test.com" };

        var response = await _fixture.Client.PostAsJsonAsync("/api/users", payload);

        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
    }
}

[Collection("Integration")]
public class ProductApiTests
{
    private readonly IntegrationTestFixture _fixture;

    public ProductApiTests(IntegrationTestFixture fixture)
    {
        _fixture = fixture;
    }

    [Fact]
    public async Task GetProducts_ReturnsOk()
    {
        var response = await _fixture.Client.GetAsync("/api/products");

        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
    }
}
```

### Collection vs Class Fixture Comparison

| Aspect | IClassFixture | ICollectionFixture |
|--------|--------------|-------------------|
| Scope | One test class | Multiple test classes |
| Shared instance | Same fixture for all tests in one class | Same fixture for all tests in all classes in the collection |
| Declaration | `IClassFixture<T>` on the class | `[Collection("name")]` on each class, `ICollectionFixture<T>` on the definition |
| Parallelism | Tests in the class run sequentially | Tests across classes in the same collection run sequentially |
| Use case | Class-specific database, service mock | Shared database container, web server |

---

## IAsyncLifetime for Fixtures

### Async Class Fixture

```csharp
public class ElasticsearchFixture : IAsyncLifetime
{
    private readonly ElasticsearchContainer _container;
    public ElasticClient Client { get; private set; } = null!;

    public ElasticsearchFixture()
    {
        _container = new ElasticsearchBuilder()
            .WithImage("docker.elastic.co/elasticsearch/elasticsearch:8.11.0")
            .Build();
    }

    public async Task InitializeAsync()
    {
        await _container.StartAsync();

        var settings = new ElasticsearchClientSettings(
            new Uri(_container.GetConnectionString()))
            .DefaultIndex("test-index");

        Client = new ElasticClient(settings);

        // Wait for cluster to be ready
        var healthResponse = await Client.Cluster.HealthAsync(h => h
            .WaitForStatus(WaitForStatus.Yellow)
            .Timeout("30s"));

        if (!healthResponse.IsValid)
            throw new Exception("Elasticsearch cluster is not healthy");
    }

    public async Task DisposeAsync()
    {
        await _container.DisposeAsync();
    }
}
```

### Async Test Class with IAsyncLifetime

```csharp
public class SearchServiceTests : IClassFixture<ElasticsearchFixture>, IAsyncLifetime
{
    private readonly ElasticsearchFixture _fixture;
    private readonly SearchService _searchService;

    public SearchServiceTests(ElasticsearchFixture fixture)
    {
        _fixture = fixture;
        _searchService = new SearchService(fixture.Client);
    }

    public async Task InitializeAsync()
    {
        // Per-test async setup: seed index with test data
        await _searchService.IndexAsync(new Document
        {
            Id = "doc-1",
            Title = "Getting Started with Elasticsearch",
            Content = "A comprehensive guide...",
        });
        await _fixture.Client.Indices.RefreshAsync("test-index");
    }

    [Fact]
    public async Task Search_MatchingQuery_ReturnsResults()
    {
        var results = await _searchService.SearchAsync("elasticsearch");

        Assert.NotEmpty(results);
        Assert.Contains(results, r => r.Id == "doc-1");
    }

    public async Task DisposeAsync()
    {
        // Per-test async cleanup: remove test data
        await _fixture.Client.Indices.DeleteAsync("test-index");
    }
}
```

---

## IDisposable and IAsyncDisposable Cleanup

### Synchronous Cleanup

```csharp
public class TempFileFixture : IDisposable
{
    public string TempDirectory { get; }

    public TempFileFixture()
    {
        TempDirectory = Path.Combine(Path.GetTempPath(), Guid.NewGuid().ToString());
        Directory.CreateDirectory(TempDirectory);
    }

    public string CreateTempFile(string content, string extension = ".txt")
    {
        var path = Path.Combine(TempDirectory, $"{Guid.NewGuid()}{extension}");
        File.WriteAllText(path, content);
        return path;
    }

    public void Dispose()
    {
        if (Directory.Exists(TempDirectory))
        {
            Directory.Delete(TempDirectory, recursive: true);
        }
    }
}
```

### Async Cleanup

```csharp
public class MessageBusFixture : IAsyncLifetime, IAsyncDisposable
{
    private ServiceBusClient _client = null!;
    private ServiceBusSender _sender = null!;
    private ServiceBusReceiver _receiver = null!;
    public string QueueName { get; } = $"test-queue-{Guid.NewGuid():N}";

    public async Task InitializeAsync()
    {
        var connectionString = Environment.GetEnvironmentVariable("SERVICEBUS_CONNECTION")
            ?? throw new SkipException("SERVICEBUS_CONNECTION not set");

        _client = new ServiceBusClient(connectionString);

        // Create queue for this test run
        var adminClient = new ServiceBusAdministrationClient(connectionString);
        await adminClient.CreateQueueAsync(QueueName);

        _sender = _client.CreateSender(QueueName);
        _receiver = _client.CreateReceiver(QueueName);
    }

    public ServiceBusSender GetSender() => _sender;
    public ServiceBusReceiver GetReceiver() => _receiver;

    public async Task DisposeAsync()
    {
        await _sender.DisposeAsync();
        await _receiver.DisposeAsync();
        await _client.DisposeAsync();

        // Clean up the test queue
        var connectionString = Environment.GetEnvironmentVariable("SERVICEBUS_CONNECTION")!;
        var adminClient = new ServiceBusAdministrationClient(connectionString);
        await adminClient.DeleteQueueAsync(QueueName);
    }

    async ValueTask IAsyncDisposable.DisposeAsync()
    {
        await DisposeAsync();
    }
}
```

---

## Database Fixtures

### SQLite In-Memory Fixture

```csharp
public class SqliteDbFixture : IDisposable
{
    private readonly SqliteConnection _connection;
    public AppDbContext Context { get; }

    public SqliteDbFixture()
    {
        _connection = new SqliteConnection("DataSource=:memory:");
        _connection.Open();

        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlite(_connection)
            .Options;

        Context = new AppDbContext(options);
        Context.Database.EnsureCreated();
    }

    public void SeedWith(Action<AppDbContext> seeder)
    {
        seeder(Context);
        Context.SaveChanges();
        Context.ChangeTracker.Clear(); // Prevent tracking issues
    }

    public void Dispose()
    {
        Context.Dispose();
        _connection.Dispose();
    }
}
```

### Per-Test Database Reset Pattern

For tests that mutate data, reset the database state before each test:

```csharp
public class OrderServiceTests : IClassFixture<DatabaseFixture>, IAsyncLifetime
{
    private readonly DatabaseFixture _fixture;

    public OrderServiceTests(DatabaseFixture fixture)
    {
        _fixture = fixture;
    }

    public async Task InitializeAsync()
    {
        // Reset database to known state before each test
        await _fixture.ResetAsync();
        await _fixture.SeedAsync(db =>
        {
            db.Products.AddRange(
                new Product { Id = 1, Name = "Widget", Price = 9.99m, Stock = 100 },
                new Product { Id = 2, Name = "Gadget", Price = 19.99m, Stock = 50 }
            );
        });
    }

    [Fact]
    public async Task CreateOrder_DeductsStock()
    {
        var service = new OrderService(_fixture.CreateDbContext());

        await service.CreateAsync(new OrderRequest
        {
            Items = [new OrderItem { ProductId = 1, Quantity = 5 }]
        });

        using var verifyContext = _fixture.CreateDbContext();
        var product = await verifyContext.Products.FindAsync(1);
        Assert.Equal(95, product!.Stock);
    }

    public Task DisposeAsync() => Task.CompletedTask;
}
```

---

## WebApplicationFactory for Integration Tests

### Basic WebApplicationFactory Fixture

```csharp
public class ApiFixture : IAsyncLifetime
{
    public WebApplicationFactory<Program> Factory { get; private set; } = null!;
    public HttpClient Client { get; private set; } = null!;

    public async Task InitializeAsync()
    {
        Factory = new WebApplicationFactory<Program>()
            .WithWebHostBuilder(builder =>
            {
                builder.UseEnvironment("Testing");

                builder.ConfigureServices(services =>
                {
                    // Replace real database with in-memory
                    services.RemoveAll<DbContextOptions<AppDbContext>>();
                    services.AddDbContext<AppDbContext>(options =>
                        options.UseInMemoryDatabase("TestDb"));

                    // Replace real email service with fake
                    services.RemoveAll<IEmailService>();
                    services.AddSingleton<IEmailService, FakeEmailService>();
                });
            });

        Client = Factory.CreateClient(new WebApplicationFactoryClientOptions
        {
            AllowAutoRedirect = false,
        });

        // Seed database
        using var scope = Factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.EnsureCreatedAsync();
        db.Users.Add(new User { Id = 1, Name = "Admin", Email = "admin@test.com" });
        await db.SaveChangesAsync();
    }

    public HttpClient CreateAuthenticatedClient(string role = "Admin")
    {
        return Factory.CreateClient();
        // Typically set up authentication headers or test auth handler here
    }

    public T GetService<T>() where T : notnull
    {
        var scope = Factory.Services.CreateScope();
        return scope.ServiceProvider.GetRequiredService<T>();
    }

    public async Task DisposeAsync()
    {
        Client.Dispose();
        await Factory.DisposeAsync();
    }
}
```

### Custom WebApplicationFactory

```csharp
public class CustomWebApplicationFactory : WebApplicationFactory<Program>
{
    private readonly string _connectionString;

    public CustomWebApplicationFactory(string connectionString)
    {
        _connectionString = connectionString;
    }

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseEnvironment("Testing");

        builder.ConfigureServices(services =>
        {
            services.RemoveAll<DbContextOptions<AppDbContext>>();
            services.AddDbContext<AppDbContext>(options =>
                options.UseNpgsql(_connectionString));

            // Add test authentication handler
            services.AddAuthentication("Test")
                .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>(
                    "Test", options => { });
        });

        builder.ConfigureTestServices(services =>
        {
            // Override specific services for testing
            services.AddScoped<IDateTimeProvider, FakeDateTimeProvider>();
        });
    }
}
```

---

## HttpClient Fixture Patterns

### Scoped HttpClient per Test Class

```csharp
[Collection("Integration")]
public class ProductApiTests : IAsyncLifetime
{
    private readonly IntegrationTestFixture _fixture;
    private HttpClient _client = null!;
    private string _authToken = string.Empty;

    public ProductApiTests(IntegrationTestFixture fixture)
    {
        _fixture = fixture;
    }

    public async Task InitializeAsync()
    {
        _client = _fixture.Factory.CreateClient();

        // Authenticate
        var loginResponse = await _client.PostAsJsonAsync("/api/auth/login",
            new { Username = "admin", Password = "password" });
        var tokenResponse = await loginResponse.Content.ReadFromJsonAsync<TokenResponse>();
        _authToken = tokenResponse!.AccessToken;
        _client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", _authToken);
    }

    [Fact]
    public async Task GetProducts_Authenticated_ReturnsOk()
    {
        var response = await _client.GetAsync("/api/products");

        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
        var products = await response.Content.ReadFromJsonAsync<List<ProductDto>>();
        Assert.NotNull(products);
        Assert.NotEmpty(products);
    }

    [Fact]
    public async Task CreateProduct_ValidData_Returns201()
    {
        var payload = new CreateProductRequest
        {
            Name = "New Product",
            Price = 29.99m,
            Category = "Electronics",
        };

        var response = await _client.PostAsJsonAsync("/api/products", payload);

        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        var created = await response.Content.ReadFromJsonAsync<ProductDto>();
        Assert.NotNull(created);
        Assert.Equal("New Product", created.Name);
        Assert.NotEmpty(created.Id);
    }

    public Task DisposeAsync()
    {
        _client.Dispose();
        return Task.CompletedTask;
    }
}
```

---

## Fixture Dependencies

### Fixture Depending on Another Fixture

xUnit does not natively support fixture chaining. Use composition:

```csharp
public class DatabaseContainerFixture : IAsyncLifetime
{
    private PostgreSqlContainer _container = null!;
    public string ConnectionString { get; private set; } = string.Empty;

    public async Task InitializeAsync()
    {
        _container = new PostgreSqlBuilder()
            .WithImage("postgres:16-alpine")
            .Build();
        await _container.StartAsync();
        ConnectionString = _container.GetConnectionString();
    }

    public async Task DisposeAsync()
    {
        await _container.DisposeAsync();
    }
}

// Composed fixture that depends on the database container
public class FullStackFixture : IAsyncLifetime
{
    private readonly DatabaseContainerFixture _dbFixture = new();
    public WebApplicationFactory<Program> Factory { get; private set; } = null!;
    public HttpClient Client { get; private set; } = null!;

    public async Task InitializeAsync()
    {
        // Start the database first
        await _dbFixture.InitializeAsync();

        // Then create the web app using the database connection string
        Factory = new WebApplicationFactory<Program>()
            .WithWebHostBuilder(builder =>
            {
                builder.ConfigureServices(services =>
                {
                    services.RemoveAll<DbContextOptions<AppDbContext>>();
                    services.AddDbContext<AppDbContext>(options =>
                        options.UseNpgsql(_dbFixture.ConnectionString));
                });
            });

        Client = Factory.CreateClient();

        // Run migrations
        using var scope = Factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.MigrateAsync();
    }

    public async Task DisposeAsync()
    {
        Client.Dispose();
        await Factory.DisposeAsync();
        await _dbFixture.DisposeAsync();
    }
}

[CollectionDefinition("FullStack")]
public class FullStackCollection : ICollectionFixture<FullStackFixture> { }
```

---

## TestContainers Patterns

### PostgreSQL Container Fixture

```csharp
public class PostgresFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _container;
    public string ConnectionString => _container.GetConnectionString();

    public PostgresFixture()
    {
        _container = new PostgreSqlBuilder()
            .WithImage("postgres:16-alpine")
            .WithDatabase("testdb")
            .WithUsername("testuser")
            .WithPassword("testpass")
            .WithCleanUp(true)
            .Build();
    }

    public async Task InitializeAsync()
    {
        await _container.StartAsync();
    }

    public AppDbContext CreateDbContext()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql(ConnectionString)
            .Options;
        return new AppDbContext(options);
    }

    public async Task ResetDatabaseAsync()
    {
        await using var context = CreateDbContext();
        await context.Database.EnsureDeletedAsync();
        await context.Database.EnsureCreatedAsync();
    }

    public async Task DisposeAsync()
    {
        await _container.DisposeAsync();
    }
}
```

### Redis Container Fixture

```csharp
public class RedisFixture : IAsyncLifetime
{
    private readonly RedisContainer _container;
    public IConnectionMultiplexer Connection { get; private set; } = null!;
    public IDatabase Database => Connection.GetDatabase();

    public RedisFixture()
    {
        _container = new RedisBuilder()
            .WithImage("redis:7-alpine")
            .Build();
    }

    public async Task InitializeAsync()
    {
        await _container.StartAsync();
        Connection = await ConnectionMultiplexer.ConnectAsync(
            _container.GetConnectionString());
    }

    public async Task FlushAsync()
    {
        var server = Connection.GetServer(Connection.GetEndPoints().First());
        await server.FlushAllDatabasesAsync();
    }

    public async Task DisposeAsync()
    {
        Connection.Dispose();
        await _container.DisposeAsync();
    }
}

[Collection("Redis")]
public class CacheServiceTests : IAsyncLifetime
{
    private readonly RedisFixture _redis;
    private CacheService _cache = null!;

    public CacheServiceTests(RedisFixture redis)
    {
        _redis = redis;
    }

    public async Task InitializeAsync()
    {
        await _redis.FlushAsync(); // Clean slate per test class
        _cache = new CacheService(_redis.Database);
    }

    [Fact]
    public async Task SetAndGet_ReturnsStoredValue()
    {
        await _cache.SetAsync("key1", "value1", TimeSpan.FromMinutes(5));

        var result = await _cache.GetAsync<string>("key1");

        Assert.Equal("value1", result);
    }

    [Fact]
    public async Task Get_ExpiredKey_ReturnsNull()
    {
        await _cache.SetAsync("key2", "value2", TimeSpan.FromMilliseconds(1));
        await Task.Delay(50);

        var result = await _cache.GetAsync<string>("key2");

        Assert.Null(result);
    }

    public Task DisposeAsync() => Task.CompletedTask;
}
```

---

## Best Practices

### Fixture Selection Guide

| Scenario | Fixture Type | Reason |
|----------|-------------|--------|
| Mock objects, lightweight instances | Constructor (per-test) | Fast, no shared state |
| In-memory database for one class | `IClassFixture` | Shared across tests in one class |
| Docker container (DB, Redis, etc.) | `ICollectionFixture` | Expensive startup, share across classes |
| WebApplicationFactory | `ICollectionFixture` | Share HTTP server across API test classes |
| File system temp directory | `IClassFixture` | Create once, clean up once |
| External API mock server | `ICollectionFixture` | Expensive to start, share widely |

### Naming Conventions

| Pattern | Example |
|---------|---------|
| Fixture class | `DatabaseFixture`, `ApiFixture`, `RedisFixture` |
| Collection definition | `IntegrationTestCollection`, `DatabaseCollection` |
| Collection name | `"Integration"`, `"Database"`, `"FullStack"` |

### Data Isolation Strategies

| Strategy | Description | When to Use |
|----------|-------------|-------------|
| Transaction rollback | Wrap each test in a transaction, roll back after | Fast, good for read-heavy tests |
| Database reset | `EnsureDeleted` + `EnsureCreated` before each test | When schema changes between tests |
| Unique data per test | Generate unique IDs/names per test | When tests run in parallel against same DB |
| Snapshot restore | Restore from a database snapshot | Large seed datasets |

```csharp
// Transaction rollback pattern
public class TransactionalTests : IClassFixture<DatabaseFixture>, IAsyncLifetime
{
    private readonly DatabaseFixture _fixture;
    private IDbContextTransaction _transaction = null!;

    public TransactionalTests(DatabaseFixture fixture)
    {
        _fixture = fixture;
    }

    public async Task InitializeAsync()
    {
        _transaction = await _fixture.DbContext.Database.BeginTransactionAsync();
    }

    [Fact]
    public async Task Test_WithRollback()
    {
        _fixture.DbContext.Users.Add(new User { Name = "Test" });
        await _fixture.DbContext.SaveChangesAsync();

        // Assert... the data is visible within the transaction
    }

    public async Task DisposeAsync()
    {
        await _transaction.RollbackAsync();  // All changes are undone
        await _transaction.DisposeAsync();
    }
}
```

---

## Common Pitfalls

### 1. Not Using [Collection] When Needed

```csharp
// WRONG: Two test classes each create their own fixture
// (fixture is created twice, which is wasteful for containers)
public class TestClass1 : IClassFixture<ExpensiveFixture> { }
public class TestClass2 : IClassFixture<ExpensiveFixture> { }

// CORRECT: Share via ICollectionFixture
[CollectionDefinition("Shared")]
public class SharedCollection : ICollectionFixture<ExpensiveFixture> { }

[Collection("Shared")]
public class TestClass1 { }

[Collection("Shared")]
public class TestClass2 { }
```

### 2. Mutating Shared Fixture State

```csharp
// WRONG: Tests modify shared fixture data, causing flaky tests
public class BadTests : IClassFixture<DatabaseFixture>
{
    [Fact]
    public async Task Test1_DeletesUser()
    {
        await _fixture.DbContext.Users.Where(u => u.Id == 1).ExecuteDeleteAsync();
        // Test2 will fail if it expects user 1 to exist!
    }

    [Fact]
    public async Task Test2_GetsUser()
    {
        var user = await _fixture.DbContext.Users.FindAsync(1);
        Assert.NotNull(user); // FLAKY: depends on execution order
    }
}

// CORRECT: Use per-test setup or transaction rollback
public class GoodTests : IClassFixture<DatabaseFixture>, IAsyncLifetime
{
    public async Task InitializeAsync()
    {
        await _fixture.ResetAndSeedAsync();
    }
    // Tests now have consistent starting state
}
```

### 3. Forgetting IAsyncLifetime for Async Setup

```csharp
// WRONG: Constructor cannot be async
public class BadFixture
{
    public BadFixture()
    {
        // Cannot await here!
        // container.StartAsync().Wait(); // Deadlock risk!
    }
}

// CORRECT: Use IAsyncLifetime
public class GoodFixture : IAsyncLifetime
{
    public async Task InitializeAsync()
    {
        await container.StartAsync(); // Proper async setup
    }

    public async Task DisposeAsync()
    {
        await container.DisposeAsync();
    }
}
```

### 4. Missing CollectionDefinition Class

```csharp
// WRONG: [Collection] without a matching [CollectionDefinition]
[Collection("Integration")]
public class MyTests
{
    public MyTests(IntegrationFixture fixture) { } // Fails at runtime!
}

// CORRECT: Must have a matching CollectionDefinition
[CollectionDefinition("Integration")]
public class IntegrationCollection : ICollectionFixture<IntegrationFixture> { }

[Collection("Integration")]
public class MyTests
{
    public MyTests(IntegrationFixture fixture) { } // Works!
}
```

### 5. Fixture Constructor Throwing Exceptions

```csharp
// WRONG: Exception in fixture constructor causes all tests to fail with cryptic error
public class BadFixture
{
    public BadFixture()
    {
        var value = Environment.GetEnvironmentVariable("REQUIRED_VAR")
            ?? throw new Exception("REQUIRED_VAR not set");
    }
}

// CORRECT: Use IAsyncLifetime and skip if preconditions not met
public class GoodFixture : IAsyncLifetime
{
    public async Task InitializeAsync()
    {
        var value = Environment.GetEnvironmentVariable("REQUIRED_VAR");
        if (string.IsNullOrEmpty(value))
        {
            // Log clearly so developers know why tests are skipped
            throw new SkipException("REQUIRED_VAR not set - skipping integration tests");
        }
        await SetupAsync(value);
    }

    public Task DisposeAsync() => Task.CompletedTask;
}
```

### 6. Parallel Test Conflicts with Shared State

```csharp
// WRONG: Tests in different classes modify same database table in parallel
// (xUnit runs test classes in parallel by default)

// CORRECT: Put classes that share mutable state in the same collection
// (collections run sequentially within themselves)
[Collection("Database")]
public class UserTests { }

[Collection("Database")]
public class OrderTests { }
// UserTests and OrderTests run sequentially, avoiding conflicts
```
