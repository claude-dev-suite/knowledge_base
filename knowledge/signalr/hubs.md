# SignalR Hubs

> Official Documentation: https://learn.microsoft.com/aspnet/core/signalr/hubs

Hubs are the core abstraction in ASP.NET Core SignalR for real-time client-server communication. A hub is a high-level pipeline that allows clients and servers to call methods on each other directly.

---

## 1. Hub Class Basics

A hub inherits from `Hub` and exposes public methods that clients can invoke.

```csharp
using Microsoft.AspNetCore.SignalR;

public class ChatHub : Hub
{
    public async Task SendMessage(string user, string message)
    {
        await Clients.All.SendAsync("ReceiveMessage", user, message);
    }

    public async Task SendPrivateMessage(string connectionId, string message)
    {
        await Clients.Client(connectionId).SendAsync("ReceivePrivateMessage", message);
    }
}
```

### Service Registration and Mapping

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSignalR();

var app = builder.Build();

app.MapHub<ChatHub>("/chatHub");

app.Run();
```

### Hub Method Rules

| Rule | Description |
|------|-------------|
| Public methods only | Only `public` methods are callable from clients |
| Return types | `Task`, `Task<T>`, `ValueTask`, `ValueTask<T>`, `IAsyncEnumerable<T>`, `ChannelReader<T>` |
| Parameters | Supports complex objects, collections, and `CancellationToken` |
| Overloads | Method overloading is **not supported** (use different method names) |
| Naming | Client sees PascalCase method names by default |

---

## 2. Sending Messages to Clients

The `Clients` property provides multiple targeting options for sending messages.

### Client Targeting Reference

| Property | Description |
|----------|-------------|
| `Clients.All` | All connected clients |
| `Clients.Caller` | The calling client only |
| `Clients.Others` | All clients except the caller |
| `Clients.Client(connectionId)` | A specific client by connection ID |
| `Clients.Clients(IReadOnlyList<string>)` | Multiple specific clients |
| `Clients.Group(groupName)` | All clients in a group |
| `Clients.Groups(IReadOnlyList<string>)` | All clients in multiple groups |
| `Clients.GroupExcept(groupName, excludedIds)` | Group members except specified |
| `Clients.OthersInGroup(groupName)` | Group members except caller |
| `Clients.User(userId)` | All connections for a specific user |
| `Clients.Users(IReadOnlyList<string>)` | All connections for specific users |

### Practical Examples

```csharp
public class NotificationHub : Hub
{
    // Broadcast to everyone
    public async Task BroadcastAlert(string message)
    {
        await Clients.All.SendAsync("Alert", message);
    }

    // Send only to the caller (acknowledgement pattern)
    public async Task JoinRoom(string roomName)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, roomName);
        await Clients.Caller.SendAsync("JoinedRoom", roomName);
        await Clients.OthersInGroup(roomName).SendAsync("UserJoined", Context.UserIdentifier);
    }

    // Send to a specific user (all their connections)
    public async Task NotifyUser(string userId, string notification)
    {
        await Clients.User(userId).SendAsync("Notification", notification);
    }

    // Exclude certain connections
    public async Task BroadcastExcept(string message, List<string> excludedConnectionIds)
    {
        await Clients.AllExcept(excludedConnectionIds).SendAsync("Broadcast", message);
    }
}
```

---

## 3. Strongly-Typed Hubs

Strongly-typed hubs replace magic string method names with compile-time checked interfaces.

### Define the Client Interface

```csharp
public interface IChatClient
{
    Task ReceiveMessage(string user, string message);
    Task ReceivePrivateMessage(string fromUser, string message);
    Task UserJoined(string user);
    Task UserLeft(string user);
    Task ReceiveTypingIndicator(string user, bool isTyping);
    Task UpdateOnlineUsers(List<string> users);
}
```

### Implement the Strongly-Typed Hub

```csharp
public class ChatHub : Hub<IChatClient>
{
    private static readonly ConcurrentDictionary<string, string> OnlineUsers = new();

    public async Task SendMessage(string message)
    {
        var user = Context.UserIdentifier ?? Context.ConnectionId;
        await Clients.All.ReceiveMessage(user, message);
    }

