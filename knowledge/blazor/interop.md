# Blazor JavaScript Interop

> Official Documentation: https://learn.microsoft.com/aspnet/core/blazor/javascript-interoperability/

Blazor JavaScript interop enables bidirectional communication between .NET and JavaScript. .NET code can call JavaScript functions, and JavaScript can invoke .NET methods. This is essential for accessing browser APIs, integrating third-party JS libraries, and manipulating the DOM.

---

## 1. Calling JavaScript from .NET (IJSRuntime)

### Basic InvokeAsync

```razor
@inject IJSRuntime JS

<button @onclick="ShowAlert">Show Alert</button>
<button @onclick="GetWindowWidth">Get Width</button>
<p>Window width: @windowWidth</p>

@code {
    private int windowWidth;

    private async Task ShowAlert()
    {
        // Call JS function without return value
        await JS.InvokeVoidAsync("alert", "Hello from Blazor!");
    }

    private async Task GetWindowWidth()
    {
        // Call JS function with return value
        windowWidth = await JS.InvokeAsync<int>("eval", "window.innerWidth");
    }
}
```

### Calling Custom JavaScript Functions

```html
<!-- wwwroot/index.html or _Host.cshtml -->
<script>
    window.blazorInterop = {
        showToast: function (message, type) {
            // Using a hypothetical toast library
            toastr[type](message);
        },
        getLocalStorage: function (key) {
            return localStorage.getItem(key);
        },
        setLocalStorage: function (key, value) {
            localStorage.setItem(key, value);
        },
        focusElement: function (elementId) {
            const el = document.getElementById(elementId);
            if (el) el.focus();
        }
    };
</script>
```

```razor
@inject IJSRuntime JS

@code {
    private async Task ShowToast(string message, string type = "info")
    {
        await JS.InvokeVoidAsync("blazorInterop.showToast", message, type);
    }

    private async Task<string?> GetFromStorage(string key)
    {
        return await JS.InvokeAsync<string?>("blazorInterop.getLocalStorage", key);
    }

    private async Task SaveToStorage(string key, string value)
    {
        await JS.InvokeVoidAsync("blazorInterop.setLocalStorage", key, value);
    }

    private async Task FocusElement(string elementId)
    {
        await JS.InvokeVoidAsync("blazorInterop.focusElement", elementId);
    }
}
```

---

## 2. JavaScript Module Isolation (IJSObjectReference)

### Importing JS Modules

```javascript
// wwwroot/js/chart-interop.js
export function createChart(canvasId, data, options) {
    const ctx = document.getElementById(canvasId).getContext("2d");
    return new Chart(ctx, {
        type: options.type || "bar",
        data: data,
        options: options.chartOptions || {},
    });
}

export function updateChart(chartInstance, newData) {
    chartInstance.data = newData;
    chartInstance.update();
}

export function destroyChart(chartInstance) {
    chartInstance.destroy();
}
```

```razor
@inject IJSRuntime JS
@implements IAsyncDisposable

<canvas id="myChart"></canvas>

@code {
    private IJSObjectReference? module;
    private IJSObjectReference? chartInstance;

    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            // Import the JS module
            module = await JS.InvokeAsync<IJSObjectReference>(
                "import", "./js/chart-interop.js");

            // Call an exported function
            chartInstance = await module.InvokeAsync<IJSObjectReference>(
                "createChart", "myChart", GetChartData(), GetChartOptions());
        }
    }

    private async Task UpdateData()
    {
        if (module is not null && chartInstance is not null)
        {
            await module.InvokeVoidAsync("updateChart", chartInstance, GetChartData());
        }
    }

    public async ValueTask DisposeAsync()
    {
        if (chartInstance is not null)
        {
            if (module is not null)
            {
                await module.InvokeVoidAsync("destroyChart", chartInstance);
            }
            await chartInstance.DisposeAsync();
        }
        if (module is not null)
        {
            await module.DisposeAsync();
        }
    }

    private object GetChartData() => new
    {
        labels = new[] { "Jan", "Feb", "Mar", "Apr" },
        datasets = new[]
        {
            new { label = "Sales", data = new[] { 12, 19, 3, 5 } }
        }
    };

    private object GetChartOptions() => new { type = "bar" };
}
```

