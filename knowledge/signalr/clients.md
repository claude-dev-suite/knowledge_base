# SignalR Clients

> Official Documentation: https://learn.microsoft.com/aspnet/core/signalr/javascript-client

SignalR supports multiple client platforms for connecting to hubs. This guide covers the JavaScript/TypeScript client, .NET client, connection lifecycle, authentication, and framework integration patterns.

---

## 1. JavaScript/TypeScript Client Setup

### Installation

```bash
npm install @microsoft/signalr
```

### Basic Connection

```typescript
import * as signalR from "@microsoft/signalr";

const connection = new signalR.HubConnectionBuilder()
    .withUrl("/chatHub")
    .withAutomaticReconnect()
    .configureLogging(signalR.LogLevel.Information)
    .build();

// Register handlers before starting
connection.on("ReceiveMessage", (user: string, message: string) => {
    console.log(`${user}: ${message}`);
});

// Start connection
async function start() {
    try {
        await connection.start();
        console.log("SignalR Connected.");
    } catch (err) {
        console.error("SignalR Connection Error:", err);
        setTimeout(start, 5000);
    }
}

start();
```

---

## 2. HubConnectionBuilder Options

### Full Configuration

```typescript
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/chatHub", {
        // Transport selection
        transport: signalR.HttpTransportType.WebSockets
            | signalR.HttpTransportType.ServerSentEvents,

        // Skip negotiation (WebSockets only)
        skipNegotiation: false,

        // Custom headers
        headers: { "X-Custom-Header": "value" },

        // Authentication token
        accessTokenFactory: () => getAccessToken(),

        // Timeout for HTTP requests during negotiation
        timeout: 30000,

        // Logging
        logger: signalR.LogLevel.Debug,

        // WebSocket constructor (for Node.js)
        // WebSocket: require("ws"),
    })
    .withAutomaticReconnect([0, 2000, 5000, 10000, 30000])
    .withStatefulReconnect()  // .NET 8+ stateful reconnect
    .configureLogging(signalR.LogLevel.Information)
    .build();
```

### Transport Types

| Transport | Description | Browser Support | Best For |
|-----------|-------------|-----------------|----------|
| `WebSockets` | Full-duplex, low latency | Modern browsers | Default, best performance |
| `ServerSentEvents` | Server-to-client only (uses POST for client-to-server) | All modern browsers | Fallback when WebSockets blocked |
| `LongPolling` | HTTP polling | All browsers | Last resort fallback |

**Note:** SignalR automatically negotiates the best available transport. Override only when needed.

---

## 3. Connection Lifecycle

### Connection States

```
         start()
           │
           ▼
    ┌─────────────┐
    │ Connecting   │
    └──────┬──────┘
           │ success
           ▼
    ┌─────────────┐    connection lost    ┌──────────────┐
    │  Connected   │─────────────────────▶│ Reconnecting │
    └──────┬──────┘                       └──────┬───────┘
           │                                     │ success
           │                              ┌──────┘
           │                              ▼
           │ stop()               Back to Connected
           ▼
    ┌──────────────┐   all retries fail   ┌──────────────┐
    │ Disconnected  │◀────────────────────│ Disconnected  │
    └──────────────┘                      └──────────────┘
```

### Lifecycle Events

```typescript
connection.onclose((error) => {
    console.log("Connection closed.", error?.message);
    // Connection permanently closed (either stop() called or reconnect failed)
});

connection.onreconnecting((error) => {
    console.log("Reconnecting...", error?.message);
    // Show "reconnecting" UI indicator
    updateConnectionStatus("reconnecting");
});

connection.onreconnected((connectionId) => {
    console.log("Reconnected with ID:", connectionId);
    // Hide "reconnecting" UI indicator
    updateConnectionStatus("connected");
    // Re-join groups (group membership is lost on disconnect)
    rejoinGroups();
});
```

### Connection State Checking

