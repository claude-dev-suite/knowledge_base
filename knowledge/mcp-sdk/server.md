# MCP SDK - Server

> Official Documentation: https://modelcontextprotocol.io/docs/concepts/architecture
> SDK Reference: https://github.com/modelcontextprotocol/typescript-sdk/blob/main/docs/server.md

## Overview

The TypeScript SDK provides the `McpServer` class (high-level) for building MCP servers. It handles capability negotiation, request routing, and all protocol-level concerns automatically. For advanced use cases requiring manual control over every JSON-RPC message, the low-level `Server` class is also available.

---

## Table of Contents

1. [McpServer (High-Level)](#mcpserver-high-level)
2. [Server Configuration and Capabilities](#server-configuration-and-capabilities)
3. [Registering Tools](#registering-tools)
4. [Registering Resources](#registering-resources)
5. [Registering Prompts](#registering-prompts)
6. [Transport Configuration](#transport-configuration)
7. [Stdio Transport](#stdio-transport)
8. [Streamable HTTP Transport](#streamable-http-transport)
9. [Session Management (Stateful)](#session-management-stateful)
10. [JSON Response Mode (No SSE)](#json-response-mode)
11. [Logging](#logging)
12. [Server-Initiated Requests (Sampling, Elicitation)](#server-initiated-requests)
13. [DNS Rebinding Protection](#dns-rebinding-protection)
14. [Error Handling](#error-handling)
15. [Complete Example: Weather Server](#complete-example-weather-server)

---

## McpServer (High-Level)

`McpServer` is the main entry point for building MCP servers. It abstracts away JSON-RPC handling and provides a clean API for registering tools, resources, and prompts.

```typescript
import { McpServer } from '@modelcontextprotocol/server';

const server = new McpServer(
    {
        name: 'my-server',          // required: server name
        version: '1.0.0'            // required: server version
    },
    {
        // optional: server-level capability declarations
        capabilities: {
            tools: { listChanged: true },
            resources: { subscribe: true, listChanged: true },
            prompts: { listChanged: true },
            logging: {}
        }
    }
);
```

---

## Server Configuration and Capabilities

Capabilities are declared during initialization and tell clients what features the server supports.

```typescript
// Full capabilities example
const server = new McpServer(
    { name: 'full-featured-server', version: '2.0.0' },
    {
        capabilities: {
            // Tools: support list-change notifications
            tools: { listChanged: true },

            // Resources: support subscriptions and list-change notifications
            resources: {
                subscribe: true,        // clients can subscribe to individual resource changes
                listChanged: true       // server sends notifications when resource list changes
            },

            // Prompts: support list-change notifications
            prompts: { listChanged: true },

            // Logging: server can send log messages to client
            logging: {}
        }
    }
);
```

### Minimal server (no extra capabilities)
```typescript
const server = new McpServer({ name: 'simple-server', version: '1.0.0' });
```

---

## Registering Tools

```typescript
import { McpServer, CallToolResult, ResourceLink } from '@modelcontextprotocol/server';
import { z } from 'zod';

// Basic tool
server.registerTool(
    'calculate-bmi',
    {
        title: 'BMI Calculator',
        description: 'Calculate Body Mass Index',
        inputSchema: z.object({
            weightKg: z.number().describe('Weight in kilograms'),
            heightM: z.number().describe('Height in meters')
        }),
        outputSchema: z.object({ bmi: z.number() })
    },
    async ({ weightKg, heightM }) => {
        const output = { bmi: weightKg / (heightM * heightM) };
        return {
            content: [{ type: 'text', text: JSON.stringify(output) }],
            structuredContent: output
        };
    }
);

// Tool returning resource links
server.registerTool(
    'list-files',
    {
        title: 'List Files',
        description: 'Returns project files as resource links'
    },
    async (): Promise<CallToolResult> => ({
        content: [
            {
                type: 'resource_link',
                uri: 'file:///projects/readme.md',
                name: 'README',
                mimeType: 'text/markdown'
            }
        ]
    })
);

// Tool with annotations
server.registerTool(
    'read-file',
    {
        title: 'Read File',
        description: 'Read file contents',
        inputSchema: z.object({ path: z.string() }),
        annotations: {
            readOnlyHint: true,
            idempotentHint: true,
            destructiveHint: false
        }
    },
    async ({ path }) => {
        const text = await fs.readFile(path, 'utf-8');
        return { content: [{ type: 'text', text }] };
    }
);
```

---

## Registering Resources

```typescript
import { ResourceTemplate } from '@modelcontextprotocol/server';

// Static resource
server.registerResource(
    'config',
    'config://app',
    {
        title: 'Application Config',
        description: 'Current application configuration',
        mimeType: 'application/json'
    },
    async (uri) => ({
        contents: [{
            uri: uri.href,
            mimeType: 'application/json',
            text: JSON.stringify(await loadConfig(), null, 2)
        }]
    })
);

// Dynamic resource with URI template
server.registerResource(
    'user-profile',
    new ResourceTemplate('user://{userId}/profile', {
        list: async () => ({
            resources: [
                { uri: 'user://1/profile', name: 'Alice' },
                { uri: 'user://2/profile', name: 'Bob' }
            ]
        })
    }),
    {
        title: 'User Profile',
        mimeType: 'application/json'
    },
    async (uri, { userId }) => ({
        contents: [{
            uri: uri.href,
            mimeType: 'application/json',
            text: JSON.stringify(await getUser(userId))
        }]
    })
);
```

---

## Registering Prompts

```typescript
import { completable } from '@modelcontextprotocol/server';

server.registerPrompt(
    'review-code',
    {
        title: 'Code Review',
        description: 'Review code for best practices and potential issues',
        argsSchema: z.object({
            code: z.string().describe('The code to review'),
            language: completable(
                z.string().describe('Programming language'),
                (value) => ['typescript', 'javascript', 'python', 'rust', 'go']
                    .filter(l => l.startsWith(value))
            )
        })
    },
    ({ code, language }) => ({
        messages: [
            {
                role: 'user' as const,
                content: {
                    type: 'text' as const,
                    text: `Please review this ${language} code:\n\n\`\`\`${language}\n${code}\n\`\`\``
                }
            }
        ]
    })
);
```

---

## Transport Configuration

All servers follow the same three-step pattern:
1. Create `McpServer`
2. Create transport
3. Call `server.connect(transport)`

---

## Stdio Transport

Best for: local, process-spawned servers (Claude Desktop, CLI tools, VS Code extensions).

```typescript
import { McpServer } from '@modelcontextprotocol/server';
import { StdioServerTransport } from '@modelcontextprotocol/server';
import { z } from 'zod';

const server = new McpServer({ name: 'stdio-server', version: '1.0.0' });

server.registerTool(
    'hello',
    {
        description: 'Say hello',
        inputSchema: z.object({ name: z.string() })
    },
    async ({ name }) => ({
        content: [{ type: 'text', text: `Hello, ${name}!` }]
    })
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

**Important**: When using stdio, **never write to stdout** directly. Use stderr for debugging:
```typescript
// Bad: corrupts JSON-RPC messages
console.log('debug message');

// Good: write to stderr
console.error('debug message');
process.stderr.write('debug message\n');
```

---

## Streamable HTTP Transport

Best for: remote servers, multi-client scenarios, web-based integrations.

### Stateless server (simplest setup)
```typescript
import { McpServer } from '@modelcontextprotocol/server';
import { NodeStreamableHTTPServerTransport } from '@modelcontextprotocol/node';
import { createMcpExpressApp } from '@modelcontextprotocol/express';

const app = createMcpExpressApp(); // includes DNS rebinding protection

app.post('/mcp', async (req, res) => {
    const server = new McpServer({ name: 'my-server', version: '1.0.0' });

    // Register tools/resources/prompts...
    server.registerTool('ping', { description: 'Ping' }, async () => ({
        content: [{ type: 'text', text: 'pong' }]
    }));

    const transport = new NodeStreamableHTTPServerTransport({
        sessionIdGenerator: undefined // undefined = stateless
    });

    await server.connect(transport);
    await transport.handleRequest(req, res, req.body);
});

app.listen(3000, '127.0.0.1');
console.log('MCP server running at http://127.0.0.1:3000/mcp');
```

### With Hono (Web Standard runtimes: Cloudflare Workers, Deno, Bun)
```typescript
import { McpServer } from '@modelcontextprotocol/server';
import { createMcpHonoApp } from '@modelcontextprotocol/hono';

const app = createMcpHonoApp();

app.post('/mcp', async (c) => {
    const server = new McpServer({ name: 'hono-server', version: '1.0.0' });
    // ... register tools
    const transport = /* hono transport */;
    await server.connect(transport);
    return transport.handleRequest(c);
});

export default app;
```

---

## Session Management (Stateful)

Stateful sessions enable resumability and advanced features like per-session state.

```typescript
import express from 'express';
import { randomUUID } from 'crypto';
import { McpServer } from '@modelcontextprotocol/server';
import { NodeStreamableHTTPServerTransport } from '@modelcontextprotocol/node';
import { createMcpExpressApp } from '@modelcontextprotocol/express';

const app = createMcpExpressApp();

// Map of session ID → transport (for routing subsequent requests)
const sessions = new Map<string, NodeStreamableHTTPServerTransport>();

function createServer() {
    const server = new McpServer(
        { name: 'stateful-server', version: '1.0.0' },
        { capabilities: { logging: {} } }
    );
    // Register tools, resources, prompts...
    return server;
}

// Initialize session
app.post('/mcp', async (req, res) => {
    const sessionId = req.headers['mcp-session-id'] as string | undefined;

    if (sessionId && sessions.has(sessionId)) {
        // Route to existing session
        const transport = sessions.get(sessionId)!;
        await transport.handleRequest(req, res, req.body);
        return;
    }

    // New session
    const transport = new NodeStreamableHTTPServerTransport({
        sessionIdGenerator: () => randomUUID(),
        onsessioninitialized: (id) => {
            sessions.set(id, transport);
            console.log(`Session initialized: ${id}`);
        }
    });

    transport.onclose = () => {
        if (transport.sessionId) {
            sessions.delete(transport.sessionId);
            console.log(`Session closed: ${transport.sessionId}`);
        }
    };

    const server = createServer();
    await server.connect(transport);
    await transport.handleRequest(req, res, req.body);
});

// SSE endpoint for server-to-client streaming
app.get('/mcp', async (req, res) => {
    const sessionId = req.headers['mcp-session-id'] as string;
    const transport = sessions.get(sessionId);
    if (!transport) {
        res.status(404).json({ error: 'Session not found' });
        return;
    }
    await transport.handleRequest(req, res);
});

// Terminate session
app.delete('/mcp', async (req, res) => {
    const sessionId = req.headers['mcp-session-id'] as string;
    const transport = sessions.get(sessionId);
    if (transport) {
        await transport.close();
        sessions.delete(sessionId);
    }
    res.status(200).json({ message: 'Session terminated' });
});

app.listen(3000, '127.0.0.1');
```

---

## JSON Response Mode (No SSE)

If you don't need SSE streaming, use `enableJsonResponse: true`. The server returns plain JSON responses to every POST and rejects GET requests with `405 Method Not Allowed`.

```typescript
const transport = new NodeStreamableHTTPServerTransport({
    sessionIdGenerator: () => randomUUID(),
    enableJsonResponse: true  // no SSE, pure JSON responses
});
```

---

## Logging

Declare the `logging` capability and use `ctx.mcpReq.log(level, data)` inside any handler.

```typescript
const server = new McpServer(
    { name: 'my-server', version: '1.0.0' },
    { capabilities: { logging: {} } }
);

server.registerTool(
    'process-data',
    {
        description: 'Process data with detailed logging',
        inputSchema: z.object({ data: z.string() })
    },
    async ({ data }, ctx) => {
        await ctx.mcpReq.log('info', 'Starting data processing');
        await ctx.mcpReq.log('debug', { message: 'Input received', length: data.length });

        try {
            const result = await processData(data);
            await ctx.mcpReq.log('info', 'Processing complete');
            return { content: [{ type: 'text', text: result }] };
        } catch (error) {
            await ctx.mcpReq.log('error', `Processing failed: ${error}`);
            return {
                content: [{ type: 'text', text: `Error: ${error}` }],
                isError: true
            };
        }
    }
);
```

Log levels: `'debug'` | `'info'` | `'warning'` | `'error'`

---

## Server-Initiated Requests

MCP is bidirectional — servers can send requests to clients during tool execution if clients declare matching capabilities.

### Sampling (request LLM completion from client)
```typescript
server.registerTool(
    'summarize',
    {
        description: 'Summarize text using the client LLM',
        inputSchema: z.object({ text: z.string() })
    },
    async ({ text }, ctx): Promise<CallToolResult> => {
        const response = await ctx.mcpReq.requestSampling({
            messages: [
                {
                    role: 'user',
                    content: {
                        type: 'text',
                        text: `Please summarize:\n\n${text}`
                    }
                }
            ],
            maxTokens: 500,
            modelPreferences: {
                hints: [{ name: 'claude-3-sonnet' }],
                speedPriority: 0.8,
                intelligencePriority: 0.5,
                costPriority: 0.3
            },
            systemPrompt: 'You are a concise summarizer.'
        });

        const responseText = response.content.type === 'text'
            ? response.content.text
            : '[non-text response]';

        return {
            content: [
                { type: 'text', text: `Summary (${response.model}): ${responseText}` }
            ]
        };
    }
);
```

### Elicitation (request user input from client)
```typescript
server.registerTool(
    'collect-feedback',
    {
        description: 'Collect user feedback',
        inputSchema: z.object({})
    },
    async (_args, ctx): Promise<CallToolResult> => {
        const result = await ctx.mcpReq.elicitInput({
            mode: 'form',
            message: 'Please rate your experience:',
            requestedSchema: {
                type: 'object',
                properties: {
                    rating: {
                        type: 'number',
                        title: 'Rating (1-5)',
                        minimum: 1,
                        maximum: 5
                    },
                    comment: {
                        type: 'string',
                        title: 'Comment'
                    }
                },
                required: ['rating']
            }
        });

        if (result.action === 'accept') {
            const { rating, comment } = result.content as { rating: number; comment?: string };
            await saveFeedback(rating, comment);
            return {
                content: [{ type: 'text', text: `Feedback saved: ${rating}/5 stars` }]
            };
        }

        return { content: [{ type: 'text', text: 'Feedback collection cancelled.' }] };
    }
);
```

---

## DNS Rebinding Protection

All localhost MCP servers should use DNS rebinding protection to prevent cross-origin attacks.

```typescript
import { createMcpExpressApp } from '@modelcontextprotocol/express';

// Default: DNS rebinding protection auto-enabled (binds to 127.0.0.1)
const app = createMcpExpressApp();

// Also auto-enabled for localhost
const appLocal = createMcpExpressApp({ host: 'localhost' });

// Custom allowed hosts when binding to all interfaces
const appOpen = createMcpExpressApp({
    host: '0.0.0.0',
    allowedHosts: ['localhost', '127.0.0.1', 'myapp.local']
});
```

`createMcpHonoApp()` from `@modelcontextprotocol/hono` provides the same protection for Hono-based servers.

---

## Error Handling

### Tool execution errors (soft)
```typescript
return {
    content: [{ type: 'text', text: 'Operation failed: reason' }],
    isError: true
};
```

### Protocol errors (hard — throw)
```typescript
// These become JSON-RPC errors (-32602 Invalid Params, etc.)
throw new Error('Invalid argument: path cannot be empty');
```

### Global error handler
```typescript
server.onerror = (error) => {
    console.error('MCP server error:', error);
};
```

---

## Complete Example: Weather Server

```typescript
import { McpServer } from '@modelcontextprotocol/server';
import { StdioServerTransport } from '@modelcontextprotocol/server';
import { z } from 'zod';

const NWS_API_BASE = 'https://api.weather.gov';
const USER_AGENT = 'weather-mcp-server/1.0';

const server = new McpServer(
    { name: 'weather', version: '1.0.0' },
    { capabilities: { tools: { listChanged: false } } }
);

async function makeNwsRequest(url: string) {
    const res = await fetch(url, {
        headers: { 'User-Agent': USER_AGENT, 'Accept': 'application/geo+json' }
    });
    if (!res.ok) throw new Error(`HTTP ${res.status}: ${res.statusText}`);
    return res.json();
}

server.registerTool(
    'get-alerts',
    {
        title: 'Weather Alerts',
        description: 'Get active weather alerts for a US state',
        inputSchema: z.object({
            state: z.string().length(2).describe('Two-letter US state code (e.g. CA, NY)')
        })
    },
    async ({ state }) => {
        try {
            const data = await makeNwsRequest(`${NWS_API_BASE}/alerts/active/area/${state.toUpperCase()}`);
            if (!data.features?.length) {
                return { content: [{ type: 'text', text: `No active alerts for ${state}.` }] };
            }
            const alerts = data.features.map((f: any) => {
                const p = f.properties;
                return `Event: ${p.event}\nArea: ${p.areaDesc}\nSeverity: ${p.severity}`;
            }).join('\n---\n');
            return { content: [{ type: 'text', text: alerts }] };
        } catch (error) {
            return {
                content: [{ type: 'text', text: `Failed to fetch alerts: ${error}` }],
                isError: true
            };
        }
    }
);

server.registerTool(
    'get-forecast',
    {
        title: 'Weather Forecast',
        description: 'Get weather forecast for a location by coordinates',
        inputSchema: z.object({
            latitude: z.number().min(-90).max(90).describe('Latitude'),
            longitude: z.number().min(-180).max(180).describe('Longitude')
        })
    },
    async ({ latitude, longitude }) => {
        try {
            const points = await makeNwsRequest(`${NWS_API_BASE}/points/${latitude},${longitude}`);
            const forecast = await makeNwsRequest(points.properties.forecast);
            const periods = forecast.properties.periods.slice(0, 5);
            const text = periods.map((p: any) =>
                `${p.name}: ${p.temperature}°${p.temperatureUnit}, ${p.shortForecast}`
            ).join('\n');
            return { content: [{ type: 'text', text }] };
        } catch (error) {
            return {
                content: [{ type: 'text', text: `Failed to fetch forecast: ${error}` }],
                isError: true
            };
        }
    }
);

const transport = new StdioServerTransport();
await server.connect(transport);
```
