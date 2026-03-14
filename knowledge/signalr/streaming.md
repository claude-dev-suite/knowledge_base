# SignalR Streaming

> Official Documentation: https://learn.microsoft.com/aspnet/core/signalr/streaming

SignalR supports streaming data between client and server. Server-to-client streaming sends data as it becomes available. Client-to-server streaming allows clients to push data in chunks. Both directions use `IAsyncEnumerable<T>` or `ChannelReader<T>`/`ChannelWriter<T>`.

---

## 1. Server-to-Client Streaming

### Using IAsyncEnumerable (Recommended for .NET 8+)

```csharp
public class DataHub : Hub
{
    // Simplest streaming approach using IAsyncEnumerable
    public async IAsyncEnumerable<int> CounterStream(
        int count,
        int delayMs,
        [EnumeratorCancellation] CancellationToken cancellationToken)
    {
        for (var i = 0; i < count; i++)
        {
            cancellationToken.ThrowIfCancellationRequested();
            yield return i;
            await Task.Delay(delayMs, cancellationToken);
        }
    }
}
```

### Using ChannelReader

```csharp
public class DataHub : Hub
{
    // Channel-based streaming for more control over the pipeline
    public ChannelReader<StockPrice> StreamStockPrices(
        string[] symbols,
        CancellationToken cancellationToken)
    {
        var channel = Channel.CreateUnbounded<StockPrice>();

        _ = WriteStockPricesAsync(channel.Writer, symbols, cancellationToken);

        return channel.Reader;
    }

    private async Task WriteStockPricesAsync(
        ChannelWriter<StockPrice> writer,
        string[] symbols,
        CancellationToken cancellationToken)
    {
        try
        {
            var random = new Random();
            while (!cancellationToken.IsCancellationRequested)
            {
                foreach (var symbol in symbols)
                {
                    var price = new StockPrice
                    {
                        Symbol = symbol,
                        Price = 100m + random.Next(-500, 500) / 100m,
                        Timestamp = DateTime.UtcNow
                    };
                    await writer.WriteAsync(price, cancellationToken);
                }
                await Task.Delay(1000, cancellationToken);
            }
        }
        catch (OperationCanceledException)
        {
            // Client disconnected or cancelled
        }
        catch (Exception ex)
        {
            writer.TryComplete(ex);
            return;
        }
        finally
        {
            writer.TryComplete();
        }
    }
}

public record StockPrice
{
    public string Symbol { get; init; } = "";
    public decimal Price { get; init; }
    public DateTime Timestamp { get; init; }
}
```

### IAsyncEnumerable vs ChannelReader

| Feature | `IAsyncEnumerable<T>` | `ChannelReader<T>` |
|---------|----------------------|-------------------|
| Simplicity | Simpler, uses `yield return` | More boilerplate |
| Backpressure | Automatic | Manual via bounded channels |
| Multiple writers | Not supported | Supported |
| Completion control | Automatic on method exit | Manual `TryComplete()` |
| Error handling | Standard try/catch | `TryComplete(exception)` |
| Recommended when | Simple sequential data | Complex pipelines, multi-source |

---

## 2. Consuming Server Streams (JavaScript Client)

### Using stream()

```typescript
// Start streaming
const subscription = connection.stream<number>("CounterStream", 100, 500)
    .subscribe({
        next: (value) => {
            console.log("Received:", value);
            updateUI(value);
        },
        error: (err) => {
            console.error("Stream error:", err);
        },
        complete: () => {
            console.log("Stream completed");
        },
    });

// Cancel the stream from client side
document.getElementById("stopBtn")?.addEventListener("click", () => {
    subscription.dispose();
});
```

### Typed Stream Consumption

```typescript
interface StockPrice {
    symbol: string;
    price: number;
    timestamp: string;
}

function startStockStream(symbols: string[]): signalR.ISubscription<StockPrice> {
    return connection.stream<StockPrice>("StreamStockPrices", symbols).subscribe({
        next: (stockPrice) => {
            updateStockTicker(stockPrice.symbol, stockPrice.price);
        },
        error: (err) => {
            console.error("Stock stream error:", err);
            // Optionally restart the stream
            setTimeout(() => startStockStream(symbols), 5000);
        },
        complete: () => {
            console.log("Stock stream ended");
        },
    });
}

const subscription = startStockStream(["AAPL", "GOOGL", "MSFT"]);
```