```typescript
// HubConnectionState enum values
// Disconnected, Connecting, Connected, Disconnecting, Reconnecting

function isConnected(): boolean {
    return connection.state === signalR.HubConnectionState.Connected;
}

function sendMessageSafe(user: string, message: string): void {
    if (connection.state === signalR.HubConnectionState.Connected) {
        connection.invoke("SendMessage", user, message);
    } else {
        console.warn("Cannot send: not connected. State:", connection.state);
        queueMessage({ user, message });
    }
}
```

---

## 4. Automatic Reconnection

### Default Reconnect Policy

```typescript
// Default: retries at 0, 2000, 10000, 30000 ms then stops
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/chatHub")
    .withAutomaticReconnect()
    .build();
```

### Custom Retry Delays

```typescript
// Custom fixed delays
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/chatHub")
    .withAutomaticReconnect([0, 1000, 2000, 5000, 10000, 30000, 60000])
    .build();
```

### Custom Reconnect Policy

```typescript
class ExponentialBackoffPolicy implements signalR.IRetryPolicy {
    private readonly maxRetries = 10;
    private readonly maxDelayMs = 60000;

    nextRetryDelayInMilliseconds(retryContext: signalR.RetryContext): number | null {
        if (retryContext.previousRetryCount >= this.maxRetries) {
            return null; // Stop retrying
        }

        // Exponential backoff with jitter
        const baseDelay = Math.min(
            1000 * Math.pow(2, retryContext.previousRetryCount),
            this.maxDelayMs
        );
        const jitter = baseDelay * 0.2 * Math.random();
        return baseDelay + jitter;
    }
}

const connection = new signalR.HubConnectionBuilder()
    .withUrl("/chatHub")
    .withAutomaticReconnect(new ExponentialBackoffPolicy())
    .build();
```

### Handling Permanent Disconnection

```typescript
connection.onclose(async (error) => {
    // This fires when reconnection has permanently failed
    console.error("Permanently disconnected:", error);
    showReconnectDialog(); // Prompt user to manually reconnect
});

function showReconnectDialog(): void {
    const shouldRetry = confirm("Connection lost. Retry?");
    if (shouldRetry) {
        startConnection();
    }
}

async function startConnection(): Promise<void> {
    try {
        await connection.start();
    } catch (err) {
        console.error("Failed to reconnect:", err);
        setTimeout(startConnection, 5000);
    }
}
```

---

## 5. Invoking Hub Methods

### invoke vs send

```typescript
// invoke: waits for server response (returns Promise)
try {
    const result = await connection.invoke<string>("GetGreeting", "World");
    console.log("Server returned:", result);
} catch (err) {
    console.error("Invoke failed:", err);
}

// send: fire-and-forget (no server response, returns Promise<void>)
try {
    await connection.send("SendMessage", "user1", "Hello!");
    // Resolves when message is sent, NOT when server processes it
} catch (err) {
    console.error("Send failed:", err);
}
```

### Receiving Messages

```typescript
// Register a handler
connection.on("ReceiveMessage", (user: string, message: string) => {
    appendMessageToChat(user, message);
});

// Register handler for complex objects
interface NotificationDto {
    id: string;
    title: string;
    body: string;
    createdAt: string;
}

connection.on("ReceiveNotification", (notification: NotificationDto) => {
    showNotification(notification);
});

// Remove a specific handler
function myHandler(user: string, message: string) {
    console.log(user, message);
}
connection.on("ReceiveMessage", myHandler);
connection.off("ReceiveMessage", myHandler);

// Remove all handlers for a method
connection.off("ReceiveMessage");
```

---

## 6. .NET Client

### Installation

```bash
dotnet add package Microsoft.AspNetCore.SignalR.Client
```

### Basic Connection

