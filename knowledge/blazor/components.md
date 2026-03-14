# Blazor Components

> Official Documentation: https://learn.microsoft.com/aspnet/core/blazor/components/

Components are the fundamental building blocks of Blazor applications. Each component is a self-contained unit of UI defined in a `.razor` file that combines C# logic with Razor markup.

---

## 1. Razor Component Basics

A Blazor component is a `.razor` file that combines HTML markup with C# code.

```razor
@* Counter.razor *@
<h3>Counter</h3>

<p>Current count: @currentCount</p>

<button class="btn btn-primary" @onclick="IncrementCount">Click me</button>

@code {
    private int currentCount = 0;

    private void IncrementCount()
    {
        currentCount++;
    }
}
```

### Code-Behind Pattern

Separate markup from logic using partial classes.

```csharp
// Counter.razor.cs
public partial class Counter
{
    private int currentCount = 0;

    private void IncrementCount()
    {
        currentCount++;
    }
}
```

```razor
@* Counter.razor *@
<h3>Counter</h3>
<p>Current count: @currentCount</p>
<button class="btn btn-primary" @onclick="IncrementCount">Click me</button>
```

### Component Namespaces and Routing

```razor
@page "/counter"
@page "/counter/{InitialCount:int}"
@namespace MyApp.Pages
@using MyApp.Models
@inject ILogger<Counter> Logger
@inject NavigationManager Navigation

<h3>Counter starting at @InitialCount</h3>

@code {
    [Parameter]
    public int InitialCount { get; set; } = 0;
}
```

---

## 2. Component Parameters

Parameters allow parent components to pass data to child components.

### Basic Parameters

```razor
@* Alert.razor *@
<div class="alert alert-@Type" role="alert">
    <strong>@Title</strong> @Message
</div>

@code {
    [Parameter]
    public string Title { get; set; } = "";

    [Parameter]
    public string Message { get; set; } = "";

    [Parameter]
    public string Type { get; set; } = "info"; // info, warning, danger, success
}
```

### Usage

```razor
<Alert Title="Success!" Message="The operation completed." Type="success" />
<Alert Title="Warning!" Message="Check your input." Type="warning" />
```

### Required Parameters (.NET 8+)

```razor
@code {
    [Parameter, EditorRequired]
    public string ProductId { get; set; } = default!;

    [Parameter, EditorRequired]
    public EventCallback<string> OnSelected { get; set; }

    [Parameter]
    public string? OptionalLabel { get; set; }
}
```

### Arbitrary Attributes (Splatting)

```razor
@* CustomInput.razor *@
<input class="form-control @CssClass" @attributes="AdditionalAttributes" />

@code {
    [Parameter]
    public string CssClass { get; set; } = "";

    [Parameter(CaptureUnmatchedValues = true)]
    public Dictionary<string, object>? AdditionalAttributes { get; set; }
}
```

```razor
@* Usage: extra attributes pass through *@
<CustomInput CssClass="large" placeholder="Enter name" aria-label="Name" maxlength="50" />
```

### Parameter Change Detection

```razor
@code {
    private string? _previousProductId;

    [Parameter]
    public string ProductId { get; set; } = "";

    protected override void OnParametersSet()
    {
        if (ProductId != _previousProductId)
        {
            _previousProductId = ProductId;
            // React to parameter change
            LoadProduct(ProductId);
        }
    }

    private void LoadProduct(string id) { /* ... */ }
}
```

---

## 3. EventCallback and EventCallback\<T\>

EventCallbacks enable child-to-parent communication.

### Basic EventCallback

```razor
@* ConfirmDialog.razor *@
<div class="modal">
    <p>@Message</p>
    <button @onclick="OnConfirm">Confirm</button>
    <button @onclick="OnCancel">Cancel</button>
</div>

@code {
    [Parameter]
    public string Message { get; set; } = "Are you sure?";

    [Parameter]
    public EventCallback OnConfirm { get; set; }

    [Parameter]
    public EventCallback OnCancel { get; set; }
}
```

### Typed EventCallback

```razor
@* SearchBox.razor *@
<div class="search-box">
    <input @bind="searchText" @bind:event="oninput"
           @onkeyup="HandleKeyUp"
           placeholder="Search..." />
    <button @onclick="DoSearch">Search</button>
</div>

@code {
    private string searchText = "";

    [Parameter]
    public EventCallback<string> OnSearch { get; set; }

    private async Task DoSearch()
    {
        if (!string.IsNullOrWhiteSpace(searchText))
        {
            await OnSearch.InvokeAsync(searchText);
        }
    }

    private async Task HandleKeyUp(KeyboardEventArgs e)
    {
        if (e.Key == "Enter")
        {
            await DoSearch();
        }
    }
}
```

