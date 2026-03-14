# Blazor Render Modes

> Official Documentation: https://learn.microsoft.com/aspnet/core/blazor/components/render-modes

.NET 8 introduced a unified Blazor model where different rendering strategies coexist in a single application. Render modes determine how a component is rendered and whether it is interactive. Understanding render modes is essential for building performant, scalable Blazor applications.

---

## 1. Overview of Render Modes

| Render Mode | Attribute | Interactive | Hosting | First Paint | Payload |
|-------------|-----------|-------------|---------|-------------|---------|
| Static SSR | (none / default) | No | Server | Fast | HTML only |
| Interactive Server | `InteractiveServer` | Yes | Server (SignalR) | Fast | HTML + SignalR circuit |
| Interactive WebAssembly | `InteractiveWebAssembly` | Yes | Client (WASM) | Slower (download) | .NET runtime + DLLs |
| Interactive Auto | `InteractiveAuto` | Yes | Server then WASM | Fast, then client | Both |

### Rendering Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Blazor Web App (.NET 8+)                  │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  Static SSR   │  │ Interactive  │  │   Interactive     │  │
│  │  Components   │  │   Server     │  │   WebAssembly    │  │
│  │              │  │  Components   │  │   Components     │  │
│  │  • No interop│  │  • SignalR    │  │   • In-browser   │  │
│  │  • No events │  │  • Full .NET  │  │   • Full .NET    │  │
│  │  • Fast      │  │  • Low latency│  │   • Offline      │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │           Interactive Auto Components               │    │
│  │  • Server on first visit (fast start)               │    │
│  │  • WebAssembly on subsequent visits (cached runtime)│    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Static Server-Side Rendering (Static SSR)

Static SSR renders components on the server and sends plain HTML. No interactivity is added -- no event handlers, no SignalR circuit.

```razor
@* This is a static SSR component by default (no @rendermode) *@
@page "/products"

<h1>Products</h1>

<ul>
    @foreach (var product in products)
    {
        <li>
            <a href="/products/@product.Id">@product.Name - @product.Price.ToString("C")</a>
        </li>
    }
</ul>

@code {
    private List<Product> products = [];

    [Inject] private IProductService ProductService { get; set; } = default!;

    protected override async Task OnInitializedAsync()
    {
        products = await ProductService.GetAllAsync();
    }
}
```

**Use Static SSR when:**
- Page is read-only (product listings, blog posts, documentation)
- SEO is critical
- Fastest possible initial load is needed
- No client-side interactivity required

---

## 3. Interactive Server

Interactivity is handled via a SignalR connection. UI events are sent to the server, which processes them and sends back DOM diffs.

```razor
@page "/counter"
@rendermode InteractiveServer

<h1>Interactive Counter (Server)</h1>
<p>Count: @count</p>
<button @onclick="Increment">Increment</button>

@code {
    private int count = 0;

    private void Increment() => count++;
}
```

### Setting Render Mode on a Component Instance

```razor
@* Parent page *@
<Counter @rendermode="InteractiveServer" />
```

**Characteristics:**
- Full .NET runtime on server (access to databases, file system, etc.)
- Low initial load (no WASM download)
- Requires constant server connection (SignalR)
- UI latency depends on network round-trip
- Server memory scales with connected users

---

## 4. Interactive WebAssembly

The component runs entirely in the browser using the .NET WebAssembly runtime.

```razor
@page "/calculator"
@rendermode InteractiveWebAssembly

<h1>Calculator (WebAssembly)</h1>
<input @bind="expression" placeholder="Enter expression" />
<button @onclick="Calculate">Calculate</button>
<p>Result: @result</p>

@code {
    private string expression = "";
    private string result = "";

    private void Calculate()
    {
        // Runs entirely in the browser
        try
        {
            var table = new System.Data.DataTable();
            var computed = table.Compute(expression, null);
            result = computed?.ToString() ?? "Error";
        }
        catch
        {
            result = "Invalid expression";
        }
    }
}
```

**Characteristics:**
- Runs in browser (no server connection needed after download)
- First load downloads .NET runtime (~2-3 MB cached)
- No server memory per user
- Cannot directly access server resources (needs HTTP APIs)
- Supports offline scenarios