```csharp
using Microsoft.AspNetCore.SignalR.Client;

var connection = new HubConnectionBuilder()
    .WithUrl("https://localhost:5001/chatHub")
    .WithAutomaticReconnect()
    .Build();

// Register handlers
connection.On<string, string>("ReceiveMessage", (user, message) =>
{
    Console.WriteLine($"{user}: {message}");
});

// Typed handler with complex objects
connection.On<NotificationDto>("ReceiveNotification", notification =>
{
    Console.WriteLine($"[{notification.Title}] {notification.Body}");
});

// Start connection
await connection.StartAsync();

// Invoke hub methods
await connection.InvokeAsync("SendMessage", "dotnetUser", "Hello from .NET!");

// Invoke with return value
var greeting = await connection.InvokeAsync<string>("GetGreeting", "World");
Console.WriteLine(greeting);
```

### .NET Client Lifecycle Events

```csharp
connection.Closed += async (error) =>
{
    Console.WriteLine($"Connection closed: {error?.Message}");
    await Task.Delay(5000);
    await connection.StartAsync();
};

connection.Reconnecting += (error) =>
{
    Console.WriteLine($"Reconnecting: {error?.Message}");
    return Task.CompletedTask;
};

connection.Reconnected += (connectionId) =>
{
    Console.WriteLine($"Reconnected with ID: {connectionId}");
    return Task.CompletedTask;
};
```

### .NET Client Configuration

```csharp
var connection = new HubConnectionBuilder()
    .WithUrl("https://localhost:5001/chatHub", options =>
    {
        // Authentication
        options.AccessTokenProvider = () => Task.FromResult(GetAccessToken());

        // Headers
        options.Headers.Add("X-Custom-Header", "value");

        // Transport selection
        options.Transports = HttpTransportType.WebSockets
            | HttpTransportType.ServerSentEvents;

        // Skip negotiation (WebSockets only)
        options.SkipNegotiation = false;

        // Cookie-based auth
        options.Cookies = new CookieContainer();

        // Client certificate
        options.ClientCertificates = new X509CertificateCollection();

        // Proxy
        options.Proxy = new WebProxy("http://proxy:8080");
    })
    .WithAutomaticReconnect(new[] { TimeSpan.Zero, TimeSpan.FromSeconds(2),
        TimeSpan.FromSeconds(5), TimeSpan.FromSeconds(30) })
    .ConfigureLogging(logging =>
    {
        logging.SetMinimumLevel(LogLevel.Debug);
        logging.AddConsole();
    })
    .AddMessagePackProtocol()
    .Build();
```

### .NET Client as Hosted Service

```csharp
public class SignalRClientService : BackgroundService
{
    private readonly HubConnection _connection;
    private readonly ILogger<SignalRClientService> _logger;

    public SignalRClientService(IConfiguration config, ILogger<SignalRClientService> logger)
    {
        _logger = logger;
        _connection = new HubConnectionBuilder()
            .WithUrl(config["SignalR:HubUrl"]!)
            .WithAutomaticReconnect()
            .Build();

        _connection.On<string>("Notification", HandleNotification);

        _connection.Closed += async (error) =>
        {
            _logger.LogWarning(error, "Connection closed");
            await Task.Delay(5000);
            await ConnectWithRetryAsync(CancellationToken.None);
        };
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await ConnectWithRetryAsync(stoppingToken);

        // Keep the service running
        while (!stoppingToken.IsCancellationRequested)
        {
            await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken);
            _logger.LogDebug("SignalR connection state: {State}", _connection.State);
        }
    }

    private async Task ConnectWithRetryAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            try
            {
                await _connection.StartAsync(ct);
                _logger.LogInformation("SignalR connected");
                return;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to connect. Retrying in 5s...");
                await Task.Delay(5000, ct);
            }
        }
    }

    private void HandleNotification(string message)
    {
        _logger.LogInformation("Received: {Message}", message);
    }

    public override async Task StopAsync(CancellationToken cancellationToken)
    {
        await _connection.DisposeAsync();
        await base.StopAsync(cancellationToken);
    }
}
```

---

## 7. Authentication

### JavaScript Bearer Token Authentication

