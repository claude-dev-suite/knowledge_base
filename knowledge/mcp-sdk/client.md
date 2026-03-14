# MCP SDK - Client

> Official Documentation: https://modelcontextprotocol.io/docs/concepts/architecture
> SDK Reference: https://github.com/modelcontextprotocol/typescript-sdk/blob/main/docs/client.md

## Overview

The `Client` class from `@modelcontextprotocol/client` is the primary API for connecting to MCP servers, discovering their capabilities (tools, resources, prompts), and invoking them. MCP is bidirectional — servers can also send requests back to the client (sampling, elicitation, roots), which requires the client to declare matching capabilities and register handlers.

---

## Table of Contents

1. [Imports](#imports)
2. [Creating a Client](#creating-a-client)
3. [Connecting to a Server](#connecting-to-a-server)
4. [Disconnecting](#disconnecting)
5. [Server Instructions](#server-instructions)
6. [Calling Tools](#calling-tools)
7. [Reading Resources](#reading-resources)
8. [Getting Prompts](#getting-prompts)
9. [Completions](#completions)
10. [Notifications and List-Change Tracking](#notifications-and-list-change-tracking)
11. [Handling Server-Initiated Requests (Sampling, Elicitation, Roots)](#handling-server-initiated-requests)
12. [Authentication (OAuth)](#authentication)
13. [Error Handling](#error-handling)
14. [Timeouts](#timeouts)
15. [Client Middleware](#client-middleware)
16. [Resumption Tokens](#resumption-tokens)
17. [Complete Example](#complete-example)

---

## Imports

```typescript
import type { Prompt, Resource, Tool } from '@modelcontextprotocol/client';
import {
    applyMiddlewares,
    Client,
    ClientCredentialsProvider,
    createMiddleware,
    PrivateKeyJwtProvider,
    ProtocolError,
    SdkError,
    SdkErrorCode,
    SSEClientTransport,
    StdioClientTransport,
    StreamableHTTPClientTransport
} from '@modelcontextprotocol/client';
```

---

## Creating a Client

```typescript
// Minimal client
const client = new Client({ name: 'my-client', version: '1.0.0' });

// Client with declared capabilities (for server-initiated requests)
const client = new Client(
    { name: 'my-client', version: '1.0.0' },
    {
        capabilities: {
            sampling: {},               // can handle sampling/createMessage requests
            elicitation: { form: {} },  // can handle elicitation/create requests
            roots: { listChanged: true } // exposes filesystem roots to server
        }
    }
);

// Client with automatic list-change tracking
const client = new Client(
    { name: 'my-client', version: '1.0.0' },
    {
        listChanged: {
            tools: {
                onChanged: (error, tools) => {
                    if (error) {
                        console.error('Failed to refresh tools:', error);
                        return;
                    }
                    console.log('Tools updated:', tools?.map(t => t.name));
                }
            },
            prompts: {
                onChanged: (error, prompts) => console.log('Prompts updated:', prompts)
            },
            resources: {
                onChanged: (error, resources) => console.log('Resources updated:', resources)
            }
        }
    }
);
```

---

## Connecting to a Server

### Streamable HTTP (remote server)
```typescript
const client = new Client({ name: 'my-client', version: '1.0.0' });

const transport = new StreamableHTTPClientTransport(
    new URL('http://localhost:3000/mcp')
);

await client.connect(transport);
```

### stdio (local process)
```typescript
const client = new Client({ name: 'my-client', version: '1.0.0' });

const transport = new StdioClientTransport({
    command: 'node',
    args: ['./path/to/server.js'],
    // Optional: environment variables for the spawned process
    env: { NODE_ENV: 'production' }
});

await client.connect(transport);
```

### SSE fallback for legacy servers
```typescript
async function connectWithFallback(url: string) {
    const baseUrl = new URL(url);

    try {
        // Try modern Streamable HTTP first
        const client = new Client({ name: 'my-client', version: '1.0.0' });
        const transport = new StreamableHTTPClientTransport(baseUrl);
        await client.connect(transport);
        console.log('Connected via Streamable HTTP');
        return { client, transport };
    } catch {
        // Fall back to legacy SSE transport
        const client = new Client({ name: 'my-client', version: '1.0.0' });
        const transport = new SSEClientTransport(baseUrl);
        await client.connect(transport);
        console.log('Connected via SSE (legacy)');
        return { client, transport };
    }
}
```

---

## Disconnecting

```typescript
// For Streamable HTTP: terminate the server-side session first
await transport.terminateSession(); // sends DELETE to server
await client.close();

// For stdio: client.close() handles graceful process shutdown
// (closes stdin → SIGTERM → SIGKILL if needed)
await client.close();
```

---

## Server Instructions

After connecting, retrieve the server's instructions to include in the system prompt:

```typescript
const instructions = client.getInstructions();

const systemPrompt = ['You are a helpful assistant.', instructions]
    .filter(Boolean)
    .join('\n\n');

console.log(systemPrompt);
```

---

## Calling Tools

### List and call tools
```typescript
// List all tools (with pagination)
const allTools: Tool[] = [];
let toolCursor: string | undefined;

do {
    const { tools, nextCursor } = await client.listTools({ cursor: toolCursor });
    allTools.push(...tools);
    toolCursor = nextCursor;
} while (toolCursor);

console.log('Available tools:', allTools.map(t => t.name));

// Call a tool
const result = await client.callTool({
    name: 'calculate-bmi',
    arguments: { weightKg: 70, heightM: 1.75 }
});

console.log(result.content);
```

### Access structured output
```typescript
const result = await client.callTool({
    name: 'get-weather',
    arguments: { location: 'New York' }
});

// Unstructured content (for LLM)
console.log(result.content);

// Structured content (for programmatic use)
if (result.structuredContent) {
    const weather = result.structuredContent as { temperature: number; conditions: string };
    console.log(`Temperature: ${weather.temperature}°C`);
}
```

### Track progress on long-running tools
```typescript
const result = await client.callTool(
    { name: 'long-operation', arguments: { dataSize: 10000 } },
    {
        onprogress: ({ progress, total }: { progress: number; total?: number }) => {
            const pct = total ? Math.round((progress / total) * 100) : progress;
            console.log(`Progress: ${pct}%`);
        },
        resetTimeoutOnProgress: true,   // keep alive while server is actively reporting
        maxTotalTimeout: 600_000        // absolute cap: 10 minutes
    }
);
```

---

## Reading Resources

### List and read resources
```typescript
// List all resources (with pagination)
const allResources: Resource[] = [];
let resourceCursor: string | undefined;

do {
    const { resources, nextCursor } = await client.listResources({ cursor: resourceCursor });
    allResources.push(...resources);
    resourceCursor = nextCursor;
} while (resourceCursor);

console.log('Available resources:', allResources.map(r => r.name));

// Read a specific resource
const { contents } = await client.readResource({ uri: 'config://app' });
for (const item of contents) {
    if ('text' in item) {
        console.log(item.text);
    } else if ('blob' in item) {
        const binary = Buffer.from(item.blob, 'base64');
        console.log(`Binary data: ${binary.length} bytes`);
    }
}
```

### List resource templates
```typescript
const { resourceTemplates } = await client.listResourceTemplates();
console.log('Templates:', resourceTemplates.map(t => t.uriTemplate));
```

### Subscribe to resource changes
```typescript
// Subscribe
await client.subscribeResource({ uri: 'config://app' });

// Handle change notifications
client.setNotificationHandler('notifications/resources/updated', async (notification) => {
    if (notification.params.uri === 'config://app') {
        const { contents } = await client.readResource({ uri: 'config://app' });
        console.log('Config updated:', contents);
        applyNewConfig(contents);
    }
});

// Unsubscribe when done
await client.unsubscribeResource({ uri: 'config://app' });
```

---

## Getting Prompts

```typescript
// List all prompts (with pagination)
const allPrompts: Prompt[] = [];
let promptCursor: string | undefined;

do {
    const { prompts, nextCursor } = await client.listPrompts({ cursor: promptCursor });
    allPrompts.push(...prompts);
    promptCursor = nextCursor;
} while (promptCursor);

console.log('Available prompts:', allPrompts.map(p => p.name));

// Get a specific prompt with arguments
const { messages, description } = await client.getPrompt({
    name: 'review-code',
    arguments: {
        code: 'const x = 1;\nconsole.log(x)',
        language: 'javascript'
    }
});

console.log(`Prompt description: ${description}`);
console.log('Messages:', messages);

// Use the messages in an LLM call
const llmResponse = await callLLM({ messages });
```

---

## Completions

Request autocompletion suggestions for prompt or resource arguments:

```typescript
// Completions for a prompt argument
const { completion } = await client.complete({
    ref: {
        type: 'ref/prompt',
        name: 'review-code'
    },
    argument: {
        name: 'language',
        value: 'type'  // partial value typed by user
    }
});
console.log('Suggestions:', completion.values); // e.g. ['typescript']

// Completions for a resource template parameter
const { completion: resourceCompletion } = await client.complete({
    ref: {
        type: 'ref/resource',
        uri: 'user://{userId}/profile'
    },
    argument: {
        name: 'userId',
        value: '1'
    }
});
console.log('User IDs:', resourceCompletion.values);
```

---

## Notifications and List-Change Tracking

### Automatic tracking (recommended)
Configure `listChanged` on the `Client` constructor for auto-refresh with debouncing:

```typescript
const client = new Client(
    { name: 'my-client', version: '1.0.0' },
    {
        listChanged: {
            tools: {
                onChanged: (error, tools) => {
                    if (error) {
                        console.error('Failed to refresh tools:', error);
                        return;
                    }
                    updateAvailableTools(tools);
                }
            },
            prompts: {
                onChanged: (error, prompts) => {
                    if (!error) updateAvailablePrompts(prompts);
                }
            }
        }
    }
);
```

### Manual notification handlers
For full control or notification types not covered by `listChanged`:

```typescript
// Log messages from the server
client.setNotificationHandler('notifications/message', (notification) => {
    const { level, data } = notification.params;
    console.log(`[Server ${level}]`, data);
});

// Resource list changed — re-fetch
client.setNotificationHandler('notifications/resources/list_changed', async () => {
    const { resources } = await client.listResources();
    console.log('Resources changed:', resources.length);
});

// Tool list changed
client.setNotificationHandler('notifications/tools/list_changed', async () => {
    const { tools } = await client.listTools();
    updateToolRegistry(tools);
});
```

> **Warning**: `listChanged` and `setNotificationHandler` are mutually exclusive per notification type — using both for the same notification will cause the manual handler to be overwritten.

### Set log level
```typescript
// Only receive warning and error messages from server
await client.setLoggingLevel('warning');
```

---

## Handling Server-Initiated Requests

When servers need to communicate back to clients during tool execution, they send requests. Clients must declare matching capabilities and register request handlers.

### Sampling (server requests LLM completion)
```typescript
const client = new Client(
    { name: 'my-client', version: '1.0.0' },
    { capabilities: { sampling: {} } }
);

client.setRequestHandler('sampling/createMessage', async (request) => {
    const { messages, modelPreferences, systemPrompt, maxTokens } = request.params;

    // Show request to user for approval (human-in-the-loop)
    const approved = await showSamplingApprovalDialog(messages);
    if (!approved) {
        throw new Error('User rejected sampling request');
    }

    // Call your LLM (e.g., Anthropic API, OpenAI, etc.)
    const response = await callYourLLM({
        messages,
        systemPrompt,
        maxTokens,
        model: selectModel(modelPreferences)
    });

    return {
        model: response.model,
        role: 'assistant' as const,
        content: {
            type: 'text' as const,
            text: response.text
        },
        stopReason: 'endTurn'
    };
});
```

### Model preferences in sampling
```typescript
// Server sends model preferences like:
{
    hints: [
        { name: 'claude-3-sonnet' },  // prefer Sonnet-class models
        { name: 'claude' }             // fall back to any Claude
    ],
    costPriority: 0.3,           // cost is less important
    speedPriority: 0.8,          // speed is very important
    intelligencePriority: 0.5    // moderate capability needs
}

// Client selects model based on priorities and available models
function selectModel(prefs: ModelPreferences): string {
    // Map hints to available models
    for (const hint of prefs?.hints ?? []) {
        if (hint.name && availableModels.some(m => m.includes(hint.name!))) {
            return availableModels.find(m => m.includes(hint.name!))!;
        }
    }
    return defaultModel;
}
```

### Elicitation (server requests user input)
```typescript
const client = new Client(
    { name: 'my-client', version: '1.0.0' },
    { capabilities: { elicitation: { form: {} } } }
);

client.setRequestHandler('elicitation/create', async (request) => {
    const { message, mode, requestedSchema } = request.params;

    if (mode === 'form') {
        // Present form to user
        console.log('Server asks:', message);
        console.log('Schema:', requestedSchema);

        // Example: collect user input
        const userInput = await showFormDialog(message, requestedSchema);

        if (userInput === null) {
            return { action: 'decline' };
        }

        return { action: 'accept', content: userInput };
    }

    if (mode === 'url') {
        // Open URL in browser for sensitive data (OAuth, payments)
        await openBrowser(request.params.url);
        return { action: 'decline' }; // result comes via the URL flow
    }

    return { action: 'decline' };
});
```

### Roots (expose filesystem boundaries to server)
```typescript
const client = new Client(
    { name: 'my-client', version: '1.0.0' },
    { capabilities: { roots: { listChanged: true } } }
);

// Handle roots/list requests from server
client.setRequestHandler('roots/list', async () => ({
    roots: [
        { uri: 'file:///home/user/projects/my-app', name: 'My App' },
        { uri: 'file:///home/user/data', name: 'Data Directory' }
    ]
}));

// Notify server when roots change
async function addNewRoot(uri: string, name: string) {
    roots.push({ uri, name });
    await client.sendRootsListChanged();
}
```

---

## Authentication

### Client credentials (service-to-service)
```typescript
const authProvider = new ClientCredentialsProvider({
    clientId: 'my-service-id',
    clientSecret: process.env.CLIENT_SECRET!
});

const transport = new StreamableHTTPClientTransport(
    new URL('https://api.example.com/mcp'),
    { authProvider }
);

const client = new Client({ name: 'my-client', version: '1.0.0' });
await client.connect(transport);
```

### Private key JWT
```typescript
const authProvider = new PrivateKeyJwtProvider({
    clientId: 'my-service',
    privateKey: pemEncodedPrivateKey,
    algorithm: 'RS256'
});

const transport = new StreamableHTTPClientTransport(
    new URL('https://api.example.com/mcp'),
    { authProvider }
);
```

### Full OAuth (user-facing applications)
```typescript
import { OAuthClientProvider, UnauthorizedError } from '@modelcontextprotocol/client';

// Implement OAuthClientProvider interface
class MyOAuthProvider implements OAuthClientProvider {
    // Implement: redirectUrl, clientMetadata, tokens, saveTokens, redirectToAuthorization, etc.
}

const authProvider = new MyOAuthProvider();
const transport = new StreamableHTTPClientTransport(url, { authProvider });

try {
    await client.connect(transport);
} catch (error) {
    if (error instanceof UnauthorizedError) {
        // Complete browser OAuth flow, then:
        await transport.finishAuth(authorizationCode);
        await client.connect(transport); // reconnect
    }
}
```

---

## Error Handling

### Tool errors vs protocol errors
```typescript
try {
    const result = await client.callTool({
        name: 'fetch-data',
        arguments: { url: 'https://example.com' }
    });

    // Soft error: tool ran but reported a problem
    if (result.isError) {
        console.error('Tool reported error:', result.content);
        return;
    }

    console.log('Success:', result.content);

} catch (error) {
    // Protocol error: request itself failed
    if (error instanceof ProtocolError) {
        // JSON-RPC errors from server (method not found, invalid params, internal error)
        console.error(`Protocol error ${error.code}: ${error.message}`);
    } else if (error instanceof SdkError) {
        // Local SDK errors
        switch (error.code) {
            case SdkErrorCode.RequestTimeout:
                console.error('Request timed out');
                break;
            case SdkErrorCode.ConnectionClosed:
                console.error('Connection was closed');
                break;
            case SdkErrorCode.CapabilityNotSupported:
                console.error('Server does not support this capability');
                break;
            default:
                console.error(`SDK error [${error.code}]: ${error.message}`);
        }
    } else {
        throw error; // unexpected error
    }
}
```

### Connection lifecycle errors
```typescript
// Out-of-band transport errors (SSE disconnects, parse errors)
client.onerror = (error) => {
    console.error('Transport error:', error.message);
    // Optionally: trigger reconnection logic
};

// Connection closed (pending requests are rejected with CONNECTION_CLOSED)
client.onclose = () => {
    console.log('MCP connection closed');
    // Optionally: reconnect or notify the application
};
```

---

## Timeouts

All requests have a 60-second default timeout. On timeout, the SDK sends a cancellation notification to the server.

```typescript
// Override timeout for a specific request
const result = await client.callTool(
    { name: 'slow-analysis', arguments: { dataset: 'large' } },
    { timeout: 300_000 } // 5 minutes
);

// Handle timeout
try {
    await client.callTool({ name: 'task', arguments: {} }, { timeout: 10_000 });
} catch (error) {
    if (error instanceof SdkError && error.code === SdkErrorCode.RequestTimeout) {
        console.error('Request timed out after 10 seconds');
    }
}
```

---

## Client Middleware

Use `createMiddleware()` and `applyMiddlewares()` to compose fetch middleware pipelines for adding headers, retries, or logging.

```typescript
// Auth header middleware
const authMiddleware = createMiddleware(async (next, input, init) => {
    const headers = new Headers(init?.headers);
    headers.set('Authorization', `Bearer ${getToken()}`);
    return next(input, { ...init, headers });
});

// Logging middleware
const loggingMiddleware = createMiddleware(async (next, input, init) => {
    console.log('MCP request:', input);
    const response = await next(input, init);
    console.log('MCP response status:', response.status);
    return response;
});

// Retry middleware
const retryMiddleware = createMiddleware(async (next, input, init) => {
    for (let attempt = 0; attempt < 3; attempt++) {
        try {
            return await next(input, init);
        } catch (error) {
            if (attempt === 2) throw error;
            await new Promise(r => setTimeout(r, 1000 * (attempt + 1)));
        }
    }
    throw new Error('All retries failed');
});

const transport = new StreamableHTTPClientTransport(
    new URL('http://localhost:3000/mcp'),
    {
        fetch: applyMiddlewares(authMiddleware, loggingMiddleware, retryMiddleware)(fetch)
    }
);
```

---

## Resumption Tokens

For SSE-based streaming, track resumption tokens to resume from a disconnection point:

```typescript
let lastToken: string | undefined;

// Load token from persistent storage if available
lastToken = await loadFromStorage('mcp-resumption-token');

const result = await client.request(
    {
        method: 'tools/call',
        params: { name: 'long-running-task', arguments: {} }
    },
    {
        resumptionToken: lastToken,  // resume from where we left off
        onresumptiontoken: async (token: string) => {
            lastToken = token;
            await saveToStorage('mcp-resumption-token', token);
        }
    }
);
```

---

## Complete Example

```typescript
import {
    Client,
    StdioClientTransport,
    StreamableHTTPClientTransport,
    ProtocolError,
    SdkError,
    SdkErrorCode
} from '@modelcontextprotocol/client';

async function main() {
    // Connect to a local server via stdio
    const client = new Client(
        { name: 'example-client', version: '1.0.0' },
        {
            capabilities: { sampling: {} },
            listChanged: {
                tools: {
                    onChanged: (err, tools) => {
                        if (!err) console.log('Tools updated:', tools?.map(t => t.name));
                    }
                }
            }
        }
    );

    const transport = new StdioClientTransport({
        command: 'node',
        args: ['./my-mcp-server.js']
    });

    client.onerror = (err) => console.error('Transport error:', err);
    client.onclose = () => console.log('Disconnected from server');

    // Handle sampling requests from the server
    client.setRequestHandler('sampling/createMessage', async (req) => {
        // Forward to your LLM
        const response = await callAnthropicAPI(req.params);
        return {
            model: 'claude-3-sonnet-20240307',
            role: 'assistant' as const,
            content: { type: 'text' as const, text: response.text }
        };
    });

    await client.connect(transport);
    console.log('Connected! Instructions:', client.getInstructions());

    // Discover capabilities
    const { tools } = await client.listTools();
    console.log('Tools:', tools.map(t => `${t.name}: ${t.description}`));

    const { resources } = await client.listResources();
    console.log('Resources:', resources.map(r => r.uri));

    const { prompts } = await client.listPrompts();
    console.log('Prompts:', prompts.map(p => p.name));

    // Use a tool
    try {
        const result = await client.callTool({
            name: 'get-weather',
            arguments: { location: 'San Francisco' }
        });

        if (result.isError) {
            console.error('Tool error:', result.content);
        } else {
            console.log('Weather:', result.content);
        }
    } catch (error) {
        if (error instanceof ProtocolError) {
            console.error(`Protocol error ${error.code}: ${error.message}`);
        } else if (error instanceof SdkError) {
            console.error(`SDK error [${error.code}]: ${error.message}`);
        }
    }

    // Read a resource
    const { contents } = await client.readResource({ uri: 'config://app' });
    console.log('Config:', contents);

    // Get a prompt
    const { messages } = await client.getPrompt({
        name: 'review-code',
        arguments: { code: 'const x = 1;', language: 'typescript' }
    });
    console.log('Prompt messages:', messages);

    // Disconnect
    await client.close();
}

main().catch(console.error);
```