---

## 3. Collocated JavaScript Files (.razor.js)

In .NET 8+, JavaScript files can be collocated with their component.

```javascript
// Counter.razor.js
export function showConfetti(elementId) {
    const el = document.getElementById(elementId);
    // Confetti animation logic
    el.classList.add("confetti-active");
    setTimeout(() => el.classList.remove("confetti-active"), 2000);
}

export function playSound(url) {
    const audio = new Audio(url);
    audio.play();
}
```

```razor
@* Counter.razor *@
@inject IJSRuntime JS
@implements IAsyncDisposable

<div id="counter-container">
    <p>Count: @count</p>
    <button @onclick="Increment">Increment</button>
</div>

@code {
    private IJSObjectReference? module;
    private int count = 0;

    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            // Path matches the component file with .js extension
            module = await JS.InvokeAsync<IJSObjectReference>(
                "import", "./_content/MyApp/Counter.razor.js");
            // Or for pages in the app: "./Components/Pages/Counter.razor.js"
        }
    }

    private async Task Increment()
    {
        count++;
        if (count % 10 == 0 && module is not null)
        {
            await module.InvokeVoidAsync("showConfetti", "counter-container");
        }
    }

    public async ValueTask DisposeAsync()
    {
        if (module is not null)
        {
            await module.DisposeAsync();
        }
    }
}
```

---

## 4. Calling .NET from JavaScript ([JSInvokable])

### Static .NET Methods

```csharp
public class JsInteropMethods
{
    [JSInvokable]
    public static string GetAppVersion()
    {
        return "1.0.0";
    }

    [JSInvokable]
    public static async Task<WeatherForecast[]> GetForecasts()
    {
        // Simulate async data fetch
        await Task.Delay(100);
        return new[]
        {
            new WeatherForecast { Date = DateTime.Now, Summary = "Sunny", TemperatureC = 25 },
            new WeatherForecast { Date = DateTime.Now.AddDays(1), Summary = "Cloudy", TemperatureC = 20 }
        };
    }
}
```

```javascript
// Calling static .NET method from JavaScript
async function callDotNet() {
    const version = await DotNet.invokeMethodAsync("MyApp", "GetAppVersion");
    console.log("App version:", version);

    const forecasts = await DotNet.invokeMethodAsync("MyApp", "GetForecasts");
    console.log("Forecasts:", forecasts);
}
```

### Instance .NET Methods with DotNetObjectReference

```razor
@inject IJSRuntime JS
@implements IDisposable

<div id="dropzone" class="drop-area">
    Drop files here
</div>

@if (droppedFiles.Any())
{
    <ul>
        @foreach (var file in droppedFiles)
        {
            <li>@file</li>
        }
    </ul>
}

@code {
    private DotNetObjectReference<FileDropZone>? objRef;
    private List<string> droppedFiles = [];

    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            objRef = DotNetObjectReference.Create(this);
            await JS.InvokeVoidAsync("setupDropZone", "dropzone", objRef);
        }
    }

    [JSInvokable]
    public void OnFilesDropped(string[] fileNames)
    {
        droppedFiles.AddRange(fileNames);
        StateHasChanged();
    }

    [JSInvokable]
    public async Task<bool> ValidateFile(string fileName, long fileSize)
    {
        // .NET-side validation called from JS
        var maxSize = 10 * 1024 * 1024; // 10 MB
        var allowedExtensions = new[] { ".jpg", ".png", ".pdf" };
        var extension = Path.GetExtension(fileName).ToLower();

        return fileSize <= maxSize && allowedExtensions.Contains(extension);
    }

    public void Dispose()
    {
        objRef?.Dispose();
    }
}
```