```typescript
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/chatHub", {
        accessTokenFactory: () => {
            // Return the token (string or Promise<string>)
            return localStorage.getItem("jwt_token") ?? "";
        }
    })
    .withAutomaticReconnect()
    .build();
```

### Token Refresh Pattern

```typescript
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/chatHub", {
        accessTokenFactory: async () => {
            let token = getStoredToken();

            if (isTokenExpired(token)) {
                token = await refreshToken();
                storeToken(token);
            }

            return token;
        }
    })
    .withAutomaticReconnect()
    .build();

function isTokenExpired(token: string): boolean {
    try {
        const payload = JSON.parse(atob(token.split(".")[1]));
        return payload.exp * 1000 < Date.now() - 30000; // 30s buffer
    } catch {
        return true;
    }
}
```

### Cookie Authentication

```typescript
// Cookies are sent automatically when using same-origin requests
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/chatHub", {
        // For cross-origin, include credentials
        withCredentials: true
    })
    .build();
```

### .NET Client Authentication

```csharp
var connection = new HubConnectionBuilder()
    .WithUrl("https://api.example.com/chatHub", options =>
    {
        options.AccessTokenProvider = async () =>
        {
            var token = await _tokenService.GetAccessTokenAsync();
            return token;
        };
    })
    .Build();
```

---

## 8. Logging Configuration

### JavaScript Logging Levels

```typescript
// Available levels: Trace, Debug, Information, Warning, Error, Critical, None
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/chatHub")
    .configureLogging(signalR.LogLevel.Debug)
    .build();

// Custom logger
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/chatHub")
    .configureLogging({
        log(logLevel: signalR.LogLevel, message: string) {
            if (logLevel >= signalR.LogLevel.Warning) {
                console.warn(`[SignalR ${signalR.LogLevel[logLevel]}]`, message);
            }
        }
    })
    .build();
```

### Logging Level Reference

| Level | Use Case |
|-------|----------|
| `Trace` | Most detailed, transport-level messages |
| `Debug` | Detailed diagnostic information |
| `Information` | General flow (connected, disconnected) |
| `Warning` | Reconnection attempts, non-critical issues |
| `Error` | Connection failures, invocation errors |
| `Critical` | Unrecoverable errors |
| `None` | Disable all logging |

---

## 9. Transport Selection

### Force Specific Transport

```typescript
// WebSockets only
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/chatHub", {
        transport: signalR.HttpTransportType.WebSockets
    })
    .build();

// Skip negotiation (only valid with WebSockets-only)
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/chatHub", {
        transport: signalR.HttpTransportType.WebSockets,
        skipNegotiation: true
    })
    .build();
```

### Transport Selection on Server

```csharp
app.MapHub<ChatHub>("/chatHub", options =>
{
    // Limit allowed transports on the server
    options.Transports =
        HttpTransportType.WebSockets |
        HttpTransportType.ServerSentEvents;
    // Exclude LongPolling for better performance
});
```

---

## 10. MessagePack Protocol (Client)

### JavaScript Client

```bash
npm install @microsoft/signalr-protocol-msgpack
```

```typescript
import * as signalR from "@microsoft/signalr";
import { MessagePackHubProtocol } from "@microsoft/signalr-protocol-msgpack";

const connection = new signalR.HubConnectionBuilder()
    .withUrl("/chatHub")
    .withHubProtocol(new MessagePackHubProtocol())
    .build();
```

### .NET Client

```csharp
// dotnet add package Microsoft.AspNetCore.SignalR.Protocols.MessagePack

var connection = new HubConnectionBuilder()
    .WithUrl("https://localhost:5001/chatHub")
    .AddMessagePackProtocol()
    .Build();
```

---

## 11. Angular Integration

### SignalR Service