### Parent Consuming EventCallback

```razor
@* ParentPage.razor *@
@page "/products"

<SearchBox OnSearch="HandleSearch" />

@if (isLoading)
{
    <p>Searching...</p>
}
else
{
    @foreach (var product in filteredProducts)
    {
        <ProductCard Product="product" OnSelected="HandleProductSelected" />
    }
}

@code {
    private List<Product> filteredProducts = [];
    private bool isLoading = false;

    private async Task HandleSearch(string query)
    {
        isLoading = true;
        filteredProducts = await ProductService.SearchAsync(query);
        isLoading = false;
    }

    private void HandleProductSelected(Product product)
    {
        Navigation.NavigateTo($"/products/{product.Id}");
    }
}
```

---

## 4. Cascading Values and Parameters

Cascading values pass data down the component tree without explicit parameter passing.

### Providing a Cascading Value

```razor
@* App.razor or a layout *@
<CascadingValue Value="theme">
    <Router AppAssembly="typeof(App).Assembly">
        <Found Context="routeData">
            <RouteView RouteData="routeData" DefaultLayout="typeof(MainLayout)" />
        </Found>
    </Router>
</CascadingValue>

@code {
    private ThemeInfo theme = new()
    {
        PrimaryColor = "#1a73e8",
        SecondaryColor = "#4285f4",
        FontFamily = "Inter, sans-serif",
        IsDarkMode = false
    };
}
```

### Consuming a Cascading Parameter

```razor
@* ThemedButton.razor *@
<button style="background-color: @Theme.PrimaryColor; font-family: @Theme.FontFamily"
        @onclick="OnClick">
    @ChildContent
</button>

@code {
    [CascadingParameter]
    public ThemeInfo Theme { get; set; } = default!;

    [Parameter]
    public EventCallback OnClick { get; set; }

    [Parameter]
    public RenderFragment? ChildContent { get; set; }
}
```

### Named Cascading Values

```razor
@* Provider *@
<CascadingValue Value="currentUser" Name="CurrentUser">
    <CascadingValue Value="appSettings" Name="Settings">
        @Body
    </CascadingValue>
</CascadingValue>

@* Consumer *@
@code {
    [CascadingParameter(Name = "CurrentUser")]
    public UserInfo User { get; set; } = default!;

    [CascadingParameter(Name = "Settings")]
    public AppSettings Settings { get; set; } = default!;
}
```

### Fixed Cascading Values (Performance Optimization)

```razor
@* Use IsFixed when the value never changes *@
<CascadingValue Value="httpClient" IsFixed="true">
    @ChildContent
</CascadingValue>
```

---

## 5. Component Lifecycle

### Lifecycle Methods Reference

| Method | When Called | Use Case |
|--------|-----------|----------|
| `SetParametersAsync` | Before parameters are set | Advanced: intercept raw parameters |
| `OnInitialized` | After first render initialization | Sync setup |
| `OnInitializedAsync` | After first render initialization | Async setup (API calls) |
| `OnParametersSet` | After parameters are set/changed | React to parameter changes |
| `OnParametersSetAsync` | After parameters are set/changed | Async reaction to parameter changes |
| `OnAfterRender(bool firstRender)` | After DOM render | JS interop, focus management |
| `OnAfterRenderAsync(bool firstRender)` | After DOM render | Async JS interop |
| `ShouldRender` | Before each render | Skip unnecessary renders |
| `Dispose` / `DisposeAsync` | Component removed from UI | Cleanup resources |

### Lifecycle Flow Diagram

```
Component Created
       │
       ▼
SetParametersAsync
       │
       ▼
OnInitialized / OnInitializedAsync  ◄── Only on first render
       │
       ▼
OnParametersSet / OnParametersSetAsync  ◄── On every parameter change
       │
       ▼
ShouldRender  ──── false ──► (skip render)
       │ true
       ▼
BuildRenderTree (Render)
       │
       ▼
OnAfterRender / OnAfterRenderAsync
       │
       ▼
(Awaiting next state change or parameter update)
       │
       ▼
Dispose / DisposeAsync  ◄── Component removed
```

### Practical Lifecycle Example