    public async Task SendPrivateMessage(string targetUserId, string message)
    {
        var fromUser = Context.UserIdentifier ?? "Anonymous";
        await Clients.User(targetUserId).ReceivePrivateMessage(fromUser, message);
        await Clients.Caller.ReceivePrivateMessage(fromUser, message); // echo
    }

    public async Task SetTyping(string groupName, bool isTyping)
    {
        var user = Context.UserIdentifier ?? Context.ConnectionId;
        await Clients.OthersInGroup(groupName).ReceiveTypingIndicator(user, isTyping);
    }

    public override async Task OnConnectedAsync()
    {
        var user = Context.UserIdentifier ?? Context.ConnectionId;
        OnlineUsers.TryAdd(Context.ConnectionId, user);
        await Clients.All.UserJoined(user);
        await Clients.All.UpdateOnlineUsers(OnlineUsers.Values.Distinct().ToList());
        await base.OnConnectedAsync();
    }

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        OnlineUsers.TryRemove(Context.ConnectionId, out var user);
        await Clients.All.UserLeft(user ?? Context.ConnectionId);
        await Clients.All.UpdateOnlineUsers(OnlineUsers.Values.Distinct().ToList());
        await base.OnDisconnectedAsync(exception);
    }
}
```

**Benefits of strongly-typed hubs:**
- Compile-time checking of method names and parameter types
- IDE autocompletion for client methods
- Refactoring safety when renaming methods
- Self-documenting client API contract

---

## 4. Hub Context

The `Context` property provides information about the current connection.

### Context Properties

| Property | Type | Description |
|----------|------|-------------|
| `Context.ConnectionId` | `string` | Unique ID for this connection |
| `Context.UserIdentifier` | `string?` | User identifier (from claims) |
| `Context.User` | `ClaimsPrincipal` | The authenticated user |
| `Context.Items` | `IDictionary<object, object?>` | Per-connection key/value storage |
| `Context.Features` | `IFeatureCollection` | HTTP feature collection |
| `Context.ConnectionAborted` | `CancellationToken` | Fires when connection is aborted |

### Accessing User Information

```csharp
public class SecureHub : Hub<ISecureClient>
{
    public async Task GetProfile()
    {
        var userId = Context.UserIdentifier; // NameIdentifier claim by default
        var userName = Context.User?.FindFirst(ClaimTypes.Name)?.Value;
        var email = Context.User?.FindFirst(ClaimTypes.Email)?.Value;
        var roles = Context.User?.FindAll(ClaimTypes.Role).Select(c => c.Value).ToList();

        await Clients.Caller.ReceiveProfile(new UserProfile
        {
            Id = userId!,
            Name = userName!,
            Email = email!,
            Roles = roles ?? []
        });
    }
}
```

### Custom UserIdentifier Provider

```csharp
public class EmailBasedUserIdProvider : IUserIdProvider
{
    public string? GetUserId(HubConnectionContext connection)
    {
        return connection.User?.FindFirst(ClaimTypes.Email)?.Value;
    }
}

// Registration
builder.Services.AddSingleton<IUserIdProvider, EmailBasedUserIdProvider>();
```

### Per-Connection State with Context.Items

```csharp
public class GameHub : Hub
{
    public async Task JoinGame(string gameId)
    {
        Context.Items["GameId"] = gameId;
        Context.Items["JoinedAt"] = DateTime.UtcNow;
        await Groups.AddToGroupAsync(Context.ConnectionId, $"game-{gameId}");
    }

    public async Task MakeMove(int x, int y)
    {
        var gameId = Context.Items["GameId"] as string;
        if (gameId is null)
        {
            throw new HubException("You are not in a game.");
        }
        await Clients.OthersInGroup($"game-{gameId}")
            .SendAsync("MoveMade", Context.ConnectionId, x, y);
    }
}
```

---

## 5. Groups

Groups are the primary mechanism for targeting subsets of clients.

```csharp
public class CollaborationHub : Hub<ICollaborationClient>
{
    public async Task JoinDocument(string documentId)
    {
        var groupName = $"doc-{documentId}";
        await Groups.AddToGroupAsync(Context.ConnectionId, groupName);
        await Clients.OthersInGroup(groupName)
            .UserJoinedDocument(Context.UserIdentifier!, documentId);
    }