---

## 5. Interactive Auto

Interactive Auto uses Interactive Server on the first visit (for fast startup), then switches to WebAssembly on subsequent visits once the runtime is cached.

```razor
@page "/dashboard"
@rendermode InteractiveAuto

<h1>Dashboard (Auto)</h1>
<DashboardWidgets />

@code {
    // First visit: runs on server via SignalR
    // Subsequent visits: runs in browser via WebAssembly
}
```

### How Auto Mode Works

```
First Visit:
  Browser ──request──▶ Server
  Server ──HTML + SignalR──▶ Browser (Interactive Server mode)
  Background: WASM runtime downloads and caches

Subsequent Visits:
  Browser ──request──▶ Server
  Server ──HTML──▶ Browser
  Browser uses cached WASM runtime (Interactive WebAssembly mode)
```

**Characteristics:**
- Best of both worlds: fast first paint + client-side on return
- Components must work in both Server and WebAssembly contexts
- Shared service registration required (see below)
- Most complex to reason about

---

## 6. Setting Render Modes

### Per-Component (Attribute Directive)

```razor
@* In the component file *@
@page "/my-page"
@rendermode InteractiveServer
```

### Per-Component Instance (Parent Sets It)

```razor
@* In the parent *@
<MyComponent @rendermode="InteractiveServer" />
<AnotherComponent @rendermode="InteractiveWebAssembly" />
<ThirdComponent @rendermode="InteractiveAuto" />
```

### Global Render Mode (App.razor)

```razor
@* App.razor - Set default for all pages *@
<!DOCTYPE html>
<html>
<head>
    <HeadOutlet @rendermode="InteractiveServer" />
</head>
<body>
    <Routes @rendermode="InteractiveServer" />
    <script src="_framework/blazor.web.js"></script>
</body>
</html>
```

### Custom Render Mode Instances

```razor
@* With prerendering disabled *@
<MyComponent @rendermode="new InteractiveServerRenderMode(prerender: false)" />
<MyComponent @rendermode="new InteractiveWebAssemblyRenderMode(prerender: false)" />
<MyComponent @rendermode="new InteractiveAutoRenderMode(prerender: false)" />
```

---

## 7. Render Mode Propagation

Render modes propagate from parent to child. A child cannot have a "higher" interactivity mode than its parent.

### Valid Propagation

```
Static SSR parent ──▶ Static SSR child            (OK)
Static SSR parent ──▶ InteractiveServer child      (OK - creates boundary)
InteractiveServer parent ──▶ InteractiveServer child (OK - inherits)
InteractiveServer parent ──▶ (no rendermode) child   (OK - inherits Server)
```

### Invalid Propagation

```
InteractiveServer parent ──▶ InteractiveWebAssembly child   (ERROR)
InteractiveWebAssembly parent ──▶ InteractiveServer child   (ERROR)
```

```razor
@* This will throw a runtime error *@
@rendermode InteractiveServer

<div>
    @* Cannot nest a different interactive mode *@
    <WasmComponent @rendermode="InteractiveWebAssembly" />  @* ERROR *@
</div>
```

**Rule:** Once a render mode boundary is established, all descendants inherit that mode. You cannot mix Server and WebAssembly within the same interactive subtree.

---

## 8. Streaming Rendering

Streaming rendering sends the initial HTML immediately, then streams updated content as async operations complete.

```razor
@page "/weather"
@attribute [StreamRendering]

<h1>Weather Forecast</h1>

@if (forecasts is null)
{
    <p>Loading forecasts...</p>
}
else
{
    <table class="table">
        <thead>
            <tr>
                <th>Date</th>
                <th>Temp (C)</th>
                <th>Summary</th>
            </tr>
        </thead>
        <tbody>
            @foreach (var forecast in forecasts)
            {
                <tr>
                    <td>@forecast.Date.ToShortDateString()</td>
                    <td>@forecast.TemperatureC</td>
                    <td>@forecast.Summary</td>
                </tr>
            }
        </tbody>
    </table>
}

@code {
    private WeatherForecast[]? forecasts;

    protected override async Task OnInitializedAsync()
    {
        // Initial render shows "Loading..." immediately
        // After data loads, the table streams to the browser
        await Task.Delay(2000); // Simulating slow API call
        forecasts = await WeatherService.GetForecastsAsync();
    }
}
```