---

## 3. Consuming Server Streams (.NET Client)

### Using StreamAsync (IAsyncEnumerable)

```csharp
var stream = connection.StreamAsync<int>("CounterStream", 100, 500);

await foreach (var value in stream)
{
    Console.WriteLine($"Received: {value}");
}
Console.WriteLine("Stream completed");
```

### Using StreamAsChannelAsync

```csharp
var channel = await connection.StreamAsChannelAsync<StockPrice>(
    "StreamStockPrices",
    new[] { "AAPL", "GOOGL" });

while (await channel.WaitToReadAsync())
{
    while (channel.TryRead(out var stockPrice))
    {
        Console.WriteLine($"{stockPrice.Symbol}: ${stockPrice.Price}");
    }
}
```

### Cancelling a Stream

```csharp
using var cts = new CancellationTokenSource();

// Cancel after 30 seconds
cts.CancelAfter(TimeSpan.FromSeconds(30));

try
{
    await foreach (var value in connection.StreamAsync<int>("CounterStream", 1000, 100, cts.Token))
    {
        Console.WriteLine($"Received: {value}");

        // Or cancel based on a condition
        if (value >= 50)
        {
            cts.Cancel();
        }
    }
}
catch (OperationCanceledException)
{
    Console.WriteLine("Stream cancelled by client");
}
```

---

## 4. Client-to-Server Streaming

### Hub Method Accepting a Stream

```csharp
public class UploadHub : Hub
{
    private readonly ILogger<UploadHub> _logger;

    public UploadHub(ILogger<UploadHub> logger) => _logger = logger;

    // Accept IAsyncEnumerable from client
    public async Task UploadData(
        string datasetName,
        IAsyncEnumerable<DataPoint> stream)
    {
        var count = 0;
        await foreach (var point in stream)
        {
            count++;
            // Process each data point as it arrives
            _logger.LogDebug("Received point {Count}: {Value}", count, point.Value);
        }

        _logger.LogInformation("Upload complete: {DatasetName}, {Count} points",
            datasetName, count);

        await Clients.Caller.SendAsync("UploadComplete", datasetName, count);
    }

    // Accept ChannelReader from client
    public async Task UploadStream(ChannelReader<string> stream)
    {
        while (await stream.WaitToReadAsync())
        {
            while (stream.TryRead(out var line))
            {
                // Process each line
                await ProcessLineAsync(line);
            }
        }
    }

    private Task ProcessLineAsync(string line)
    {
        _logger.LogInformation("Processing: {Line}", line);
        return Task.CompletedTask;
    }
}

public record DataPoint(double Value, DateTime Timestamp);
```

### JavaScript Client-to-Server Streaming

```typescript
import * as signalR from "@microsoft/signalr";

// Create a Subject (acts as a writable stream)
const subject = new signalR.Subject<{ value: number; timestamp: string }>();

// Start the streaming invocation
connection.send("UploadData", "sensor-readings", subject);

// Send data points over time
let count = 0;
const interval = setInterval(() => {
    subject.next({
        value: Math.random() * 100,
        timestamp: new Date().toISOString(),
    });
    count++;

    if (count >= 100) {
        subject.complete(); // Signal end of stream
        clearInterval(interval);
    }
}, 100);

// Or complete with an error
// subject.error(new Error("Upload failed"));
```

### .NET Client-to-Server Streaming

```csharp
// Using ChannelWriter
var channel = Channel.CreateBounded<DataPoint>(10);

// Start the upload (runs in background)
await connection.SendAsync("UploadData", "sensor-data", channel.Reader);

// Write data points
for (var i = 0; i < 100; i++)
{
    await channel.Writer.WriteAsync(new DataPoint(
        Random.Shared.NextDouble() * 100,
        DateTime.UtcNow
    ));
    await Task.Delay(100);
}

// Signal completion
channel.Writer.Complete();
```

### .NET Client Streaming with IAsyncEnumerable