```javascript
// wwwroot/js/dropzone.js
function setupDropZone(elementId, dotNetRef) {
    const el = document.getElementById(elementId);

    el.addEventListener("dragover", (e) => {
        e.preventDefault();
        el.classList.add("drag-over");
    });

    el.addEventListener("dragleave", () => {
        el.classList.remove("drag-over");
    });

    el.addEventListener("drop", async (e) => {
        e.preventDefault();
        el.classList.remove("drag-over");

        const fileNames = [];
        for (const file of e.dataTransfer.files) {
            // Call .NET instance method to validate
            const isValid = await dotNetRef.invokeMethodAsync("ValidateFile", file.name, file.size);
            if (isValid) {
                fileNames.push(file.name);
            }
        }

        if (fileNames.length > 0) {
            // Call .NET instance method with results
            dotNetRef.invokeMethodAsync("OnFilesDropped", fileNames);
        }
    });
}
```

---

## 5. ElementReference for DOM Access

```razor
<input @ref="inputElement" type="text" placeholder="Auto-focused input" />
<button @onclick="FocusInput">Focus Input</button>

<div @ref="scrollContainer" style="height: 300px; overflow-y: auto;">
    @foreach (var item in items)
    {
        <div>@item</div>
    }
</div>
<button @onclick="ScrollToBottom">Scroll to Bottom</button>

@code {
    private ElementReference inputElement;
    private ElementReference scrollContainer;
    private List<string> items = Enumerable.Range(1, 100).Select(i => $"Item {i}").ToList();

    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            await inputElement.FocusAsync(); // Built-in focus method (.NET 8+)
        }
    }

    private async Task FocusInput()
    {
        await inputElement.FocusAsync();
    }

    private async Task ScrollToBottom()
    {
        await JS.InvokeVoidAsync("scrollToBottom", scrollContainer);
    }
}
```

```javascript
function scrollToBottom(element) {
    element.scrollTop = element.scrollHeight;
}
```

---

## 6. Blazor Server vs WebAssembly JS Interop Differences

| Aspect | Blazor Server | Blazor WebAssembly |
|--------|--------------|-------------------|
| JS call mechanism | SignalR (network round-trip) | In-process (direct) |
| Latency | Higher (network dependent) | Negligible |
| Synchronous calls | Not supported | Supported via `IJSInProcessRuntime` |
| Large data transfers | Slow (serialized over SignalR) | Fast (in-memory) |
| JS debugging | Browser DevTools work normally | Browser DevTools work normally |
| `IJSInProcessRuntime` | Not available | Available |
| Error handling | May include network errors | Standard exceptions only |

### Synchronous Calls (WebAssembly Only)

```razor
@inject IJSRuntime JS

@code {
    private string GetLocalStorageSync(string key)
    {
        // Only works in WebAssembly
        var jsInProcess = (IJSInProcessRuntime)JS;
        return jsInProcess.Invoke<string>("localStorage.getItem", key);
    }

    private void SetLocalStorageSync(string key, string value)
    {
        var jsInProcess = (IJSInProcessRuntime)JS;
        jsInProcess.InvokeVoid("localStorage.setItem", key, value);
    }
}
```

---

## 7. Marshalling Data Between JS and .NET

### Supported Types

| .NET Type | JavaScript Type | Notes |
|-----------|----------------|-------|
| `string` | `string` | Direct mapping |
| `int`, `long`, `double` | `number` | Numeric types |
| `bool` | `boolean` | Direct mapping |
| `DateTime` | `string` (ISO 8601) | Serialized as string |
| `byte[]` | `Uint8Array` | Binary data |
| `object`, `record` | `object` | JSON serialized |
| `IJSObjectReference` | JS object | Managed reference |
| `ElementReference` | DOM element | Only in JS interop calls |
| `DotNetObjectReference<T>` | .NET object proxy | For instance callbacks |

### Complex Object Marshalling

```csharp
public record ChartConfig
{
    public string Type { get; init; } = "bar";
    public string Title { get; init; } = "";
    public List<DataSeries> Series { get; init; } = [];
    public AxisConfig XAxis { get; init; } = new();
    public AxisConfig YAxis { get; init; } = new();
}

public record DataSeries
{
    public string Name { get; init; } = "";
    public double[] Values { get; init; } = [];
    public string Color { get; init; } = "#1a73e8";
}

public record AxisConfig
{
    public string Label { get; init; } = "";
    public string[] Categories { get; init; } = [];
}
```