```typescript
// signalr.service.ts
import { Injectable, OnDestroy } from "@angular/core";
import * as signalR from "@microsoft/signalr";
import { BehaviorSubject, Subject } from "rxjs";

export type ConnectionStatus = "disconnected" | "connecting" | "connected" | "reconnecting";

@Injectable({ providedIn: "root" })
export class SignalRService implements OnDestroy {
    private connection: signalR.HubConnection;
    private connectionStatus$ = new BehaviorSubject<ConnectionStatus>("disconnected");
    private messageReceived$ = new Subject<{ user: string; message: string }>();

    readonly status$ = this.connectionStatus$.asObservable();
    readonly messages$ = this.messageReceived$.asObservable();

    constructor() {
        this.connection = new signalR.HubConnectionBuilder()
            .withUrl("/chatHub", {
                accessTokenFactory: () => localStorage.getItem("token") ?? "",
            })
            .withAutomaticReconnect([0, 2000, 5000, 10000, 30000])
            .configureLogging(signalR.LogLevel.Information)
            .build();

        this.registerEvents();
    }

    private registerEvents(): void {
        this.connection.on("ReceiveMessage", (user: string, message: string) => {
            this.messageReceived$.next({ user, message });
        });

        this.connection.onreconnecting(() => {
            this.connectionStatus$.next("reconnecting");
        });

        this.connection.onreconnected(() => {
            this.connectionStatus$.next("connected");
        });

        this.connection.onclose(() => {
            this.connectionStatus$.next("disconnected");
        });
    }

    async start(): Promise<void> {
        if (this.connection.state === signalR.HubConnectionState.Disconnected) {
            this.connectionStatus$.next("connecting");
            try {
                await this.connection.start();
                this.connectionStatus$.next("connected");
            } catch (err) {
                this.connectionStatus$.next("disconnected");
                throw err;
            }
        }
    }

    async stop(): Promise<void> {
        await this.connection.stop();
    }

    async sendMessage(user: string, message: string): Promise<void> {
        await this.connection.invoke("SendMessage", user, message);
    }

    ngOnDestroy(): void {
        this.connection.stop();
    }
}
```

### Angular Component Usage

```typescript
// chat.component.ts
import { Component, OnInit, OnDestroy } from "@angular/core";
import { SignalRService, ConnectionStatus } from "./signalr.service";
import { Subscription } from "rxjs";

@Component({
    selector: "app-chat",
    template: `
        <div class="status" [class]="connectionStatus">{{ connectionStatus }}</div>
        <div class="messages">
            @for (msg of messages; track msg) {
                <div class="message"><strong>{{ msg.user }}:</strong> {{ msg.message }}</div>
            }
        </div>
        <input [(ngModel)]="newMessage" (keyup.enter)="send()" placeholder="Type a message" />
        <button (click)="send()" [disabled]="connectionStatus !== 'connected'">Send</button>
    `,
})
export class ChatComponent implements OnInit, OnDestroy {
    messages: { user: string; message: string }[] = [];
    newMessage = "";
    connectionStatus: ConnectionStatus = "disconnected";
    private subscriptions = new Subscription();

    constructor(private signalR: SignalRService) {}

    async ngOnInit(): Promise<void> {
        this.subscriptions.add(
            this.signalR.status$.subscribe((status) => (this.connectionStatus = status)),
        );
        this.subscriptions.add(
            this.signalR.messages$.subscribe((msg) => this.messages.push(msg)),
        );
        await this.signalR.start();
    }

    async send(): Promise<void> {
        if (this.newMessage.trim()) {
            await this.signalR.sendMessage("me", this.newMessage);
            this.newMessage = "";
        }
    }

    ngOnDestroy(): void {
        this.subscriptions.unsubscribe();
        this.signalR.stop();
    }
}
```

---

## 12. React Integration

### Custom Hook