```razor
@page "/user/{UserId}"
@implements IAsyncDisposable

<h3>@user?.Name</h3>
<p>@user?.Email</p>

@if (isLoading)
{
    <div class="spinner"></div>
}

@code {
    [Parameter]
    public string UserId { get; set; } = "";

    [Inject]
    private IUserService UserService { get; set; } = default!;

    [Inject]
    private ILogger<UserProfile> Logger { get; set; } = default!;

    private UserDto? user;
    private bool isLoading = true;
    private CancellationTokenSource cts = new();

    protected override async Task OnInitializedAsync()
    {
        Logger.LogInformation("Component initialized for user {UserId}", UserId);
        await LoadUserAsync();
    }

    protected override async Task OnParametersSetAsync()
    {
        // Called when UserId route parameter changes
        Logger.LogInformation("Parameters set: UserId={UserId}", UserId);
        await LoadUserAsync();
    }

    protected override bool ShouldRender()
    {
        // Optionally skip render (e.g., during background timer updates)
        return true;
    }

    protected override void OnAfterRender(bool firstRender)
    {
        if (firstRender)
        {
            Logger.LogInformation("First render complete");
        }
    }

    private async Task LoadUserAsync()
    {
        isLoading = true;
        try
        {
            user = await UserService.GetByIdAsync(UserId, cts.Token);
        }
        catch (OperationCanceledException)
        {
            // Component disposed during load
        }
        finally
        {
            isLoading = false;
        }
    }

    public async ValueTask DisposeAsync()
    {
        Logger.LogInformation("Component disposing");
        cts.Cancel();
        cts.Dispose();
    }
}
```

---

## 6. StateHasChanged and Manual Rendering

`StateHasChanged` notifies the component that its state has changed and a re-render is needed.

```razor
@implements IDisposable

<p>Time: @currentTime.ToString("HH:mm:ss")</p>

@code {
    private DateTime currentTime = DateTime.Now;
    private Timer? timer;

    protected override void OnInitialized()
    {
        timer = new Timer(_ =>
        {
            currentTime = DateTime.Now;
            // Must use InvokeAsync from non-UI thread
            InvokeAsync(StateHasChanged);
        }, null, 0, 1000);
    }

    public void Dispose()
    {
        timer?.Dispose();
    }
}
```

**When StateHasChanged is called automatically:**
- After an event handler (`@onclick`, `@onchange`, etc.)
- After `OnInitializedAsync` or `OnParametersSetAsync` completes
- After awaited `Task` in lifecycle methods

**When you must call it manually:**
- Timer callbacks
- External event handlers (SignalR `On()`, `IObservable` subscriptions)
- Background task completions

---

## 7. Component References

Use `@ref` to get a reference to a child component for direct method invocation.

```razor
@* ChildComponent.razor *@
<div>@message</div>

@code {
    private string message = "Initial";

    public void UpdateMessage(string newMessage)
    {
        message = newMessage;
        StateHasChanged();
    }

    public string GetCurrentMessage() => message;
}
```

```razor
@* Parent.razor *@
<ChildComponent @ref="childRef" />
<button @onclick="UpdateChild">Update Child</button>

@code {
    private ChildComponent? childRef;

    private void UpdateChild()
    {
        childRef?.UpdateMessage("Updated from parent!");
    }
}
```

**Note:** `@ref` is populated after `OnAfterRender`. It is `null` during `OnInitialized` and `OnParametersSet`.

---

## 8. Key Attribute for List Rendering

Use `@key` to help Blazor's diffing algorithm track elements in lists.

```razor
@* Without @key: Blazor may reuse wrong elements when list order changes *@
@foreach (var item in items)
{
    <TodoItem @key="item.Id" Item="item" OnDelete="HandleDelete" />
}

@* With @key on elements *@
<div>
    @foreach (var person in people)
    {
        <div @key="person.Id" class="person-card">
            <PersonEditor Person="person" />
        </div>
    }
</div>

@code {
    private List<TodoDto> items = [];
    private List<PersonDto> people = [];

    private void HandleDelete(TodoDto item) => items.Remove(item);
}
```

**When to use `@key`:**
- Lists that are reordered, inserted into, or removed from
- Components with internal state (form inputs, expanded/collapsed)
- Any collection rendering where element identity matters

**Do not use `@key` with the loop index** -- it defeats the purpose since indices change on reorder.

---

## 9. Templated Components (RenderFragment)

RenderFragment allows components to accept UI content as parameters.