```razor
@code {
    private async Task CreateChart()
    {
        var config = new ChartConfig
        {
            Type = "line",
            Title = "Monthly Sales",
            Series =
            [
                new DataSeries
                {
                    Name = "Revenue",
                    Values = [100, 200, 150, 300, 250],
                    Color = "#4285f4"
                }
            ],
            XAxis = new AxisConfig
            {
                Label = "Month",
                Categories = ["Jan", "Feb", "Mar", "Apr", "May"]
            }
        };

        // Object is automatically JSON-serialized for JS
        await JS.InvokeVoidAsync("chartLib.create", "chart-container", config);
    }
}
```

---

## 8. Error Handling in JS Interop

```razor
@inject IJSRuntime JS
@inject ILogger<MyComponent> Logger

@code {
    private async Task SafeJsCall()
    {
        try
        {
            var result = await JS.InvokeAsync<string>(
                "someJsFunction", "arg1", "arg2");
        }
        catch (JSException ex)
        {
            // JavaScript threw an error
            Logger.LogError(ex, "JavaScript error: {Message}", ex.Message);
        }
        catch (JSDisconnectedException)
        {
            // Blazor Server: SignalR circuit disconnected
            Logger.LogWarning("JS interop failed: circuit disconnected");
        }
        catch (TaskCanceledException)
        {
            // Call was cancelled (e.g., component disposed)
            Logger.LogDebug("JS interop call cancelled");
        }
        catch (InvalidOperationException ex) when (ex.Message.Contains("prerendering"))
        {
            // Attempted JS interop during prerendering
            Logger.LogDebug("JS interop not available during prerender");
        }
    }

    // Timeout for JS calls
    private async Task JsCallWithTimeout()
    {
        using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
        try
        {
            await JS.InvokeVoidAsync("longRunningJsFunction", cts.Token);
        }
        catch (TaskCanceledException)
        {
            Logger.LogWarning("JS call timed out after 5 seconds");
        }
    }
}
```

---

## 9. IAsyncDisposable for JS Module Cleanup

Always clean up JS references to prevent memory leaks.

```razor
@implements IAsyncDisposable

@code {
    private IJSObjectReference? module;
    private IJSObjectReference? mapInstance;
    private DotNetObjectReference<MapComponent>? dotNetRef;

    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            dotNetRef = DotNetObjectReference.Create(this);
            module = await JS.InvokeAsync<IJSObjectReference>("import", "./js/map.js");
            mapInstance = await module.InvokeAsync<IJSObjectReference>(
                "initMap", "map-container", dotNetRef);
        }
    }

    public async ValueTask DisposeAsync()
    {
        try
        {
            if (mapInstance is not null)
            {
                // Clean up JS resources
                await mapInstance.InvokeVoidAsync("destroy");
                await mapInstance.DisposeAsync();
            }

            if (module is not null)
            {
                await module.DisposeAsync();
            }
        }
        catch (JSDisconnectedException)
        {
            // Circuit already disconnected, JS cleanup not possible
        }

        dotNetRef?.Dispose();
    }
}
```

---

## 10. Calling JS on First Render

JS interop is only available after the component has rendered. Never call JS in `OnInitialized`.

```razor
@code {
    // WRONG: JS is not available during initialization
    protected override async Task OnInitializedAsync()
    {
        // This will throw InvalidOperationException during prerendering
        // var value = await JS.InvokeAsync<string>("localStorage.getItem", "key");
    }

    // CORRECT: Use OnAfterRenderAsync
    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            var value = await JS.InvokeAsync<string?>("localStorage.getItem", "key");
            if (value is not null)
            {
                // Update state and re-render
                savedValue = value;
                StateHasChanged();
            }
        }
    }
}
```

---

## 11. JS Initializers

JS initializers run before and after Blazor starts, ideal for global setup.