```typescript
// useSignalR.ts
import { useEffect, useRef, useState, useCallback } from "react";
import * as signalR from "@microsoft/signalr";

type ConnectionStatus = "disconnected" | "connecting" | "connected" | "reconnecting";

interface UseSignalROptions {
    url: string;
    accessTokenFactory?: () => string | Promise<string>;
    autoStart?: boolean;
}

export function useSignalR({ url, accessTokenFactory, autoStart = true }: UseSignalROptions) {
    const connectionRef = useRef<signalR.HubConnection | null>(null);
    const [status, setStatus] = useState<ConnectionStatus>("disconnected");

    useEffect(() => {
        const connection = new signalR.HubConnectionBuilder()
            .withUrl(url, {
                accessTokenFactory: accessTokenFactory ?? (() => ""),
            })
            .withAutomaticReconnect([0, 2000, 5000, 10000, 30000])
            .configureLogging(signalR.LogLevel.Information)
            .build();

        connection.onreconnecting(() => setStatus("reconnecting"));
        connection.onreconnected(() => setStatus("connected"));
        connection.onclose(() => setStatus("disconnected"));

        connectionRef.current = connection;

        if (autoStart) {
            setStatus("connecting");
            connection.start().then(
                () => setStatus("connected"),
                () => setStatus("disconnected"),
            );
        }

        return () => {
            connection.stop();
        };
    }, [url]);

    const invoke = useCallback(
        async <T = void>(methodName: string, ...args: unknown[]): Promise<T> => {
            if (!connectionRef.current) throw new Error("No connection");
            return connectionRef.current.invoke<T>(methodName, ...args);
        },
        [],
    );

    const on = useCallback((methodName: string, handler: (...args: any[]) => void) => {
        connectionRef.current?.on(methodName, handler);
        return () => connectionRef.current?.off(methodName, handler);
    }, []);

    const send = useCallback(async (methodName: string, ...args: unknown[]): Promise<void> => {
        if (!connectionRef.current) throw new Error("No connection");
        return connectionRef.current.send(methodName, ...args);
    }, []);

    return { connection: connectionRef.current, status, invoke, on, send };
}
```

### React Component Usage

```tsx
// Chat.tsx
import { useState, useEffect, useRef } from "react";
import { useSignalR } from "./useSignalR";

interface ChatMessage {
    user: string;
    message: string;
    timestamp: Date;
}

export function Chat() {
    const { status, invoke, on } = useSignalR({
        url: "/chatHub",
        accessTokenFactory: () => localStorage.getItem("token") ?? "",
    });
    const [messages, setMessages] = useState<ChatMessage[]>([]);
    const [input, setInput] = useState("");
    const messagesEndRef = useRef<HTMLDivElement>(null);

    useEffect(() => {
        const unsubscribe = on("ReceiveMessage", (user: string, message: string) => {
            setMessages((prev) => [...prev, { user, message, timestamp: new Date() }]);
        });
        return unsubscribe;
    }, [on]);

    useEffect(() => {
        messagesEndRef.current?.scrollIntoView({ behavior: "smooth" });
    }, [messages]);

    const sendMessage = async () => {
        if (input.trim() && status === "connected") {
            await invoke("SendMessage", "me", input);
            setInput("");
        }
    };

    return (
        <div className="chat-container">
            <div className={`status-badge ${status}`}>{status}</div>
            <div className="messages">
                {messages.map((msg, i) => (
                    <div key={i} className="message">
                        <strong>{msg.user}:</strong> {msg.message}
                    </div>
                ))}
                <div ref={messagesEndRef} />
            </div>
            <div className="input-area">
                <input
                    value={input}
                    onChange={(e) => setInput(e.target.value)}
                    onKeyDown={(e) => e.key === "Enter" && sendMessage()}
                    placeholder="Type a message..."
                    disabled={status !== "connected"}
                />
                <button onClick={sendMessage} disabled={status !== "connected"}>
                    Send
                </button>
            </div>
        </div>
    );
}
```

---

## 13. Connection Error Handling

### Comprehensive Error Handling Pattern