    public async Task LeaveDocument(string documentId)
    {
        var groupName = $"doc-{documentId}";
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, groupName);
        await Clients.OthersInGroup(groupName)
            .UserLeftDocument(Context.UserIdentifier!, documentId);
    }

    public async Task EditDocument(string documentId, DocumentEdit edit)
    {
        var groupName = $"doc-{documentId}";
        await Clients.OthersInGroup(groupName)
            .ReceiveEdit(Context.UserIdentifier!, edit);
    }

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        // Groups are automatically cleaned up on disconnect.
        // No need to manually remove from groups.
        await base.OnDisconnectedAsync(exception);
    }
}
```

**Group behavior:**
- Groups are created implicitly when the first client is added
- Groups are destroyed automatically when the last client is removed
- A connection can belong to multiple groups simultaneously
- Group membership does not persist across reconnections (must re-join)

---

## 6. Hub Lifetime Events

Override `OnConnectedAsync` and `OnDisconnectedAsync` for connection tracking.

```csharp
public class PresenceHub : Hub<IPresenceClient>
{
    private readonly IPresenceService _presence;
    private readonly ILogger<PresenceHub> _logger;

    public PresenceHub(IPresenceService presence, ILogger<PresenceHub> logger)
    {
        _presence = presence;
        _logger = logger;
    }

    public override async Task OnConnectedAsync()
    {
        var userId = Context.UserIdentifier ?? throw new HubException("Not authenticated");

        _logger.LogInformation("User {UserId} connected with {ConnectionId}",
            userId, Context.ConnectionId);

        await _presence.UserConnectedAsync(userId, Context.ConnectionId);
        var onlineUsers = await _presence.GetOnlineUsersAsync();

        // Tell the caller who is online
        await Clients.Caller.SyncOnlineUsers(onlineUsers);
        // Tell everyone else this user came online
        await Clients.Others.UserOnline(userId);

        await base.OnConnectedAsync();
    }

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        var userId = Context.UserIdentifier;

        if (exception is not null)
        {
            _logger.LogWarning(exception, "User {UserId} disconnected with error", userId);
        }

        if (userId is not null)
        {
            var hasOtherConnections = await _presence
                .UserDisconnectedAsync(userId, Context.ConnectionId);

            if (!hasOtherConnections)
            {
                await Clients.Others.UserOffline(userId);
            }
        }

        await base.OnDisconnectedAsync(exception);
    }
}
```

---

## 7. Hub Filters

Hub filters are analogous to ASP.NET Core middleware but for SignalR hub method invocations.

```csharp
public class LoggingHubFilter : IHubFilter
{
    private readonly ILogger<LoggingHubFilter> _logger;

    public LoggingHubFilter(ILogger<LoggingHubFilter> logger)
    {
        _logger = logger;
    }

    public async ValueTask<object?> InvokeMethodAsync(
        HubInvocationContext invocationContext,
        Func<HubInvocationContext, ValueTask<object?>> next)
    {
        var hubName = invocationContext.Hub.GetType().Name;
        var methodName = invocationContext.HubMethodName;
        var connectionId = invocationContext.Context.ConnectionId;

        _logger.LogInformation(
            "Hub {Hub}.{Method} invoked by {ConnectionId}",
            hubName, methodName, connectionId);

        var stopwatch = Stopwatch.StartNew();
        try
        {
            var result = await next(invocationContext);
            stopwatch.Stop();
            _logger.LogInformation(
                "Hub {Hub}.{Method} completed in {ElapsedMs}ms",
                hubName, methodName, stopwatch.ElapsedMilliseconds);
            return result;
        }
        catch (Exception ex)
        {
            stopwatch.Stop();
            _logger.LogError(ex,
                "Hub {Hub}.{Method} failed after {ElapsedMs}ms",
                hubName, methodName, stopwatch.ElapsedMilliseconds);
            throw;
        }
    }

    public Task OnConnectedAsync(
        HubLifetimeContext context,
        Func<HubLifetimeContext, Task> next)
    {
        _logger.LogInformation("Connection {ConnectionId} established",
            context.Context.ConnectionId);
        return next(context);
    }

