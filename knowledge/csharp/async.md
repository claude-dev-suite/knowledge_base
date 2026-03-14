# C# Asynchronous Programming

> Official Documentation: https://learn.microsoft.com/dotnet/csharp/asynchronous-programming/

## Table of Contents

1. [Async/Await Fundamentals](#asyncawait-fundamentals)
2. [Task and Task T](#task-and-task-t)
3. [ValueTask and ValueTask T](#valuetask-and-valuetask-t)
4. [IAsyncEnumerable](#iasyncenumerable)
5. [CancellationToken Patterns](#cancellationtoken-patterns)
6. [Task.WhenAll and Task.WhenAny](#taskwhenall-and-taskwhenany)
7. [Parallel.ForEachAsync](#parallelforeachasync)
8. [ConfigureAwait Considerations](#configureawait-considerations)
9. [SemaphoreSlim for Async Synchronization](#semaphoreslim-for-async-synchronization)
10. [Channel T for Producer/Consumer](#channel-t-for-producerconsumer)
11. [Async Streams](#async-streams)
12. [Exception Handling in Async Code](#exception-handling-in-async-code)
13. [Async Disposal](#async-disposal)
14. [Fire-and-Forget Patterns](#fire-and-forget-patterns)
15. [TaskCompletionSource](#taskcompletionsource)
16. [Best Practices and Common Pitfalls](#best-practices-and-common-pitfalls)

---

## Async/Await Fundamentals

The `async` and `await` keywords enable asynchronous programming without blocking threads.

```csharp
// Basic async method
public async Task<string> GetDataAsync()
{
    using var client = new HttpClient();
    var response = await client.GetStringAsync("https://api.example.com/data");
    return response;
}

// Async method in ASP.NET Core controller
[HttpGet("{id:int}")]
public async Task<ActionResult<ProductDto>> GetProduct(int id)
{
    var product = await context.Products.FindAsync(id);
    if (product is null) return NotFound();

    return new ProductDto(product.Id, product.Name, product.Price);
}

// Async in a console app
var app = WebApplication.Create(args);
app.MapGet("/", async () =>
{
    await Task.Delay(100);
    return "Hello, World!";
});
app.Run();
```

### How Async/Await Works

```csharp
public async Task<int> ProcessAsync()
{
    Console.WriteLine("1. Before await");  // Runs synchronously
    await Task.Delay(1000);                 // Suspends, returns control to caller
    Console.WriteLine("2. After await");    // Continuation runs after delay
    return 42;                              // Wraps result in Task<int>
}
```

Key rules:
- `async` enables the `await` keyword in the method body
- `await` suspends execution until the awaited task completes
- The method returns immediately to the caller at the first `await`
- The continuation runs on the original synchronization context (in UI/ASP.NET) or thread pool

---

## Task and Task T

`Task` represents an asynchronous operation. `Task<T>` represents one that returns a value.

```csharp
// Task (no return value)
public async Task SaveDataAsync(Data data)
{
    await repository.SaveAsync(data);
    await cache.InvalidateAsync(data.Key);
}

// Task<T> (returns a value)
public async Task<List<Product>> GetProductsAsync()
{
    return await context.Products.ToListAsync();
}

// Already completed tasks (no allocation)
public Task<int> GetCachedCountAsync()
{
    if (_cache.TryGetValue("count", out int count))
        return Task.FromResult(count); // No async machinery needed

    return GetCountFromDatabaseAsync();
}

// Task.CompletedTask for void async
public Task HandleAsync(Event e)
{
    if (!ShouldHandle(e))
        return Task.CompletedTask;

    return ProcessEventAsync(e);
}
```

### Task Status

```csharp
var task = ProcessAsync();

Console.WriteLine(task.Status);        // WaitingForActivation, Running, etc.
Console.WriteLine(task.IsCompleted);   // true when RanToCompletion, Faulted, or Canceled
Console.WriteLine(task.IsCompletedSuccessfully);
Console.WriteLine(task.IsFaulted);
Console.WriteLine(task.IsCanceled);
```

---

## ValueTask and ValueTask T

`ValueTask<T>` avoids heap allocation when the result is often available synchronously. Use it for hot paths where the result may be cached.

```csharp
// Good use: method often returns synchronously from cache
public ValueTask<Product?> GetProductAsync(int id)
{
    if (_cache.TryGetValue(id, out Product? cached))
        return ValueTask.FromResult(cached); // No allocation

    return new ValueTask<Product?>(LoadFromDatabaseAsync(id)); // Wraps Task<T>
}

private async Task<Product?> LoadFromDatabaseAsync(int id)
{
    var product = await context.Products.FindAsync(id);
    if (product is not null)
        _cache.Set(id, product);
    return product;
}

// ValueTask for void operations
public ValueTask ProcessAsync(Message message)
{
    if (_processed.Contains(message.Id))
        return ValueTask.CompletedTask;

    return new ValueTask(ProcessInternalAsync(message));
}
```

### When to Use ValueTask vs Task

| Scenario | Use | Reason |
|----------|-----|--------|
| Interface/abstract method | `Task<T>` | More flexible for implementers |
| Rarely synchronous | `Task<T>` | ValueTask adds complexity |
| Frequently synchronous (cache hit) | `ValueTask<T>` | Avoids allocation on hot path |
| High-performance library API | `ValueTask<T>` | Reduces GC pressure |
| Multiple awaits on same result | `Task<T>` | ValueTask can only be awaited once |

### ValueTask Rules

```csharp
var vt = GetProductAsync(1);

// WRONG: Cannot await ValueTask more than once
// var a = await vt;
// var b = await vt; // Undefined behavior!

// WRONG: Cannot use GetAwaiter().GetResult() while incomplete
// var result = vt.GetAwaiter().GetResult();

// If you need to await multiple times, convert to Task
var task = vt.AsTask();
var a = await task;
var b = await task; // OK
```

---

## IAsyncEnumerable

`IAsyncEnumerable<T>` enables asynchronous iteration, producing elements one at a time.

```csharp
// Produce items asynchronously
public async IAsyncEnumerable<Product> GetProductsStreamAsync(
    [EnumeratorCancellation] CancellationToken cancellationToken = default)
{
    var pageSize = 100;
    var lastId = 0;

    while (true)
    {
        var products = await context.Products
            .Where(p => p.Id > lastId)
            .OrderBy(p => p.Id)
            .Take(pageSize)
            .AsNoTracking()
            .ToListAsync(cancellationToken);

        if (products.Count == 0)
            yield break;

        foreach (var product in products)
        {
            yield return product;
            lastId = product.Id;
        }
    }
}

// Consume with await foreach
await foreach (var product in GetProductsStreamAsync())
{
    Console.WriteLine($"{product.Name}: {product.Price:C}");
}

// With cancellation
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
await foreach (var product in GetProductsStreamAsync(cts.Token))
{
    ProcessProduct(product);
}
```

### ASP.NET Core Streaming Response

```csharp
[HttpGet("stream")]
public async IAsyncEnumerable<ProductDto> StreamProducts(
    [EnumeratorCancellation] CancellationToken cancellationToken)
{
    await foreach (var product in productService.GetProductsStreamAsync(cancellationToken))
    {
        yield return new ProductDto(product.Id, product.Name, product.Price);
    }
}
```

### LINQ with IAsyncEnumerable

```csharp
// System.Linq.Async NuGet package
using System.Linq;

var expensiveProducts = await GetProductsStreamAsync()
    .Where(p => p.Price > 100)
    .Take(10)
    .ToListAsync();
```

---

## CancellationToken Patterns

```csharp
// Pass token through the entire call chain
public async Task<List<Product>> SearchAsync(
    string query,
    CancellationToken cancellationToken = default)
{
    var results = await context.Products
        .Where(p => p.Name.Contains(query))
        .ToListAsync(cancellationToken);

    return results;
}

// Check for cancellation in loops
public async Task ProcessBatchAsync(
    IEnumerable<Order> orders,
    CancellationToken cancellationToken = default)
{
    foreach (var order in orders)
    {
        cancellationToken.ThrowIfCancellationRequested();
        await ProcessOrderAsync(order, cancellationToken);
    }
}

// Create and manage CancellationTokenSource
public async Task RunWithTimeoutAsync()
{
    using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));

    try
    {
        await LongRunningOperationAsync(cts.Token);
    }
    catch (OperationCanceledException)
    {
        Console.WriteLine("Operation timed out");
    }
}

// Linked tokens (cancel if either source cancels)
public async Task HandleRequestAsync(
    HttpContext httpContext,
    CancellationToken appShutdown)
{
    using var linked = CancellationTokenSource.CreateLinkedTokenSource(
        httpContext.RequestAborted,
        appShutdown);

    await ProcessAsync(linked.Token);
}
```

### CancellationToken in ASP.NET Core

```csharp
[HttpGet]
public async Task<IActionResult> Get(CancellationToken cancellationToken)
{
    // ASP.NET Core automatically provides a token linked to request abort
    var data = await service.GetDataAsync(cancellationToken);
    return Ok(data);
}
```

---

## Task.WhenAll and Task.WhenAny

### Task.WhenAll - Concurrent Execution

```csharp
// Run independent tasks concurrently
public async Task<DashboardData> GetDashboardAsync()
{
    var productsTask = GetProductCountAsync();
    var ordersTask = GetRecentOrdersAsync();
    var revenueTask = GetTotalRevenueAsync();

    // All three run concurrently
    await Task.WhenAll(productsTask, ordersTask, revenueTask);

    return new DashboardData(
        ProductCount: await productsTask,
        RecentOrders: await ordersTask,
        TotalRevenue: await revenueTask);
}

// WhenAll with same return type
public async Task<Product[]> GetMultipleProductsAsync(int[] ids)
{
    var tasks = ids.Select(id => context.Products.FindAsync(id).AsTask());
    var products = await Task.WhenAll(tasks);
    return products.Where(p => p is not null).ToArray()!;
}

// WhenAll with exception handling
public async Task ProcessAllAsync(List<string> urls)
{
    var tasks = urls.Select(ProcessUrlAsync).ToList();

    try
    {
        await Task.WhenAll(tasks);
    }
    catch
    {
        // Check individual task results
        foreach (var task in tasks.Where(t => t.IsFaulted))
        {
            Console.WriteLine($"Failed: {task.Exception?.InnerException?.Message}");
        }
    }
}
```

### Task.WhenAny - First to Complete

```csharp
// Timeout pattern
public async Task<string?> GetWithTimeoutAsync(string url, TimeSpan timeout)
{
    using var client = new HttpClient();
    var dataTask = client.GetStringAsync(url);
    var timeoutTask = Task.Delay(timeout);

    var completed = await Task.WhenAny(dataTask, timeoutTask);

    if (completed == dataTask)
        return await dataTask;

    return null; // Timed out
}

// First successful from multiple sources
public async Task<WeatherData> GetWeatherAsync(double lat, double lon)
{
    var tasks = new[]
    {
        GetFromOpenWeatherAsync(lat, lon),
        GetFromWeatherApiAsync(lat, lon),
        GetFromAccuWeatherAsync(lat, lon)
    };

    while (tasks.Length > 0)
    {
        var completed = await Task.WhenAny(tasks);
        if (completed.IsCompletedSuccessfully)
            return await completed;

        tasks = tasks.Where(t => t != completed).ToArray();
    }

    throw new InvalidOperationException("All weather sources failed");
}
```

---

## Parallel.ForEachAsync

.NET 6+ provides `Parallel.ForEachAsync` for bounded concurrent async operations.

```csharp
// Process items with controlled parallelism
var urls = new[] { "https://api1.com", "https://api2.com", "https://api3.com" };

await Parallel.ForEachAsync(urls,
    new ParallelOptions { MaxDegreeOfParallelism = 3 },
    async (url, cancellationToken) =>
    {
        using var client = new HttpClient();
        var data = await client.GetStringAsync(url, cancellationToken);
        await ProcessDataAsync(data, cancellationToken);
    });

// With CancellationToken
using var cts = new CancellationTokenSource(TimeSpan.FromMinutes(5));

await Parallel.ForEachAsync(
    productIds,
    new ParallelOptions
    {
        MaxDegreeOfParallelism = Environment.ProcessorCount,
        CancellationToken = cts.Token
    },
    async (productId, ct) =>
    {
        await using var context = await contextFactory.CreateDbContextAsync(ct);
        var product = await context.Products.FindAsync([productId], ct);
        if (product is not null)
        {
            product.LastIndexed = DateTime.UtcNow;
            await context.SaveChangesAsync(ct);
        }
    });
```

---

## ConfigureAwait Considerations

```csharp
// Library code: use ConfigureAwait(false) to avoid capturing sync context
public async Task<byte[]> ReadFileAsync(string path)
{
    var bytes = await File.ReadAllBytesAsync(path).ConfigureAwait(false);
    return bytes; // Continues on thread pool thread
}

// ASP.NET Core: ConfigureAwait(false) is generally NOT needed
// ASP.NET Core has no SynchronizationContext by default
[HttpGet]
public async Task<IActionResult> Get()
{
    var data = await service.GetDataAsync(); // No ConfigureAwait needed
    return Ok(data);
}
```

| Scenario | Use `ConfigureAwait(false)`? | Reason |
|----------|------------------------------|--------|
| Library code (NuGet package) | Yes | Library shouldn't assume context |
| ASP.NET Core controllers | No | No SynchronizationContext exists |
| WPF/WinForms UI code | No | Need context to update UI |
| Blazor Server | No | Need context for rendering |
| Console app | No | No SynchronizationContext |

---

## SemaphoreSlim for Async Synchronization

`SemaphoreSlim` provides async-compatible mutual exclusion and throttling.

```csharp
// Mutex pattern (single-access async lock)
public class CacheService
{
    private readonly SemaphoreSlim _lock = new(1, 1);
    private readonly Dictionary<string, object> _cache = new();

    public async Task<T> GetOrCreateAsync<T>(string key, Func<Task<T>> factory)
    {
        await _lock.WaitAsync();
        try
        {
            if (_cache.TryGetValue(key, out var cached))
                return (T)cached;

            var value = await factory();
            _cache[key] = value!;
            return value;
        }
        finally
        {
            _lock.Release();
        }
    }
}

// Throttling pattern (limit concurrent operations)
public class ApiClient
{
    private readonly SemaphoreSlim _throttle = new(5, 5); // Max 5 concurrent
    private readonly HttpClient _httpClient = new();

    public async Task<string> GetAsync(string url, CancellationToken ct = default)
    {
        await _throttle.WaitAsync(ct);
        try
        {
            return await _httpClient.GetStringAsync(url, ct);
        }
        finally
        {
            _throttle.Release();
        }
    }

    public async Task<string[]> GetManyAsync(string[] urls, CancellationToken ct = default)
    {
        var tasks = urls.Select(url => GetAsync(url, ct));
        return await Task.WhenAll(tasks);
    }
}
```

---

## Channel T for Producer/Consumer

`Channel<T>` is a high-performance async producer/consumer queue.

```csharp
using System.Threading.Channels;

// Bounded channel (backpressure when full)
var channel = Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait,
    SingleReader = false,
    SingleWriter = false
});

// Producer
async Task ProduceAsync(ChannelWriter<WorkItem> writer, CancellationToken ct)
{
    try
    {
        var id = 0;
        while (!ct.IsCancellationRequested)
        {
            var item = new WorkItem(++id, $"Task {id}");
            await writer.WriteAsync(item, ct);
            Console.WriteLine($"Produced: {item.Name}");
        }
    }
    finally
    {
        writer.Complete();
    }
}

// Consumer
async Task ConsumeAsync(ChannelReader<WorkItem> reader, CancellationToken ct)
{
    await foreach (var item in reader.ReadAllAsync(ct))
    {
        Console.WriteLine($"Processing: {item.Name}");
        await ProcessWorkItemAsync(item, ct);
    }
}

public record WorkItem(int Id, string Name);

// Run producer and consumer concurrently
using var cts = new CancellationTokenSource();

var producer = ProduceAsync(channel.Writer, cts.Token);
var consumers = Enumerable.Range(0, 3)
    .Select(_ => ConsumeAsync(channel.Reader, cts.Token));

await Task.WhenAll(consumers.Append(producer));
```

---

## Exception Handling in Async Code

### Basic Exception Handling

```csharp
try
{
    var result = await GetDataAsync();
}
catch (HttpRequestException ex) when (ex.StatusCode == System.Net.HttpStatusCode.NotFound)
{
    Console.WriteLine("Resource not found");
}
catch (TaskCanceledException)
{
    Console.WriteLine("Request was cancelled");
}
catch (Exception ex)
{
    Console.WriteLine($"Unexpected error: {ex.Message}");
}
```

### Task.WhenAll Exception Handling

```csharp
// WhenAll throws AggregateException containing all failures
var tasks = new[]
{
    Task.FromException(new InvalidOperationException("Error 1")),
    Task.FromException(new ArgumentException("Error 2")),
    Task.FromResult("success")
};

try
{
    await Task.WhenAll(tasks);
}
catch (Exception ex)
{
    // 'ex' is the FIRST exception only
    Console.WriteLine(ex.Message); // "Error 1"

    // Access ALL exceptions
    var allExceptions = tasks
        .Where(t => t.IsFaulted)
        .SelectMany(t => t.Exception!.InnerExceptions);

    foreach (var inner in allExceptions)
    {
        Console.WriteLine($"  - {inner.Message}");
    }
}
```

### Exception in Async Void (Dangerous!)

```csharp
// BAD: Exceptions in async void crash the process
async void BadHandler(object sender, EventArgs e)
{
    await Task.Delay(100);
    throw new Exception("This crashes the app!"); // Unobserved exception
}

// GOOD: Return Task instead
async Task GoodHandler(object sender, EventArgs e)
{
    await Task.Delay(100);
    throw new Exception("This can be caught by the caller");
}
```

---

## Async Disposal

`IAsyncDisposable` supports async cleanup of resources.

```csharp
public class DatabaseConnection : IAsyncDisposable
{
    private readonly SqlConnection _connection;
    private bool _disposed;

    public DatabaseConnection(string connectionString)
    {
        _connection = new SqlConnection(connectionString);
    }

    public async Task OpenAsync(CancellationToken ct = default)
    {
        await _connection.OpenAsync(ct);
    }

    public async ValueTask DisposeAsync()
    {
        if (_disposed) return;

        if (_connection.State == System.Data.ConnectionState.Open)
        {
            await _connection.CloseAsync();
        }

        await _connection.DisposeAsync();
        _disposed = true;

        GC.SuppressFinalize(this);
    }
}

// Usage with await using
await using var connection = new DatabaseConnection(connectionString);
await connection.OpenAsync();
// ... use connection ...
// Automatically disposed asynchronously at end of scope
```

### Implementing Both IDisposable and IAsyncDisposable

```csharp
public class FileProcessor : IDisposable, IAsyncDisposable
{
    private readonly StreamWriter _writer;
    private bool _disposed;

    public FileProcessor(string path)
    {
        _writer = new StreamWriter(path);
    }

    public void Dispose()
    {
        if (_disposed) return;
        _writer.Dispose();
        _disposed = true;
        GC.SuppressFinalize(this);
    }

    public async ValueTask DisposeAsync()
    {
        if (_disposed) return;
        await _writer.DisposeAsync();
        _disposed = true;
        GC.SuppressFinalize(this);
    }
}
```

---

## Fire-and-Forget Patterns

### Why to Avoid Fire-and-Forget

```csharp
// BAD: Unobserved exceptions, no lifecycle management
public void ProcessOrder(Order order)
{
    _ = SendNotificationAsync(order); // Fire and forget - exceptions lost!
}

// BETTER: Use a background queue
public class OrderService
{
    private readonly IBackgroundTaskQueue _queue;

    public OrderService(IBackgroundTaskQueue queue) => _queue = queue;

    public async Task ProcessOrderAsync(Order order)
    {
        await SaveOrderAsync(order);

        // Queue background work safely
        await _queue.QueueAsync(async ct =>
        {
            await SendNotificationAsync(order);
            await UpdateAnalyticsAsync(order);
        });
    }
}
```

---

## TaskCompletionSource

`TaskCompletionSource<T>` creates a `Task<T>` that you manually complete. Useful for wrapping callback-based APIs.

```csharp
// Wrap a callback-based API
public Task<string> GetDataCallbackAsync()
{
    var tcs = new TaskCompletionSource<string>();

    var client = new LegacyClient();
    client.OnSuccess += (data) => tcs.TrySetResult(data);
    client.OnError += (ex) => tcs.TrySetException(ex);
    client.OnCancel += () => tcs.TrySetCanceled();
    client.BeginRequest();

    return tcs.Task;
}

// Manual signal (async gate)
public class AsyncGate
{
    private TaskCompletionSource _tcs = new();

    public Task WaitAsync() => _tcs.Task;

    public void Open() => _tcs.TrySetResult();

    public void Reset() => _tcs = new TaskCompletionSource();
}

// Timeout wrapper
public static async Task<T> WithTimeoutAsync<T>(
    Func<CancellationToken, Task<T>> operation,
    TimeSpan timeout)
{
    using var cts = new CancellationTokenSource(timeout);
    try
    {
        return await operation(cts.Token);
    }
    catch (OperationCanceledException) when (cts.IsCancellationRequested)
    {
        throw new TimeoutException($"Operation timed out after {timeout}");
    }
}
```

---

## Best Practices and Common Pitfalls

### Best Practices

| Practice | Description |
|----------|-------------|
| Async all the way | Don't mix sync and async; propagate async through the call chain |
| Always pass `CancellationToken` | Enable callers to cancel long operations |
| Use `Task.WhenAll` for concurrent work | Run independent operations in parallel |
| Return `Task` directly when possible | Skip `async/await` if just returning a task |
| Use `ValueTask<T>` for hot cache paths | Avoid allocation when result is often synchronous |
| Use `Channel<T>` for producer/consumer | High-performance, bounded, backpressure-aware |
| Implement `IAsyncDisposable` | Clean up async resources properly |
| Use `Parallel.ForEachAsync` for bounded parallelism | Built-in throttling and cancellation |

### Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| `async void` | Exceptions crash the process | Use `async Task` (except for event handlers) |
| `.Result` or `.Wait()` | Deadlocks in UI/ASP.NET contexts | Use `await` instead |
| Forgetting `await` | Task runs but result is not observed | Always `await` or store the task |
| No `CancellationToken` | Cannot cancel long operations | Add `CancellationToken` parameter |
| Unbounded parallelism | Thread pool starvation, OOM | Use `SemaphoreSlim` or `Parallel.ForEachAsync` |
| `Task.Run` in ASP.NET Core | Wastes thread pool threads | Use `await` directly (no `Task.Run`) |
| Fire-and-forget | Lost exceptions, lifecycle issues | Use background queues or channels |
| Awaiting in a loop sequentially | Sequential execution when concurrent is possible | Collect tasks, use `Task.WhenAll` |
| Multiple `await` on `ValueTask` | Undefined behavior | Await only once, or convert to `Task` |
| Capturing loop variable | All tasks use same variable value | Use local copy or modern `foreach` |