```typescript
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/chatHub")
    .withAutomaticReconnect()
    .build();

// Handle invocation errors
async function safeSend(method: string, ...args: unknown[]): Promise<void> {
    try {
        await connection.invoke(method, ...args);
    } catch (err) {
        if (err instanceof Error) {
            if (err.message.includes("not in the 'Connected' State")) {
                console.warn("Cannot send: not connected");
                // Queue message for retry after reconnect
            } else if (err.message.includes("Server returned an error")) {
                // HubException from server
                console.error("Server error:", err.message);
            } else {
                console.error("Unexpected error:", err);
            }
        }
    }
}

// Handle connection start errors
async function startWithRetry(maxRetries: number = 5): Promise<void> {
    for (let attempt = 0; attempt < maxRetries; attempt++) {
        try {
            await connection.start();
            console.log("Connected");
            return;
        } catch (err) {
            console.error(`Connection attempt ${attempt + 1} failed:`, err);
            if (attempt < maxRetries - 1) {
                const delay = Math.min(1000 * Math.pow(2, attempt), 30000);
                await new Promise((resolve) => setTimeout(resolve, delay));
            }
        }
    }
    throw new Error("Failed to connect after all retries");
}
```

---

## 14. Stateful Reconnect (.NET 8+)

Stateful reconnect allows the server to buffer messages during brief disconnections, preventing message loss.

### Server Configuration

```csharp
app.MapHub<ChatHub>("/chatHub", options =>
{
    options.AllowStatefulReconnects = true;
});

// Configure buffer size
builder.Services.AddSignalR(options =>
{
    options.StatefulReconnectBufferSize = 100_000; // bytes, default is 100KB
});
```

### JavaScript Client

```typescript
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/chatHub")
    .withStatefulReconnect()
    .build();
```

### .NET Client

```csharp
var connection = new HubConnectionBuilder()
    .WithUrl("https://localhost:5001/chatHub")
    .WithStatefulReconnect()
    .Build();
```

**How it works:**
- Server buffers outgoing messages in memory during disconnection
- Client reconnects and replays buffered messages
- Buffer is limited by `StatefulReconnectBufferSize`
- Falls back to regular reconnect if buffer is exceeded

---

## Best Practices

1. **Always register handlers before `start()`** -- Messages received before handlers are registered are lost.
2. **Use `withAutomaticReconnect()`** -- Handles transient network issues automatically.
3. **Implement `onclose` fallback** -- For when all automatic retries fail.
4. **Re-join groups on reconnect** -- Group membership is not preserved across disconnections.
5. **Use typed handlers** -- Define interfaces for message payloads.
6. **Implement connection status UI** -- Show users when the connection is degraded.
7. **Handle token refresh** -- Tokens can expire during long-lived connections.
8. **Use `invoke` for critical calls** -- Returns errors; `send` is fire-and-forget.
9. **Dispose connections on component unmount** -- Prevent memory leaks.
10. **Use `withStatefulReconnect()` for .NET 8+** -- Prevents message loss during brief disconnections.

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|---------|
| Registering handlers after `start()` | Missed messages during race condition | Register all `on()` handlers before calling `start()` |
| Not handling reconnection | App silently stops receiving messages | Use `withAutomaticReconnect()` + `onclose` fallback |
| Forgotten group re-join | Client stops receiving group messages after reconnect | Re-join groups in `onreconnected` handler |
| Token expiration | 401 errors after hours of connection | Refresh token in `accessTokenFactory` |
| Missing CORS credentials | Cross-origin WebSocket fails | Set `withCredentials: true` + server `.AllowCredentials()` |
| Not disposing connections | Memory leaks in SPA frameworks | Stop connection in cleanup/destroy lifecycle |
| `skipNegotiation` with multiple transports | Runtime error | Only use `skipNegotiation` with `WebSockets` transport alone |
| Blocking the UI thread | Frozen UI on message flood | Process messages asynchronously, debounce UI updates |
| Large payloads over SignalR | Performance issues, dropped messages | Use streaming for large data; keep messages small |
| Ignoring `invoke` errors | Silent failures | Always `try/catch` around `invoke` calls |