**How streaming rendering works:**
1. Server renders initial HTML (with `null` data showing loading placeholder)
2. HTML is sent immediately to the browser
3. Server continues executing `OnInitializedAsync`
4. When async work completes, server streams the updated HTML
5. Browser patches the DOM with the new content

**Streaming rendering works with Static SSR and Interactive Server prerendering.**

---

## 9. Enhanced Navigation and Form Handling

.NET 8 Blazor enhances traditional navigation and form posts to feel like an SPA.

### Enhanced Navigation

```razor
@* Enhanced navigation is automatic for internal links *@
<nav>
    <a href="/products">Products</a>      @* Enhanced: fetches and patches DOM *@
    <a href="/about">About</a>             @* Enhanced: no full page reload *@
    <a href="https://external.com" data-enhance-nav="false">External</a>  @* Opt out *@
</nav>
```

### Enhanced Form Handling

```razor
@page "/contact"
@attribute [StreamRendering]

<EditForm Model="model" OnValidSubmit="HandleSubmit" FormName="ContactForm" Enhance>
    <DataAnnotationsValidator />

    <div class="mb-3">
        <label>Name</label>
        <InputText @bind-Value="model.Name" class="form-control" />
        <ValidationMessage For="() => model.Name" />
    </div>

    <div class="mb-3">
        <label>Email</label>
        <InputText @bind-Value="model.Email" class="form-control" />
        <ValidationMessage For="() => model.Email" />
    </div>

    <div class="mb-3">
        <label>Message</label>
        <InputTextArea @bind-Value="model.Message" class="form-control" />
        <ValidationMessage For="() => model.Message" />
    </div>

    <button type="submit" class="btn btn-primary" disabled="@isSubmitting">
        @(isSubmitting ? "Sending..." : "Send")
    </button>

    @if (submitted)
    {
        <div class="alert alert-success mt-3">Message sent successfully!</div>
    }
</EditForm>

@code {
    [SupplyParameterFromForm]
    private ContactModel model { get; set; } = new();

    private bool isSubmitting = false;
    private bool submitted = false;

    private async Task HandleSubmit()
    {
        isSubmitting = true;
        await ContactService.SendAsync(model);
        submitted = true;
        isSubmitting = false;
        model = new ContactModel();
    }

    public class ContactModel
    {
        [Required, StringLength(100)]
        public string Name { get; set; } = "";

        [Required, EmailAddress]
        public string Email { get; set; } = "";

        [Required, StringLength(1000)]
        public string Message { get; set; } = "";
    }
}
```

The `Enhance` attribute on the form makes it submit via `fetch` instead of a full-page POST, and the server streams back the updated HTML.

---

## 10. Prerendering Behavior

By default, interactive components are prerendered on the server, then the interactive runtime takes over.

### Prerendering Timeline

```
1. Server renders component to HTML (prerender)
2. HTML sent to browser (user sees content immediately)
3. Interactive runtime loads (SignalR connects or WASM downloads)
4. Runtime "hydrates" the component (attaches event handlers)
5. Component is fully interactive
```

### Disabling Prerendering

```razor
@* Disable prerendering for specific component *@
<LiveChat @rendermode="new InteractiveServerRenderMode(prerender: false)" />

@* The component will show nothing until SignalR connects *@
```

### Why Disable Prerendering?

| Scenario | Reason |
|----------|--------|
| Component relies heavily on JS interop | Avoids "flash" when JS initializes |
| Component shows user-specific data | Prevents showing stale/wrong data during prerender |
| Component uses browser APIs | APIs unavailable during server prerender |
| Authentication-dependent UI | Avoids showing unauthenticated state flash |

---

## 11. Detecting Render Mode

### Using RendererInfo (.NET 8+)