    public Task OnDisconnectedAsync(
        HubLifetimeContext context,
        Exception? exception,
        Func<HubLifetimeContext, Exception?, Task> next)
    {
        _logger.LogInformation("Connection {ConnectionId} terminated",
            context.Context.ConnectionId);
        return next(context, exception);
    }
}

// Registration
builder.Services.AddSignalR(options =>
{
    options.AddFilter<LoggingHubFilter>();
});
```

### Rate-Limiting Hub Filter

```csharp
public class RateLimitingHubFilter : IHubFilter
{
    private readonly ConcurrentDictionary<string, (int Count, DateTime Window)> _tracker = new();
    private readonly int _maxCallsPerWindow = 50;
    private readonly TimeSpan _windowDuration = TimeSpan.FromSeconds(10);

    public async ValueTask<object?> InvokeMethodAsync(
        HubInvocationContext invocationContext,
        Func<HubInvocationContext, ValueTask<object?>> next)
    {
        var connectionId = invocationContext.Context.ConnectionId;
        var now = DateTime.UtcNow;

        var entry = _tracker.AddOrUpdate(
            connectionId,
            _ => (1, now),
            (_, existing) =>
            {
                if (now - existing.Window > _windowDuration)
                    return (1, now);
                return (existing.Count + 1, existing.Window);
            });

        if (entry.Count > _maxCallsPerWindow)
        {
            throw new HubException("Rate limit exceeded. Please slow down.");
        }

        return await next(invocationContext);
    }
}
```

---

## 8. Error Handling

### HubException

`HubException` is the only exception type whose message is sent to the client. All other exceptions result in a generic error message.

```csharp
public class OrderHub : Hub<IOrderClient>
{
    private readonly IOrderService _orders;

    public OrderHub(IOrderService orders) => _orders = orders;

    public async Task PlaceOrder(OrderRequest request)
    {
        // Validate input
        if (string.IsNullOrWhiteSpace(request.ProductId))
        {
            throw new HubException("Product ID is required.");
        }

        if (request.Quantity <= 0)
        {
            throw new HubException("Quantity must be greater than zero.");
        }

        try
        {
            var order = await _orders.PlaceOrderAsync(request);
            await Clients.Caller.OrderConfirmed(order);
            await Clients.Group("admins").NewOrderPlaced(order);
        }
        catch (InsufficientStockException ex)
        {
            // HubException message IS sent to client
            throw new HubException($"Insufficient stock: {ex.Available} available.");
        }
        catch (Exception ex)
        {
            // Generic exceptions are NOT sent to client in production
            // Client receives: "An unexpected error occurred."
            throw new HubException("Failed to place order. Please try again.");
        }
    }
}
```

### Enabling Detailed Errors (Development Only)

```csharp
builder.Services.AddSignalR(options =>
{
    if (builder.Environment.IsDevelopment())
    {
        options.EnableDetailedErrors = true; // Sends full exception messages to client
    }
});
```

---

## 9. Authentication and Authorization

### Securing the Entire Hub

```csharp
[Authorize]
public class AdminHub : Hub<IAdminClient>
{
    [Authorize(Roles = "Admin")]
    public async Task DeleteUser(string userId)
    {
        // Only Admin role can invoke this
        await Clients.All.UserDeleted(userId);
    }

    [Authorize(Policy = "RequireSuperAdmin")]
    public async Task ResetSystem()
    {
        await Clients.All.SystemReset();
    }

    // Accessible to any authenticated user (hub-level [Authorize])
    public async Task GetSystemStatus()
    {
        await Clients.Caller.SystemStatus(new { Uptime = "12h" });
    }
}
```

### JWT Bearer Authentication for SignalR

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidateAudience = true,
            ValidAudience = builder.Configuration["Jwt:Audience"],
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]!))
        };

        // SignalR sends tokens via query string for WebSockets
        options.Events = new JwtBearerEvents
        {
            OnMessageReceived = context =>
            {
                var accessToken = context.Request.Query["access_token"];
                var path = context.HttpContext.Request.Path;
                if (!string.IsNullOrEmpty(accessToken) && path.StartsWithSegments("/hubs"))
                {
                    context.Token = accessToken;
                }
                return Task.CompletedTask;
            }
        };
    });
```