### Basic ChildContent

```razor
@* Card.razor *@
<div class="card">
    <div class="card-header">@Title</div>
    <div class="card-body">
        @ChildContent
    </div>
    @if (Footer is not null)
    {
        <div class="card-footer">@Footer</div>
    }
</div>

@code {
    [Parameter]
    public string Title { get; set; } = "";

    [Parameter]
    public RenderFragment? ChildContent { get; set; }

    [Parameter]
    public RenderFragment? Footer { get; set; }
}
```

```razor
<Card Title="User Profile">
    <p>Name: John Doe</p>
    <p>Email: john@example.com</p>
    <Footer>
        <button class="btn btn-primary">Edit</button>
    </Footer>
</Card>
```

### Typed RenderFragment\<T\>

```razor
@* DataTable.razor *@
@typeparam TItem

<table class="table">
    <thead>
        <tr>@Header</tr>
    </thead>
    <tbody>
        @foreach (var item in Items)
        {
            <tr @key="item">@Row(item)</tr>
        }
    </tbody>
</table>

@if (!Items.Any())
{
    @if (EmptyContent is not null)
    {
        @EmptyContent
    }
    else
    {
        <p>No items to display.</p>
    }
}

@code {
    [Parameter, EditorRequired]
    public IReadOnlyList<TItem> Items { get; set; } = [];

    [Parameter, EditorRequired]
    public RenderFragment Header { get; set; } = default!;

    [Parameter, EditorRequired]
    public RenderFragment<TItem> Row { get; set; } = default!;

    [Parameter]
    public RenderFragment? EmptyContent { get; set; }
}
```

```razor
<DataTable Items="users" Context="user">
    <Header>
        <th>Name</th>
        <th>Email</th>
        <th>Actions</th>
    </Header>
    <Row>
        <td>@user.Name</td>
        <td>@user.Email</td>
        <td><button @onclick="() => Edit(user)">Edit</button></td>
    </Row>
    <EmptyContent>
        <p class="text-muted">No users found. Add one to get started.</p>
    </EmptyContent>
</DataTable>
```

---

## 10. Generic Components

```razor
@* SelectList.razor *@
@typeparam TItem where TItem : notnull

<select class="form-select" @onchange="HandleChange">
    @if (Placeholder is not null)
    {
        <option value="">@Placeholder</option>
    }
    @foreach (var item in Items)
    {
        <option value="@GetValue(item)" selected="@(EqualityComparer<TItem>.Default.Equals(item, SelectedItem))">
            @GetDisplay(item)
        </option>
    }
</select>

@code {
    [Parameter, EditorRequired]
    public IReadOnlyList<TItem> Items { get; set; } = [];

    [Parameter]
    public TItem? SelectedItem { get; set; }

    [Parameter]
    public EventCallback<TItem?> SelectedItemChanged { get; set; }

    [Parameter]
    public Func<TItem, string> DisplayExpression { get; set; } = item => item.ToString()!;

    [Parameter]
    public Func<TItem, string> ValueExpression { get; set; } = item => item.ToString()!;

    [Parameter]
    public string? Placeholder { get; set; }

    private string GetDisplay(TItem item) => DisplayExpression(item);
    private string GetValue(TItem item) => ValueExpression(item);

    private async Task HandleChange(ChangeEventArgs e)
    {
        var value = e.Value?.ToString();
        var selected = Items.FirstOrDefault(i => GetValue(i) == value);
        await SelectedItemChanged.InvokeAsync(selected);
    }
}
```

```razor
<SelectList TItem="Country"
            Items="countries"
            @bind-SelectedItem="selectedCountry"
            DisplayExpression="c => c.Name"
            ValueExpression="c => c.Code"
            Placeholder="Select a country..." />
```

---

## 11. Sections (.NET 8+)

Sections allow components to provide content to specific outlets in the layout.

### Layout with SectionOutlet

```razor
@* MainLayout.razor *@
@inherits LayoutComponentBase

<header>
    <nav>
        <SectionOutlet SectionName="page-title" />
    </nav>
    <div class="toolbar">
        <SectionOutlet SectionName="toolbar-actions" />
    </div>
</header>

<main>
    @Body
</main>
```

### Page Providing Section Content