```javascript
// wwwroot/{ASSEMBLY_NAME}.lib.module.js
// File name must match the assembly name

export function beforeStart(options, extensions) {
    // Runs before Blazor starts
    console.log("Blazor is about to start");

    // Register custom event types
    // Configure global error handlers
    // Initialize third-party libraries
}

export function afterStarted(blazor) {
    // Runs after Blazor has fully started
    console.log("Blazor has started");

    // Register custom events
    blazor.registerCustomEventType("custompaste", {
        browserEventName: "paste",
        createEventArgs: (event) => {
            return {
                text: event.clipboardData.getData("text"),
            };
        },
    });

    // Set up global navigation events
    blazor.addEventListener("enhancedload", () => {
        console.log("Enhanced navigation completed");
    });
}

export function beforeWebAssemblyStart(options, extensions) {
    // WebAssembly-specific initialization
    console.log("WebAssembly runtime loading");
}

export function afterWebAssemblyStarted(blazor) {
    // WebAssembly-specific post-start
    console.log("WebAssembly runtime ready");
}

export function beforeServerStart(options, extensions) {
    // Server-specific initialization
}

export function afterServerStarted(blazor) {
    // Server circuit established
}
```

---

## 12. Handling Prerendering

During prerendering, JavaScript is not available. Guard JS interop calls accordingly.

### Check for Prerendering

```razor
@inject IJSRuntime JS

@code {
    private bool isPrerendering = true;

    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            isPrerendering = false;
            // Now safe to call JS
            await InitializeJsAsync();
        }
    }

    private async Task InitializeJsAsync()
    {
        if (isPrerendering) return;
        await JS.InvokeVoidAsync("initializeComponent");
    }
}
```

### Using OperatingSystem.IsBrowser() (WebAssembly)

```csharp
// Only in WebAssembly context
if (OperatingSystem.IsBrowser())
{
    // Running in the browser (WebAssembly)
    var jsInProcess = (IJSInProcessRuntime)JS;
    jsInProcess.InvokeVoid("console.log", "Running in WASM");
}
```

---

## 13. Import Maps

.NET 8+ supports import maps for module resolution.

```html
<!-- In App.razor or index.html -->
<script type="importmap">
{
    "imports": {
        "lodash": "https://cdn.jsdelivr.net/npm/lodash-es@4.17.21/lodash.min.js",
        "chart.js": "https://cdn.jsdelivr.net/npm/chart.js@4/dist/chart.umd.min.js"
    }
}
</script>
```

```javascript
// In your .razor.js or module file
import _ from "lodash";
import { Chart } from "chart.js";

export function processData(data) {
    return _.groupBy(data, "category");
}
```

---

## 14. Third-Party JS Library Integration

### Full Integration Pattern

```javascript
// wwwroot/js/monaco-interop.js
let editors = new Map();

export function createEditor(elementId, dotNetRef, options) {
    const container = document.getElementById(elementId);

    const editor = monaco.editor.create(container, {
        value: options.value || "",
        language: options.language || "javascript",
        theme: options.theme || "vs-dark",
        automaticLayout: true,
        minimap: { enabled: options.showMinimap ?? true },
    });

    // Notify .NET on content change
    editor.onDidChangeModelContent(() => {
        const value = editor.getValue();
        dotNetRef.invokeMethodAsync("OnContentChanged", value);
    });

    editors.set(elementId, editor);
    return editor;
}

export function getValue(elementId) {
    return editors.get(elementId)?.getValue() ?? "";
}

export function setValue(elementId, value) {
    editors.get(elementId)?.setValue(value);
}

export function setLanguage(elementId, language) {
    const editor = editors.get(elementId);
    if (editor) {
        monaco.editor.setModelLanguage(editor.getModel(), language);
    }
}

export function dispose(elementId) {
    const editor = editors.get(elementId);
    if (editor) {
        editor.dispose();
        editors.delete(elementId);
    }
}
```