---

## 10. IHubContext for Background Services

Use `IHubContext<THub>` or `IHubContext<THub, TClient>` to send messages from outside a hub.

```csharp
public class OrderProcessingService : BackgroundService
{
    private readonly IHubContext<OrderHub, IOrderClient> _hubContext;
    private readonly IOrderQueue _queue;
    private readonly ILogger<OrderProcessingService> _logger;

    public OrderProcessingService(
        IHubContext<OrderHub, IOrderClient> hubContext,
        IOrderQueue queue,
        ILogger<OrderProcessingService> logger)
    {
        _hubContext = hubContext;
        _queue = queue;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var order in _queue.DequeueAllAsync(stoppingToken))
        {
            _logger.LogInformation("Processing order {OrderId}", order.Id);

            // Notify the specific user who placed the order
            await _hubContext.Clients.User(order.UserId)
                .OrderStatusChanged(order.Id, "Processing");

            // Process the order...
            await ProcessOrderAsync(order, stoppingToken);

            // Notify completion
            await _hubContext.Clients.User(order.UserId)
                .OrderStatusChanged(order.Id, "Completed");

            // Notify admin group
            await _hubContext.Clients.Group("admins")
                .OrderProcessed(order);
        }
    }

    private Task ProcessOrderAsync(Order order, CancellationToken ct) => Task.Delay(100, ct);
}
```

### Using IHubContext in a Controller

```csharp
[ApiController]
[Route("api/[controller]")]
public class NotificationsController : ControllerBase
{
    private readonly IHubContext<NotificationHub, INotificationClient> _hubContext;

    public NotificationsController(IHubContext<NotificationHub, INotificationClient> hubContext)
    {
        _hubContext = hubContext;
    }

    [HttpPost("broadcast")]
    public async Task<IActionResult> Broadcast([FromBody] NotificationDto notification)
    {
        await _hubContext.Clients.All.ReceiveNotification(notification);
        return Ok();
    }

    [HttpPost("user/{userId}")]
    public async Task<IActionResult> NotifyUser(string userId, [FromBody] NotificationDto notification)
    {
        await _hubContext.Clients.User(userId).ReceiveNotification(notification);
        return Ok();
    }

    [HttpPost("group/{groupName}")]
    public async Task<IActionResult> NotifyGroup(string groupName, [FromBody] NotificationDto notification)
    {
        await _hubContext.Clients.Group(groupName).ReceiveNotification(notification);
        return Ok();
    }
}
```

---

## 11. SignalR Configuration

### Comprehensive Configuration

```csharp
builder.Services.AddSignalR(options =>
{
    // Timeouts
    options.KeepAliveInterval = TimeSpan.FromSeconds(15);       // Server pings client
    options.ClientTimeoutInterval = TimeSpan.FromSeconds(30);   // Client considered disconnected
    options.HandshakeTimeout = TimeSpan.FromSeconds(15);        // Time to complete handshake

    // Message size
    options.MaximumReceiveMessageSize = 64 * 1024;              // 64 KB (null = unlimited)
    options.StreamBufferCapacity = 10;                          // Items buffered per stream
    options.MaximumParallelInvocationsPerClient = 1;            // Concurrent hub calls per client

    // Errors
    options.EnableDetailedErrors = false;                       // true only for development

    // Filters
    options.AddFilter<LoggingHubFilter>();
    options.AddFilter<RateLimitingHubFilter>();
});
```

### Configuration Reference

| Option | Default | Description |
|--------|---------|-------------|
| `KeepAliveInterval` | 15 seconds | Interval for server-to-client ping |
| `ClientTimeoutInterval` | 30 seconds | Time before client is considered disconnected |
| `HandshakeTimeout` | 15 seconds | Maximum time for initial handshake |
| `MaximumReceiveMessageSize` | 32 KB | Max incoming message size (null = unlimited) |
| `StreamBufferCapacity` | 10 | Buffer size for client upload streams |
| `MaximumParallelInvocationsPerClient` | 1 | Concurrent hub calls per client |
| `EnableDetailedErrors` | false | Send exception details to client |
| `DisableImplicitFromServicesParameters` | false | Disable automatic DI parameter injection |