```csharp
async IAsyncEnumerable<DataPoint> GenerateDataAsync(
    int count,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    for (var i = 0; i < count; i++)
    {
        ct.ThrowIfCancellationRequested();
        yield return new DataPoint(Random.Shared.NextDouble() * 100, DateTime.UtcNow);
        await Task.Delay(100, ct);
    }
}

// Send the async enumerable as a stream
await connection.SendAsync("UploadData", "sensor-data", GenerateDataAsync(100));
```

---

## 5. Bidirectional Streaming

Combine server-to-client and client-to-server streaming for full-duplex scenarios.

```csharp
public class TransformHub : Hub
{
    // Accept a client stream and return a server stream
    public async IAsyncEnumerable<ProcessedResult> TransformStream(
        IAsyncEnumerable<RawInput> inputStream,
        [EnumeratorCancellation] CancellationToken cancellationToken)
    {
        await foreach (var input in inputStream.WithCancellation(cancellationToken))
        {
            // Transform each input and yield the result
            var result = new ProcessedResult
            {
                OriginalValue = input.Value,
                TransformedValue = input.Value.ToUpperInvariant(),
                ProcessedAt = DateTime.UtcNow
            };
            yield return result;
        }
    }
}

public record RawInput(string Value);
public record ProcessedResult
{
    public string OriginalValue { get; init; } = "";
    public string TransformedValue { get; init; } = "";
    public DateTime ProcessedAt { get; init; }
}
```

---

## 6. Real-Time Data Feed Pattern

A production-ready example for streaming real-time data updates.

```csharp
public interface IDataFeedClient
{
    Task ReceiveSnapshot(List<SensorReading> readings);
}

public class DataFeedHub : Hub<IDataFeedClient>
{
    private readonly ISensorService _sensors;

    public DataFeedHub(ISensorService sensors) => _sensors = sensors;

    // Stream sensor readings for selected sensors
    public async IAsyncEnumerable<SensorReading> StreamSensorData(
        string[] sensorIds,
        int intervalMs = 1000,
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        // Send initial snapshot
        var snapshot = await _sensors.GetLatestReadingsAsync(sensorIds);
        await Clients.Caller.ReceiveSnapshot(snapshot);

        // Stream updates
        while (!cancellationToken.IsCancellationRequested)
        {
            foreach (var sensorId in sensorIds)
            {
                var reading = await _sensors.GetReadingAsync(sensorId, cancellationToken);
                if (reading is not null)
                {
                    yield return reading;
                }
            }
            await Task.Delay(intervalMs, cancellationToken);
        }
    }
}

public record SensorReading
{
    public string SensorId { get; init; } = "";
    public double Value { get; init; }
    public string Unit { get; init; } = "";
    public DateTime Timestamp { get; init; }
    public string Status { get; init; } = "Normal";
}
```

### Client Consumption with UI Updates

```typescript
interface SensorReading {
    sensorId: string;
    value: number;
    unit: string;
    timestamp: string;
    status: string;
}

class SensorDashboard {
    private subscription: signalR.ISubscription<SensorReading> | null = null;
    private readings = new Map<string, SensorReading>();

    start(sensorIds: string[]): void {
        this.subscription = connection
            .stream<SensorReading>("StreamSensorData", sensorIds, 500)
            .subscribe({
                next: (reading) => {
                    this.readings.set(reading.sensorId, reading);
                    this.updateGauge(reading);
                },
                error: (err) => {
                    console.error("Sensor stream error:", err);
                    this.showError("Lost connection to sensor feed");
                    setTimeout(() => this.start(sensorIds), 5000);
                },
                complete: () => {
                    console.log("Sensor stream ended");
                },
            });
    }

    stop(): void {
        this.subscription?.dispose();
        this.subscription = null;
    }

    private updateGauge(reading: SensorReading): void {
        const el = document.getElementById(`sensor-${reading.sensorId}`);
        if (el) {
            el.textContent = `${reading.value.toFixed(2)} ${reading.unit}`;
            el.className = reading.status === "Normal" ? "normal" : "alert";
        }
    }

    private showError(message: string): void {
        console.error(message);
    }
}
```

---

## 7. Progress Reporting Pattern

Use streaming to report progress of long-running operations.