```razor
@* CodeEditor.razor *@
@inject IJSRuntime JS
@implements IAsyncDisposable

<div id="@elementId" style="height: @Height; width: 100%;"></div>

@code {
    private IJSObjectReference? module;
    private DotNetObjectReference<CodeEditor>? dotNetRef;
    private readonly string elementId = $"editor-{Guid.NewGuid():N}";

    [Parameter] public string Value { get; set; } = "";
    [Parameter] public EventCallback<string> ValueChanged { get; set; }
    [Parameter] public string Language { get; set; } = "csharp";
    [Parameter] public string Theme { get; set; } = "vs-dark";
    [Parameter] public string Height { get; set; } = "400px";
    [Parameter] public bool ShowMinimap { get; set; } = true;

    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            dotNetRef = DotNetObjectReference.Create(this);
            module = await JS.InvokeAsync<IJSObjectReference>("import", "./js/monaco-interop.js");
            await module.InvokeVoidAsync("createEditor", elementId, dotNetRef, new
            {
                value = Value,
                language = Language,
                theme = Theme,
                showMinimap = ShowMinimap
            });
        }
    }

    [JSInvokable]
    public async Task OnContentChanged(string newValue)
    {
        Value = newValue;
        await ValueChanged.InvokeAsync(newValue);
    }

    public async Task SetLanguageAsync(string language)
    {
        if (module is not null)
        {
            await module.InvokeVoidAsync("setLanguage", elementId, language);
        }
    }

    public async ValueTask DisposeAsync()
    {
        try
        {
            if (module is not null)
            {
                await module.InvokeVoidAsync("dispose", elementId);
                await module.DisposeAsync();
            }
        }
        catch (JSDisconnectedException) { }

        dotNetRef?.Dispose();
    }
}
```

### Usage

```razor
<CodeEditor @bind-Value="sourceCode" Language="csharp" Height="500px" />

@code {
    private string sourceCode = "Console.WriteLine(\"Hello, World!\");";
}
```

---

## Best Practices

1. **Use JavaScript isolation (modules)** -- Avoid polluting the global `window` object.
2. **Use collocated `.razor.js` files** -- Keeps JS close to the component that uses it.
3. **Always dispose `IJSObjectReference`** -- Prevents memory leaks in both JS and .NET.
4. **Always dispose `DotNetObjectReference`** -- Prevents .NET objects from being retained.
5. **Guard JS calls during prerendering** -- Use `OnAfterRenderAsync` for JS interop, never `OnInitialized`.
6. **Handle `JSDisconnectedException`** -- In Blazor Server, the circuit may disconnect during disposal.
7. **Minimize JS interop calls** -- Batch operations; each call has overhead (especially in Server mode).
8. **Use `InvokeVoidAsync` when no return value is needed** -- Slightly more efficient.
9. **Pass minimal data to JS** -- Serialize only what is needed; avoid sending large objects.
10. **Use `IJSInProcessRuntime` in WASM for sync calls** -- Avoids unnecessary async overhead.

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|---------|
| Calling JS in `OnInitialized` | `InvalidOperationException` during prerender | Move to `OnAfterRenderAsync(firstRender: true)` |
| Not disposing `IJSObjectReference` | JS memory leak | Implement `IAsyncDisposable` and dispose all references |
| Not disposing `DotNetObjectReference` | .NET memory leak | Dispose in component's `Dispose`/`DisposeAsync` |
| Using `IJSInProcessRuntime` in Server mode | `InvalidCastException` | Only use `IJSInProcessRuntime` in WebAssembly |
| Missing assembly name in `DotNet.invokeMethodAsync` | Method not found | First arg must be the .NET assembly name |
| `[JSInvokable]` method not public | Method not callable from JS | Method must be `public` |
| Large data over SignalR (Server) | Timeout or disconnection | Stream data in chunks or use HTTP endpoint |
| Not catching `JSDisconnectedException` in dispose | Unhandled exception on circuit disconnect | Wrap disposal JS calls in try/catch |
| Forgetting `StateHasChanged` after `[JSInvokable]` callback | UI does not update | Call `StateHasChanged()` or use `InvokeAsync` |
| JS module path wrong for collocated files | Module not found | Path is relative to wwwroot: `./_content/{Assembly}/Component.razor.js` |