---

## 12. MessagePack Protocol

MessagePack is a binary serialization format that produces smaller messages than JSON.

```csharp
// Install: dotnet add package Microsoft.AspNetCore.SignalR.Protocols.MessagePack

builder.Services.AddSignalR()
    .AddMessagePackProtocol(options =>
    {
        options.SerializerOptions = MessagePack.MessagePackSerializerOptions.Standard
            .WithSecurity(MessagePack.MessagePackSecurity.UntrustedData)
            .WithResolver(MessagePack.Resolvers.StandardResolverAllowPrivate.Instance);
    });
```

### MessagePack vs JSON Comparison

| Feature | JSON | MessagePack |
|---------|------|-------------|
| Format | Text | Binary |
| Message size | Larger | ~30-50% smaller |
| Serialization speed | Slower | Faster |
| Human readable | Yes | No |
| Browser DevTools | Easy to debug | Requires tools |
| Client support | All | Requires package |

---

## 13. Scaling with Redis Backplane

When running multiple server instances, a Redis backplane ensures messages reach all connected clients.

```csharp
// Install: dotnet add package Microsoft.AspNetCore.SignalR.StackExchangeRedis

builder.Services.AddSignalR()
    .AddStackExchangeRedis(builder.Configuration.GetConnectionString("Redis")!, options =>
    {
        options.Configuration.ChannelPrefix = RedisChannel.Literal("MyApp");
    });
```

### Multi-Server Architecture

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Client A  │     │   Client B  │     │   Client C  │
│  (Browser)  │     │  (Browser)  │     │  (Browser)  │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       ▼                   ▼                   ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Server 1   │    │   Server 2   │    │   Server 3   │
│  (SignalR)   │    │  (SignalR)   │    │  (SignalR)   │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                   │                   │
       └───────────┬───────┴───────────────────┘
                   │
            ┌──────▼──────┐
            │    Redis     │
            │  Backplane   │
            └─────────────┘
```

**Without Redis:** `Server 1` sending to `Clients.All` only reaches clients connected to Server 1.

**With Redis:** Messages are published to Redis, and all servers receive and forward them to their connected clients.

---

## 14. Dependency Injection in Hubs

Hubs are transient -- a new instance is created for each hub method invocation. Use constructor injection.

```csharp
public class ProjectHub : Hub<IProjectClient>
{
    private readonly IProjectService _projects;
    private readonly INotificationService _notifications;
    private readonly ILogger<ProjectHub> _logger;

    public ProjectHub(
        IProjectService projects,
        INotificationService notifications,
        ILogger<ProjectHub> logger)
    {
        _projects = projects;
        _notifications = notifications;
        _logger = logger;
    }

    public async Task CreateTask(string projectId, CreateTaskRequest request)
    {
        var task = await _projects.CreateTaskAsync(projectId, request);
        _logger.LogInformation("Task {TaskId} created in project {ProjectId}",
            task.Id, projectId);

        await Clients.Group($"project-{projectId}").TaskCreated(task);
        await _notifications.NotifyProjectMembersAsync(projectId, $"New task: {task.Title}");
    }
}
```

**Important:** Do not store state in hub instance fields between invocations. Each invocation gets a fresh hub instance.

---

## 15. Complete Production Example

```csharp
// --- Contracts ---
public interface INotificationClient
{
    Task ReceiveNotification(NotificationDto notification);
    Task NotificationCountUpdated(int count);
    Task ReceiveToast(string message, string severity);
}

public record NotificationDto(string Id, string Title, string Body, DateTime CreatedAt);

// --- Hub ---
[Authorize]
public class NotificationHub : Hub<INotificationClient>
{
    private readonly INotificationService _service;
    private readonly ILogger<NotificationHub> _logger;

    public NotificationHub(INotificationService service, ILogger<NotificationHub> logger)
    {
        _service = service;
        _logger = logger;
    }