```csharp
public class TaskHub : Hub
{
    private readonly IFileProcessor _processor;

    public TaskHub(IFileProcessor processor) => _processor = processor;

    public async IAsyncEnumerable<ProgressUpdate> ProcessFiles(
        string[] filePaths,
        [EnumeratorCancellation] CancellationToken cancellationToken)
    {
        var totalFiles = filePaths.Length;

        yield return new ProgressUpdate
        {
            Phase = "Starting",
            PercentComplete = 0,
            Message = $"Processing {totalFiles} files..."
        };

        for (var i = 0; i < filePaths.Length; i++)
        {
            cancellationToken.ThrowIfCancellationRequested();

            var filePath = filePaths[i];
            var fileName = Path.GetFileName(filePath);

            yield return new ProgressUpdate
            {
                Phase = "Processing",
                PercentComplete = (int)((double)i / totalFiles * 100),
                Message = $"Processing {fileName}...",
                CurrentItem = fileName
            };

            await _processor.ProcessFileAsync(filePath, cancellationToken);
        }

        yield return new ProgressUpdate
        {
            Phase = "Complete",
            PercentComplete = 100,
            Message = $"All {totalFiles} files processed successfully."
        };
    }
}

public record ProgressUpdate
{
    public string Phase { get; init; } = "";
    public int PercentComplete { get; init; }
    public string Message { get; init; } = "";
    public string? CurrentItem { get; init; }
}
```

### Client Progress Bar

```typescript
interface ProgressUpdate {
    phase: string;
    percentComplete: number;
    message: string;
    currentItem: string | null;
}

function startProcessing(files: string[]): void {
    const progressBar = document.getElementById("progress") as HTMLProgressElement;
    const statusText = document.getElementById("status")!;

    connection.stream<ProgressUpdate>("ProcessFiles", files).subscribe({
        next: (update) => {
            progressBar.value = update.percentComplete;
            statusText.textContent = update.message;
        },
        error: (err) => {
            statusText.textContent = `Error: ${err}`;
            progressBar.classList.add("error");
        },
        complete: () => {
            statusText.textContent = "Done!";
        },
    });
}
```

---

## 8. File Upload Streaming Pattern

Stream file uploads in chunks to handle large files without loading them entirely into memory.

```csharp
public class FileHub : Hub
{
    private readonly IWebHostEnvironment _env;
    private readonly ILogger<FileHub> _logger;

    public FileHub(IWebHostEnvironment env, ILogger<FileHub> logger)
    {
        _env = env;
        _logger = logger;
    }

    public async Task<UploadResult> UploadFile(
        string fileName,
        long fileSize,
        IAsyncEnumerable<byte[]> chunkStream)
    {
        var safeName = Path.GetRandomFileName() + Path.GetExtension(fileName);
        var filePath = Path.Combine(_env.ContentRootPath, "uploads", safeName);

        Directory.CreateDirectory(Path.GetDirectoryName(filePath)!);

        long bytesReceived = 0;

        await using var fileStream = new FileStream(filePath, FileMode.Create, FileAccess.Write);

        await foreach (var chunk in chunkStream)
        {
            await fileStream.WriteAsync(chunk);
            bytesReceived += chunk.Length;

            // Report progress back to caller
            var percent = (int)((double)bytesReceived / fileSize * 100);
            await Clients.Caller.SendAsync("UploadProgress", percent);
        }

        _logger.LogInformation("File uploaded: {FileName} ({Bytes} bytes)", safeName, bytesReceived);

        return new UploadResult
        {
            FileName = safeName,
            Size = bytesReceived,
            Success = true
        };
    }
}

public record UploadResult
{
    public string FileName { get; init; } = "";
    public long Size { get; init; }
    public bool Success { get; init; }
}
```

### JavaScript File Upload Client