```razor
@page "/mode-info"
@rendermode InteractiveAuto

<h1>Render Mode Info</h1>

<p>Renderer Name: @RendererInfo.Name</p>
<p>Is Interactive: @RendererInfo.IsInteractive</p>

@if (RendererInfo.Name == "Static")
{
    <p>Rendering statically on the server (prerender or SSR).</p>
}
else if (RendererInfo.Name == "InteractiveServer")
{
    <p>Running interactively on the server via SignalR.</p>
}
else if (RendererInfo.Name == "InteractiveWebAssembly")
{
    <p>Running interactively in the browser via WebAssembly.</p>
}
```

### Using OperatingSystem.IsBrowser()

```csharp
// Returns true when running in WebAssembly
if (OperatingSystem.IsBrowser())
{
    // WebAssembly-specific code
    Console.WriteLine("Running in the browser");
}
else
{
    // Server-side code
    Console.WriteLine("Running on the server");
}
```

### Conditional Service Registration

```csharp
// In the main server project (Program.cs)
builder.Services.AddScoped<IWeatherService, ServerWeatherService>();

// In the WebAssembly client project (Program.cs)
builder.Services.AddScoped<IWeatherService, HttpWeatherService>();
```

---

## 12. PersistentComponentState

Persist state across prerendering to avoid duplicate data fetching.

```razor
@page "/products/{Id:int}"
@implements IDisposable
@inject PersistentComponentState ApplicationState

<h1>@product?.Name</h1>
<p>@product?.Description</p>
<p>Price: @product?.Price.ToString("C")</p>

@code {
    [Parameter] public int Id { get; set; }

    private Product? product;
    private PersistingComponentStateSubscription persistingSubscription;

    protected override async Task OnInitializedAsync()
    {
        persistingSubscription = ApplicationState.RegisterOnPersisting(PersistData);

        // Try to restore state from prerender
        if (!ApplicationState.TryTakeFromJson<Product>($"product-{Id}", out var restored))
        {
            // State not available (first render or not prerendered)
            product = await ProductService.GetByIdAsync(Id);
        }
        else
        {
            product = restored;
        }
    }

    private Task PersistData()
    {
        // Persist state so interactive runtime can pick it up
        ApplicationState.PersistAsJson($"product-{Id}", product);
        return Task.CompletedTask;
    }

    public void Dispose()
    {
        persistingSubscription.Dispose();
    }
}
```

**Without `PersistentComponentState`:**
1. Server prerenders the component (fetches data from DB)
2. Client hydrates the component (fetches data again from API)
3. User sees a brief flash as data reloads

**With `PersistentComponentState`:**
1. Server prerenders the component (fetches data from DB)
2. State is embedded in the HTML response
3. Client hydrates and uses the persisted state (no second fetch)

---

## 13. Render Mode Decision Guide

| Consideration | Static SSR | Interactive Server | Interactive WASM | Interactive Auto |
|---------------|-----------|-------------------|-----------------|-----------------|
| **Interactivity needed** | No | Yes | Yes | Yes |
| **SEO important** | Best | Good (with prerender) | Good (with prerender) | Good (with prerender) |
| **Offline support** | No | No | Yes | Partial |
| **Initial load speed** | Fastest | Fast | Slow (first visit) | Fast |
| **Server memory per user** | None | High (circuit) | None | Varies |
| **Network dependency** | Page load only | Constant (SignalR) | API calls only | Varies |
| **Access to server resources** | Direct | Direct | Via API | Via API (WASM) |
| **Maximum scalability** | Highest | Limited by circuits | Highest | High |
| **JS interop** | Not available | Available | Available | Available |
| **Real-time updates** | No | Yes (SignalR) | Via own connection | Via own connection |

### Decision Flowchart

```
Does the page need interactivity?
├── No ──▶ Static SSR
└── Yes
    ├── Does it need direct server resource access (DB, file system)?
    │   ├── Yes ──▶ Interactive Server
    │   └── No
    │       ├── Does it need offline support?
    │       │   ├── Yes ──▶ Interactive WebAssembly
    │       │   └── No
    │       │       ├── Is first-visit load time critical?
    │       │       │   ├── Yes ──▶ Interactive Auto
    │       │       │   └── No ──▶ Interactive WebAssembly
    │       │       └──
    │       └──
    └──
```