```razor
@page "/orders"

<SectionContent SectionName="page-title">
    <h1>Orders Management</h1>
</SectionContent>

<SectionContent SectionName="toolbar-actions">
    <button @onclick="CreateOrder" class="btn btn-primary">New Order</button>
    <button @onclick="ExportOrders" class="btn btn-secondary">Export</button>
</SectionContent>

<div class="order-list">
    @* Order list content *@
</div>

@code {
    private void CreateOrder() { /* ... */ }
    private void ExportOrders() { /* ... */ }
}
```

---

## 12. Error Boundaries

Error boundaries catch exceptions in child component rendering and display fallback UI.

```razor
<ErrorBoundary @ref="errorBoundary">
    <ChildContent>
        <RiskyComponent />
    </ChildContent>
    <ErrorContent Context="exception">
        <div class="alert alert-danger">
            <h4>Something went wrong</h4>
            <p>@exception.Message</p>
            <button class="btn btn-primary" @onclick="Recover">Try Again</button>
        </div>
    </ErrorContent>
</ErrorBoundary>

@code {
    private ErrorBoundary? errorBoundary;

    private void Recover()
    {
        errorBoundary?.Recover();
    }
}
```

### Nested Error Boundaries

```razor
<div class="dashboard">
    @foreach (var widget in widgets)
    {
        <ErrorBoundary>
            <ChildContent>
                <DashboardWidget @key="widget.Id" Config="widget" />
            </ChildContent>
            <ErrorContent>
                <div class="widget-error">
                    <p>Widget failed to load.</p>
                </div>
            </ErrorContent>
        </ErrorBoundary>
    }
</div>
```

---

## 13. Virtualization

The `Virtualize` component renders only visible items for large lists.

### Basic Virtualization

```razor
<div style="height: 600px; overflow-y: auto;">
    <Virtualize Items="allProducts" Context="product">
        <div class="product-row">
            <span>@product.Name</span>
            <span>@product.Price.ToString("C")</span>
        </div>
    </Virtualize>
</div>

@code {
    private List<Product> allProducts = []; // Could be thousands of items

    protected override async Task OnInitializedAsync()
    {
        allProducts = await ProductService.GetAllAsync();
    }
}
```

### Virtualization with Item Provider (On-Demand Loading)

```razor
<div style="height: 600px; overflow-y: auto;">
    <Virtualize ItemsProvider="LoadProducts" Context="product" ItemSize="50">
        <ItemContent>
            <div class="product-row" style="height: 50px;">
                <span>@product.Name</span>
                <span>@product.Price.ToString("C")</span>
            </div>
        </ItemContent>
        <Placeholder>
            <div class="product-row skeleton" style="height: 50px;">
                Loading...
            </div>
        </Placeholder>
    </Virtualize>
</div>

@code {
    private async ValueTask<ItemsProviderResult<Product>> LoadProducts(
        ItemsProviderRequest request)
    {
        var result = await ProductService.GetPagedAsync(
            request.StartIndex,
            request.Count,
            request.CancellationToken);

        return new ItemsProviderResult<Product>(result.Items, result.TotalCount);
    }
}
```

---

## 14. DynamicComponent

Render components dynamically by type at runtime.

```razor
@page "/dynamic-page"

<select @bind="selectedComponent">
    <option value="">Choose a component</option>
    @foreach (var (name, type) in availableComponents)
    {
        <option value="@name">@name</option>
    }
</select>

@if (currentType is not null)
{
    <DynamicComponent Type="currentType" Parameters="componentParameters" />
}

@code {
    private string selectedComponent = "";
    private Type? currentType;

    private Dictionary<string, Type> availableComponents = new()
    {
        ["Chart"] = typeof(ChartWidget),
        ["Table"] = typeof(TableWidget),
        ["Summary"] = typeof(SummaryWidget)
    };

    private Dictionary<string, object> componentParameters = new()
    {
        ["Title"] = "Dynamic Content",
        ["ShowHeader"] = true
    };

    private string SelectedComponent
    {
        get => selectedComponent;
        set
        {
            selectedComponent = value;
            currentType = string.IsNullOrEmpty(value) ? null : availableComponents[value];
        }
    }
}
```

---

## 15. CSS Isolation

Scoped styles that apply only to the component.

```css
/* Counter.razor.css */
h3 {
    color: #1a73e8;
    font-weight: 600;
}

.counter-value {
    font-size: 2rem;
    font-weight: bold;
}

::deep .child-class {
    /* Applies to child component elements */
    color: gray;
}
```

The Blazor build process rewrites selectors to include a unique scope identifier (e.g., `h3[b-abc123]`), ensuring styles do not leak to other components.

---

## 16. QuickGrid