```typescript
async function uploadFile(file: File): Promise<void> {
    const chunkSize = 32 * 1024; // 32 KB chunks
    const subject = new signalR.Subject<number[]>();

    // Listen for progress
    connection.on("UploadProgress", (percent: number) => {
        updateProgressBar(percent);
    });

    // Start the upload invocation
    const resultPromise = connection.invoke<{ fileName: string; size: number; success: boolean }>(
        "UploadFile",
        file.name,
        file.size,
        subject,
    );

    // Read file in chunks and send
    const reader = file.stream().getReader();
    try {
        while (true) {
            const { done, value } = await reader.read();
            if (done) break;

            // Convert Uint8Array to number array for SignalR
            subject.next(Array.from(value));
        }
        subject.complete();
    } catch (err) {
        subject.error(err as Error);
    }

    const result = await resultPromise;
    console.log("Upload result:", result);
}
```

---

## 9. Streaming with Groups

Combine streaming with group membership for targeted data distribution.

```csharp
public class DashboardHub : Hub
{
    public async Task SubscribeToDashboard(string dashboardId)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, $"dashboard-{dashboardId}");
    }

    public async Task UnsubscribeFromDashboard(string dashboardId)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, $"dashboard-{dashboardId}");
    }

    // Individual client stream for personalized data
    public async IAsyncEnumerable<MetricPoint> StreamMetrics(
        string metricName,
        int intervalMs = 1000,
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        while (!cancellationToken.IsCancellationRequested)
        {
            yield return new MetricPoint
            {
                Name = metricName,
                Value = Random.Shared.NextDouble() * 100,
                Timestamp = DateTime.UtcNow
            };
            await Task.Delay(intervalMs, cancellationToken);
        }
    }
}

// Background service that pushes to groups (not streaming, but complementary)
public class MetricBroadcastService : BackgroundService
{
    private readonly IHubContext<DashboardHub> _hub;

    public MetricBroadcastService(IHubContext<DashboardHub> hub) => _hub = hub;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var metric = new MetricPoint
            {
                Name = "system-cpu",
                Value = Random.Shared.NextDouble() * 100,
                Timestamp = DateTime.UtcNow
            };

            await _hub.Clients.Group("dashboard-main")
                .SendAsync("ReceiveMetric", metric, stoppingToken);

            await Task.Delay(2000, stoppingToken);
        }
    }
}

public record MetricPoint
{
    public string Name { get; init; } = "";
    public double Value { get; init; }
    public DateTime Timestamp { get; init; }
}
```

---

## 10. Testing Streaming Hubs

### Unit Testing with xUnit

```csharp
using Microsoft.AspNetCore.SignalR;
using Xunit;

public class DataHubTests
{
    [Fact]
    public async Task CounterStream_ReturnsExpectedCount()
    {
        // Arrange
        var hub = new DataHub();
        var count = 5;
        var delayMs = 10;
        using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));

        // Act
        var results = new List<int>();
        await foreach (var value in hub.CounterStream(count, delayMs, cts.Token))
        {
            results.Add(value);
        }

        // Assert
        Assert.Equal(count, results.Count);
        Assert.Equal(new[] { 0, 1, 2, 3, 4 }, results);
    }

    [Fact]
    public async Task CounterStream_RespectsTokenCancellation()
    {
        // Arrange
        var hub = new DataHub();
        using var cts = new CancellationTokenSource();

        // Act
        var results = new List<int>();
        await foreach (var value in hub.CounterStream(1000, 10, cts.Token))
        {
            results.Add(value);
            if (results.Count >= 3)
            {
                cts.Cancel();
            }
        }

        // Assert
        Assert.Equal(3, results.Count);
    }
}
```

### Integration Testing with WebApplicationFactory

```csharp
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.AspNetCore.SignalR.Client;
using Xunit;

public class StreamingIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;

    public StreamingIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory;
    }

    [Fact]
    public async Task StreamStockPrices_ReturnsStreamedData()
    {
        // Arrange
        var server = _factory.Server;
        var connection = new HubConnectionBuilder()
            .WithUrl(
                $"{server.BaseAddress}dataHub",
                options => options.HttpMessageHandlerFactory = _ => server.CreateHandler())
            .Build();

        await connection.StartAsync();

        // Act
        var results = new List<StockPrice>();
        using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));

        await foreach (var price in connection.StreamAsync<StockPrice>(
            "StreamStockPrices",
            new[] { "AAPL" },
            cts.Token))
        {
            results.Add(price);
            if (results.Count >= 3) break;
        }

        // Assert
        Assert.True(results.Count >= 3);
        Assert.All(results, r => Assert.Equal("AAPL", r.Symbol));

        await connection.DisposeAsync();
    }
}
```