---

## 14. Circuit Management (Interactive Server)

### Circuit Configuration

```csharp
// Program.cs
builder.Services.AddRazorComponents()
    .AddInteractiveServerComponents(options =>
    {
        options.DisconnectedCircuitRetentionPeriod = TimeSpan.FromMinutes(3);
        options.DisconnectedCircuitMaxRetained = 100;
        options.JSInteropDefaultCallTimeout = TimeSpan.FromSeconds(60);
        options.MaxBufferedUnacknowledgedRenderBatches = 10;
    });
```

### Circuit Configuration Reference

| Option | Default | Description |
|--------|---------|-------------|
| `DisconnectedCircuitRetentionPeriod` | 3 minutes | How long to keep disconnected circuits |
| `DisconnectedCircuitMaxRetained` | 100 | Max disconnected circuits in memory |
| `JSInteropDefaultCallTimeout` | 1 minute | Timeout for JS interop calls |
| `MaxBufferedUnacknowledgedRenderBatches` | 10 | Render batches before disconnecting slow clients |
| `DetailedErrors` | false | Send detailed errors to client |

### Circuit Lifecycle Events

```razor
@implements IDisposable

@code {
    [Inject] private CircuitHandler CircuitHandler { get; set; } = default!;

    // Custom CircuitHandler for global circuit events
}
```

```csharp
public class TrackingCircuitHandler : CircuitHandler
{
    private readonly ILogger<TrackingCircuitHandler> _logger;

    public TrackingCircuitHandler(ILogger<TrackingCircuitHandler> logger)
    {
        _logger = logger;
    }

    public override Task OnCircuitOpenedAsync(Circuit circuit, CancellationToken ct)
    {
        _logger.LogInformation("Circuit opened: {CircuitId}", circuit.Id);
        return Task.CompletedTask;
    }

    public override Task OnConnectionUpAsync(Circuit circuit, CancellationToken ct)
    {
        _logger.LogInformation("Connection up: {CircuitId}", circuit.Id);
        return Task.CompletedTask;
    }

    public override Task OnConnectionDownAsync(Circuit circuit, CancellationToken ct)
    {
        _logger.LogInformation("Connection down: {CircuitId}", circuit.Id);
        return Task.CompletedTask;
    }

    public override Task OnCircuitClosedAsync(Circuit circuit, CancellationToken ct)
    {
        _logger.LogInformation("Circuit closed: {CircuitId}", circuit.Id);
        return Task.CompletedTask;
    }
}

// Registration
builder.Services.AddScoped<CircuitHandler, TrackingCircuitHandler>();
```

---

## 15. WebAssembly Download Size Considerations

### Reducing Payload Size

```xml
<!-- In the .Client project .csproj -->
<PropertyGroup>
    <!-- Enable trimming to reduce DLL size -->
    <PublishTrimmed>true</PublishTrimmed>
    <TrimMode>link</TrimMode>

    <!-- Enable AOT compilation for performance (increases download but faster execution) -->
    <RunAOTCompilation>true</RunAOTCompilation>

    <!-- Enable Brotli compression -->
    <BlazorEnableCompression>true</BlazorEnableCompression>
</PropertyGroup>
```

### Lazy Loading Assemblies

```razor
@page "/admin"
@using Microsoft.AspNetCore.Components.WebAssembly.Services
@inject LazyAssemblyLoader AssemblyLoader

@if (!loaded)
{
    <p>Loading admin module...</p>
}
else
{
    <AdminDashboard />
}

@code {
    private bool loaded = false;

    protected override async Task OnInitializedAsync()
    {
        var assemblies = await AssemblyLoader.LoadAssembliesAsync(new[]
        {
            "AdminModule.dll"
        });
        loaded = true;
    }
}
```

### Typical Download Sizes

| Component | Compressed Size |
|-----------|----------------|
| .NET WASM runtime | ~2 MB |
| Base class libraries | ~1-3 MB |
| App-specific DLLs | ~100 KB - 1 MB |
| **Total first load** | **~3-6 MB** |
| **Subsequent loads (cached)** | **~0 KB** |

---

## 16. Limitations Per Render Mode