QuickGrid is a high-performance grid component for displaying tabular data.

```razor
@page "/orders"

<QuickGrid Items="orders" Pagination="pagination" Theme="default">
    <PropertyColumn Property="@(o => o.Id)" Title="Order #" Sortable="true" />
    <PropertyColumn Property="@(o => o.CustomerName)" Title="Customer" Sortable="true" />
    <PropertyColumn Property="@(o => o.Total)" Title="Total" Format="C2" Sortable="true" />
    <PropertyColumn Property="@(o => o.Status)" Title="Status" Sortable="true" />
    <TemplateColumn Title="Actions">
        <button @onclick="() => ViewOrder(context.Id)" class="btn btn-sm btn-primary">
            View
        </button>
    </TemplateColumn>
</QuickGrid>

<Paginator State="pagination" />

@code {
    private IQueryable<Order> orders = default!;
    private PaginationState pagination = new() { ItemsPerPage = 25 };

    [Inject] private IOrderService OrderService { get; set; } = default!;

    protected override async Task OnInitializedAsync()
    {
        var orderList = await OrderService.GetAllAsync();
        orders = orderList.AsQueryable();
    }

    private void ViewOrder(int id) => Navigation.NavigateTo($"/orders/{id}");
}
```

---

## 17. Component Disposal

Implement `IDisposable` or `IAsyncDisposable` to clean up resources.

```razor
@implements IAsyncDisposable

@code {
    private CancellationTokenSource cts = new();
    private Timer? refreshTimer;
    private IDisposable? eventSubscription;

    protected override void OnInitialized()
    {
        refreshTimer = new Timer(async _ =>
        {
            await InvokeAsync(async () =>
            {
                await RefreshDataAsync();
                StateHasChanged();
            });
        }, null, TimeSpan.Zero, TimeSpan.FromSeconds(30));

        eventSubscription = EventBus.Subscribe<OrderCreatedEvent>(OnOrderCreated);
    }

    public async ValueTask DisposeAsync()
    {
        cts.Cancel();
        cts.Dispose();

        if (refreshTimer is not null)
        {
            await refreshTimer.DisposeAsync();
        }

        eventSubscription?.Dispose();
    }
}
```

---

## Best Practices

1. **Use `[EditorRequired]` for mandatory parameters** -- Provides compile-time warnings when parameters are missing.
2. **Prefer EventCallback over Action delegates** -- `EventCallback` automatically triggers re-rendering.
3. **Use `@key` on list-rendered components** -- Preserves component state during list reordering.
4. **Implement `IAsyncDisposable`** -- Clean up subscriptions, timers, and cancellation tokens.
5. **Use code-behind for complex logic** -- Keep `.razor` files focused on markup.
6. **Prefer cascading values for cross-cutting concerns** -- Theme, auth, locale.
7. **Use `IsFixed="true"` for static cascading values** -- Avoids unnecessary re-renders.
8. **Minimize `StateHasChanged` calls** -- Blazor calls it automatically after events and lifecycle methods.
9. **Use Virtualize for large lists** -- Renders only visible items for better performance.
10. **Use ErrorBoundary per independent UI section** -- Prevents one widget crash from breaking the whole page.

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|---------|
| Calling `StateHasChanged` from background thread | `InvalidOperationException` | Wrap in `InvokeAsync(StateHasChanged)` |
| Using `@ref` in `OnInitialized` | Reference is `null` | Use `@ref` in `OnAfterRender(firstRender: true)` |
| Setting parameter properties in child component | Overwritten by parent on re-render | Use private backing fields; notify parent via `EventCallback` |
| Using loop variable directly in lambda | Closure captures wrong value | Capture in local: `var local = item; @onclick="() => Select(local)"` |
| Missing `@key` on dynamic lists | Stale component state after reorder | Add `@key="item.UniqueId"` |
| Not disposing subscriptions | Memory leaks, `ObjectDisposedException` | Implement `IDisposable` / `IAsyncDisposable` |
| Calling `StateHasChanged` in `OnInitialized` | No-op (render has not happened yet) | Remove unnecessary call; Blazor renders after init |
| Cascading value updates not propagating | Child uses stale value | Ensure cascading value is not marked `IsFixed` when it changes |
| Expensive computation in `BuildRenderTree` | Slow renders | Cache computed values in fields; update in lifecycle methods |
| Using `typeof(T)` in generic components | Compile error | Use `@typeparam TItem` and reference `TItem` directly |