    public async Task MarkAsRead(string notificationId)
    {
        var userId = Context.UserIdentifier
            ?? throw new HubException("User not authenticated.");

        await _service.MarkAsReadAsync(userId, notificationId);
        var unreadCount = await _service.GetUnreadCountAsync(userId);
        await Clients.Caller.NotificationCountUpdated(unreadCount);
    }

    public async Task MarkAllAsRead()
    {
        var userId = Context.UserIdentifier
            ?? throw new HubException("User not authenticated.");

        await _service.MarkAllAsReadAsync(userId);
        await Clients.Caller.NotificationCountUpdated(0);
    }

    public override async Task OnConnectedAsync()
    {
        var userId = Context.UserIdentifier;
        if (userId is not null)
        {
            await Groups.AddToGroupAsync(Context.ConnectionId, $"user-{userId}");
            var unreadCount = await _service.GetUnreadCountAsync(userId);
            await Clients.Caller.NotificationCountUpdated(unreadCount);
        }
        await base.OnConnectedAsync();
    }

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        var userId = Context.UserIdentifier;
        if (userId is not null)
        {
            await Groups.RemoveFromGroupAsync(Context.ConnectionId, $"user-{userId}");
        }
        await base.OnDisconnectedAsync(exception);
    }
}

// --- Program.cs ---
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(/* ... */);
builder.Services.AddAuthorization();

builder.Services.AddSignalR(options =>
{
    options.KeepAliveInterval = TimeSpan.FromSeconds(15);
    options.ClientTimeoutInterval = TimeSpan.FromSeconds(30);
    options.MaximumReceiveMessageSize = 64 * 1024;
    options.EnableDetailedErrors = builder.Environment.IsDevelopment();
})
.AddMessagePackProtocol();

builder.Services.AddScoped<INotificationService, NotificationService>();
builder.Services.AddHostedService<NotificationDispatchService>();

builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
    {
        policy.WithOrigins("https://myapp.com")
              .AllowAnyHeader()
              .AllowAnyMethod()
              .AllowCredentials(); // Required for SignalR
    });
});

var app = builder.Build();

app.UseCors();
app.UseAuthentication();
app.UseAuthorization();

app.MapHub<NotificationHub>("/hubs/notifications");

app.Run();
```

---

## Best Practices

1. **Always use strongly-typed hubs** -- Avoid magic strings; use `Hub<T>` with an interface.
2. **Keep hub methods thin** -- Delegate business logic to injected services.
3. **Do not store state on hub instances** -- Hubs are transient; use external stores.
4. **Use groups for topic-based messaging** -- More efficient than tracking connection IDs.
5. **Handle reconnection on the client** -- Group memberships are lost on disconnect.
6. **Use `IHubContext<T>` for out-of-hub messaging** -- Background services, controllers, etc.
7. **Set `MaximumReceiveMessageSize`** -- Protect against oversized messages.
8. **Use Redis backplane for multi-server** -- Required for load-balanced deployments.
9. **Authenticate via query string for WebSockets** -- Configure `OnMessageReceived` event.
10. **Use `CancellationToken` where possible** -- Respect client disconnections in long operations.

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|---------|
| Storing state in hub fields | State lost between invocations | Use `Context.Items`, external cache, or static `ConcurrentDictionary` |
| Not handling reconnection | Clients lose group membership | Re-join groups in `OnConnectedAsync` or client reconnect handler |
| Blocking calls in hub methods | Thread starvation | Use `async`/`await` throughout |
| Large messages without size limit | Memory exhaustion | Set `MaximumReceiveMessageSize` |
| Missing CORS `AllowCredentials` | WebSocket connection fails | Add `.AllowCredentials()` to CORS policy |
| Overloaded hub methods | Runtime error | Use unique method names (no overloads) |
| Not using Redis in multi-server | Messages lost across servers | Add `.AddStackExchangeRedis()` |
| Catching all exceptions silently | Client never receives error | Throw `HubException` with user-friendly message |
| Missing `access_token` handler for JWT | Auth fails on WebSocket upgrade | Configure `OnMessageReceived` in JWT options |
| Calling `Clients.*` after hub disposal | `ObjectDisposedException` | Use `IHubContext<T>` for async/background work |