| Feature | Static SSR | Interactive Server | Interactive WASM |
|---------|-----------|-------------------|-----------------|
| `@onclick` and event handlers | No | Yes | Yes |
| `@bind` two-way binding | No | Yes | Yes |
| JS interop | No | Yes | Yes |
| `IJSInProcessRuntime` | No | No | Yes |
| Direct DB access | Yes | Yes | No |
| `HttpContext` access | Yes | No* | No |
| Streaming rendering | Yes | Prerender only | Prerender only |
| Enhanced forms | Yes | N/A (interactive) | N/A (interactive) |
| `NavigationManager` | Limited | Full | Full |
| Component state persistence | No (stateless) | Yes (circuit) | Yes (browser) |

*`HttpContext` is available during prerendering of Interactive Server components, but not after the circuit is established.

---

## 17. Complete App.razor Example

```razor
@* App.razor - .NET 8 Blazor Web App *@
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <base href="/" />
    <link rel="stylesheet" href="app.css" />
    <link rel="stylesheet" href="MyApp.styles.css" />
    <HeadOutlet @rendermode="InteractiveAuto" />
</head>
<body>
    <Routes @rendermode="InteractiveAuto" />
    <script src="_framework/blazor.web.js"></script>
</body>
</html>
```

### Program.cs with All Render Modes

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add Razor components with both interactive renderers
builder.Services.AddRazorComponents()
    .AddInteractiveServerComponents()
    .AddInteractiveWebAssemblyComponents();

// Register services available in both Server and WASM contexts
builder.Services.AddScoped<IWeatherService, ServerWeatherService>();

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseAntiforgery();

// Map Razor components with both interactive render modes
app.MapRazorComponents<App>()
    .AddInteractiveServerRenderMode()
    .AddInteractiveWebAssemblyRenderMode()
    .AddAdditionalAssemblies(typeof(MyApp.Client._Imports).Assembly);

app.Run();
```

---

## Best Practices

1. **Default to Static SSR** -- Only add interactivity where needed; most pages are read-heavy.
2. **Use Interactive Auto for broad interactivity** -- Gives fast first paint with eventual client-side execution.
3. **Use `PersistentComponentState`** -- Prevents double data fetching during prerender-to-interactive handoff.
4. **Use `[StreamRendering]` for slow data pages** -- Shows loading UI immediately while data loads.
5. **Keep interactive subtrees small** -- Apply render modes to specific components, not entire pages when possible.
6. **Use enhanced forms for simple submissions** -- Avoids needing interactive mode for basic form posts.
7. **Monitor circuit count in production (Server mode)** -- Each active user holds a circuit in memory.
8. **Enable trimming and compression for WASM** -- Reduces download size significantly.
9. **Use `RendererInfo` to adapt component behavior** -- Different logic for different render contexts.
10. **Do not access `HttpContext` in interactive components** -- It is only available during prerendering.

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|---------|
| Mixing interactive modes in same subtree | Runtime error | Keep Server and WASM components in separate subtrees |
| Accessing `HttpContext` in interactive mode | `null` or stale data | Use `HttpContext` only in static SSR or during prerender; pass data via parameters |
| Not persisting state across prerender | Double API call, UI flash | Use `PersistentComponentState` to transfer state |
| Interactive component in Static SSR parent without `@rendermode` | Component renders as static | Explicitly set `@rendermode` on the component instance |
| Using `IJSInProcessRuntime` in Auto mode | `InvalidCastException` on Server | Check `OperatingSystem.IsBrowser()` before casting |
| Large WASM payload without lazy loading | Slow first visit | Split into lazy-loaded assemblies; enable trimming |
| Using services not registered in both Server and Client DI | `InvalidOperationException` | Register shared interfaces in both `Program.cs` files |
| Missing `AddInteractiveWebAssemblyRenderMode()` in `MapRazorComponents` | WASM components do not render | Add both render mode mappers in `Program.cs` |
| Relying on circuit state being permanent (Server) | Data loss on reconnection failure | Persist critical state to database or browser storage |
| Not calling `app.UseAntiforgery()` | Enhanced forms fail with 400 | Add antiforgery middleware before mapping components |