---

## 11. Error Handling in Streams

### Server-Side Error Handling

```csharp
public class SafeStreamHub : Hub
{
    public async IAsyncEnumerable<DataResult> StreamWithErrorHandling(
        string source,
        [EnumeratorCancellation] CancellationToken cancellationToken)
    {
        if (string.IsNullOrEmpty(source))
        {
            // Throw before yielding to send error to client
            throw new HubException("Source parameter is required.");
        }

        var errorCount = 0;

        for (var i = 0; i < 100; i++)
        {
            cancellationToken.ThrowIfCancellationRequested();

            DataResult? result;
            try
            {
                result = await FetchDataAsync(source, i);
            }
            catch (Exception ex)
            {
                errorCount++;
                if (errorCount > 5)
                {
                    // Too many errors, terminate the stream
                    throw new HubException($"Stream aborted: too many errors from {source}");
                }
                // Skip this item and continue
                continue;
            }

            yield return result;
            await Task.Delay(100, cancellationToken);
        }
    }

    private Task<DataResult> FetchDataAsync(string source, int index) =>
        Task.FromResult(new DataResult { Index = index, Value = $"{source}-{index}" });
}

public record DataResult
{
    public int Index { get; init; }
    public string Value { get; init; } = "";
}
```

### Client-Side Error Handling

```typescript
connection.stream<DataResult>("StreamWithErrorHandling", "sensor-1").subscribe({
    next: (data) => {
        // Process each item
        appendData(data);
    },
    error: (err) => {
        // Stream terminated with error
        console.error("Stream error:", err);
        showErrorBanner(`Stream failed: ${err}`);

        // Optionally restart the stream
        setTimeout(() => startStream("sensor-1"), 5000);
    },
    complete: () => {
        // Stream completed normally
        showSuccess("Stream completed");
    },
});
```

---

## Best Practices

1. **Use `IAsyncEnumerable<T>` for simple streams** -- Less boilerplate than `ChannelReader<T>`.
2. **Always accept `CancellationToken`** -- Respect client disconnections and cancellations.
3. **Use `[EnumeratorCancellation]`** -- Required attribute on the `CancellationToken` parameter of `IAsyncEnumerable` methods.
4. **Complete channels in `finally`** -- Always call `writer.TryComplete()` to avoid hung streams.
5. **Chunk large data** -- Stream in manageable pieces rather than one huge payload.
6. **Handle stream errors on client** -- Always provide an `error` callback in `subscribe()`.
7. **Use bounded channels for backpressure** -- `Channel.CreateBounded<T>(capacity)` prevents memory issues.
8. **Dispose subscriptions** -- Call `subscription.dispose()` when no longer needed.
9. **Test with cancellation** -- Verify streams stop cleanly when the client disconnects.
10. **Prefer streaming over large single responses** -- Reduces memory usage and time-to-first-byte.

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|---------|
| Missing `[EnumeratorCancellation]` | `CancellationToken` not wired to `IAsyncEnumerable` | Add the attribute to the token parameter |
| Not calling `writer.TryComplete()` | Client stream never ends | Use `try/finally` to always complete the channel |
| Throwing after first `yield return` | Stream aborts with generic error | Use `HubException` for user-friendly messages |
| No error callback on client `subscribe()` | Unhandled promise rejection | Always provide `error` handler |
| Streaming very large items | Memory pressure per message | Break items into smaller chunks |
| Not disposing client subscription | Resource leak, server keeps sending | Call `dispose()` on the subscription |
| Ignoring `CancellationToken` | Server keeps processing after client disconnects | Check `cancellationToken` in every loop iteration |
| Using `invoke` instead of `stream` | Waits for all data before returning | Use `connection.stream()` for server-to-client streaming |
| Unbounded channel with fast producer | Memory exhaustion | Use `Channel.CreateBounded<T>()` with appropriate capacity |
| Missing `WithCancellation` on `await foreach` | `IAsyncEnumerable` ignores caller cancellation in bidirectional scenarios | Pass token via `inputStream.WithCancellation(ct)` |
