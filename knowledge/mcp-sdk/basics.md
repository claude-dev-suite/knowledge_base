# MCP SDK - Basics

> Official Documentation: https://modelcontextprotocol.io/docs/concepts/architecture

## Overview

The Model Context Protocol (MCP) is an open standard that allows AI applications to connect to external data sources and tools in a standardized way. It separates the concerns of providing context from the actual LLM interaction. The TypeScript SDK (`@modelcontextprotocol/sdk`) implements this protocol and runs on Node.js, Bun, and Deno.

---

## Table of Contents

1. [Architecture: Host, Client, Server](#architecture)
2. [Layers: Data and Transport](#layers)
3. [Primitives](#primitives)
4. [Installation (TypeScript SDK)](#installation)
5. [Transport Types](#transport-types)
6. [Basic Server Structure](#basic-server-structure)
7. [Lifecycle and Capability Negotiation](#lifecycle-and-capability-negotiation)
8. [JSON-RPC Protocol](#json-rpc-protocol)

---

## Architecture

MCP follows a client-server architecture with three key participants:

### MCP Host
The AI application (e.g., Claude Desktop, Claude Code, VS Code) that coordinates and manages one or multiple MCP clients. A host instantiates one MCP client per MCP server it connects to.

### MCP Client
A component inside the host that maintains a dedicated connection with a corresponding MCP server. Each client-server pair has an exclusive connection.

### MCP Server
A program that provides context to MCP clients. Servers can run **locally** (same machine, using stdio transport) or **remotely** (another machine or cloud, using Streamable HTTP transport).

```
MCP Host (AI Application)
├── MCP Client 1 ──── MCP Server A (Local, e.g. Filesystem)
├── MCP Client 2 ──── MCP Server B (Local, e.g. Database)
├── MCP Client 3 ──── MCP Server C (Remote, e.g. Sentry)
└── MCP Client 4 ──── MCP Server C (same remote server, multiple clients)
```

A single remote server (Streamable HTTP) can serve many clients concurrently. A local server (stdio) typically serves one client.

---

## Layers

MCP consists of two layers:

### Data Layer
Implements a JSON-RPC 2.0 based exchange protocol. It defines:
- **Lifecycle management**: connection initialization, capability negotiation, termination
- **Server features**: tools, resources, prompts
- **Client features**: sampling, elicitation, logging
- **Utility features**: notifications, progress tracking

### Transport Layer
Manages communication channels and authentication between clients and servers. It handles connection establishment, message framing, and secure communication.

MCP supports two transport mechanisms:
- **Stdio transport**: uses standard input/output streams for local process communication. Optimal performance with no network overhead.
- **Streamable HTTP transport**: uses HTTP POST for client-to-server messages with optional Server-Sent Events (SSE) for streaming. Supports standard HTTP authentication (bearer tokens, API keys, custom headers, OAuth).

---

## Primitives

### Server-exposed primitives
| Primitive | Description |
|-----------|-------------|
| **Tools** | Executable functions the AI can invoke (file ops, API calls, DB queries) |
| **Resources** | Data sources providing contextual information (file contents, DB records) |
| **Prompts** | Reusable templates for structuring LLM interactions |

### Client-exposed primitives
| Primitive | Description |
|-----------|-------------|
| **Sampling** | Servers request LLM completions from the client |
| **Elicitation** | Servers request additional information from the user |
| **Roots** | Filesystem boundary declarations from client to server |

Each primitive type has associated methods:
- `*/list` — discovery
- `*/get` or `*/read` — retrieval
- `tools/call` — execution (tools only)

---

## Installation

### TypeScript SDK (v2 - pre-alpha, v1 recommended for production)

> Note: The `main` branch contains v2 (pre-alpha). v1.x is recommended for production until a stable v2 releases. The examples below follow the v2 SDK structure.

#### Server package
```bash
npm install @modelcontextprotocol/server zod
# or
bun add @modelcontextprotocol/server zod
# or
deno add npm:@modelcontextprotocol/server npm:zod
```

#### Client package
```bash
npm install @modelcontextprotocol/client zod
# or
bun add @modelcontextprotocol/client zod
# or
deno add npm:@modelcontextprotocol/client npm:zod
```

#### Optional middleware packages
```bash
# Node.js HTTP (IncomingMessage/ServerResponse) Streamable HTTP transport
npm install @modelcontextprotocol/node

# Express integration (includes DNS rebinding protection)
npm install @modelcontextprotocol/express express

# Hono integration (includes DNS rebinding protection)
npm install @modelcontextprotocol/hono hono
```

Both `@modelcontextprotocol/server` and `@modelcontextprotocol/client` require `zod` v4 as a peer dependency.

### Python SDK
```bash
pip install mcp
# or with CLI tools
pip install "mcp[cli]"
# or with uv
uv add "mcp[cli]"
```

---

## Transport Types

### Stdio Transport
Used for local, process-spawned integrations (Claude Desktop, CLI tools). The host spawns the server as a child process.

**Server side:**
```typescript
import { McpServer } from '@modelcontextprotocol/server';
import { StdioServerTransport } from '@modelcontextprotocol/server';

const server = new McpServer({ name: 'my-server', version: '1.0.0' });
const transport = new StdioServerTransport();
await server.connect(transport);
```

**Client side:**
```typescript
import { Client, StdioClientTransport } from '@modelcontextprotocol/client';

const client = new Client({ name: 'my-client', version: '1.0.0' });
const transport = new StdioClientTransport({
    command: 'node',
    args: ['server.js']
});
await client.connect(transport);
```

Important: When using stdio, **never write to stdout** from the server — it will corrupt JSON-RPC messages. Use `stderr` for logging.

### Streamable HTTP Transport
Used for remote servers. Supports request/response over HTTP POST, server-to-client notifications over SSE, optional JSON-only response mode, and session management.

**Server side (stateless):**
```typescript
import { McpServer } from '@modelcontextprotocol/server';
import { NodeStreamableHTTPServerTransport } from '@modelcontextprotocol/node';
import { createMcpExpressApp } from '@modelcontextprotocol/express';

const app = createMcpExpressApp(); // includes DNS rebinding protection

app.post('/mcp', async (req, res) => {
    const server = new McpServer({ name: 'my-server', version: '1.0.0' });
    const transport = new NodeStreamableHTTPServerTransport({
        sessionIdGenerator: undefined // stateless
    });
    await server.connect(transport);
    await transport.handleRequest(req, res, req.body);
});

app.listen(3000, '127.0.0.1');
```

**Client side:**
```typescript
import { Client, StreamableHTTPClientTransport } from '@modelcontextprotocol/client';

const client = new Client({ name: 'my-client', version: '1.0.0' });
const transport = new StreamableHTTPClientTransport(new URL('http://localhost:3000/mcp'));
await client.connect(transport);
```

### SSE Transport (legacy fallback)
For supporting older servers that use Server-Sent Events:
```typescript
import { Client, SSEClientTransport, StreamableHTTPClientTransport } from '@modelcontextprotocol/client';

const baseUrl = new URL(url);

try {
    // Try modern Streamable HTTP first
    const client = new Client({ name: 'my-client', version: '1.0.0' });
    const transport = new StreamableHTTPClientTransport(baseUrl);
    await client.connect(transport);
    return { client, transport };
} catch {
    // Fall back to legacy SSE
    const client = new Client({ name: 'my-client', version: '1.0.0' });
    const transport = new SSEClientTransport(baseUrl);
    await client.connect(transport);
    return { client, transport };
}
```

---

## Basic Server Structure

Building an MCP server takes three steps:
1. Create an `McpServer` and register tools, resources, and/or prompts
2. Create a transport (Streamable HTTP for remote, stdio for local)
3. Wire the transport and call `server.connect(transport)`

```typescript
import { McpServer } from '@modelcontextprotocol/server';
import { StdioServerTransport } from '@modelcontextprotocol/server';
import { z } from 'zod';

// Step 1: Create server
const server = new McpServer({
    name: 'my-server',
    version: '1.0.0'
});

// Register a tool
server.registerTool(
    'greet',
    {
        title: 'Greeter',
        description: 'Returns a greeting message',
        inputSchema: z.object({
            name: z.string().describe('Name to greet')
        })
    },
    async ({ name }) => ({
        content: [{ type: 'text', text: `Hello, ${name}!` }]
    })
);

// Step 2 & 3: Create transport and connect
const transport = new StdioServerTransport();
await server.connect(transport);
```

### Python equivalent (FastMCP)
```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("my-server")

@mcp.tool()
async def greet(name: str) -> str:
    """Returns a greeting message.

    Args:
        name: Name to greet
    """
    return f"Hello, {name}!"

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

---

## Lifecycle and Capability Negotiation

MCP is a stateful protocol. The lifecycle begins with capability negotiation:

### Initialize Request (client → server)
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-06-18",
    "capabilities": {
      "sampling": {},
      "elicitation": {}
    },
    "clientInfo": {
      "name": "example-client",
      "version": "1.0.0"
    }
  }
}
```

### Initialize Response (server → client)
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2025-06-18",
    "capabilities": {
      "tools": { "listChanged": true },
      "resources": { "subscribe": true, "listChanged": true },
      "prompts": { "listChanged": true },
      "logging": {}
    },
    "serverInfo": {
      "name": "example-server",
      "version": "1.0.0"
    }
  }
}
```

### Initialized Notification (client → server)
```json
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized"
}
```

The initialization process serves three purposes:
1. **Protocol version negotiation**: ensures compatibility
2. **Capability discovery**: declares what features each side supports
3. **Identity exchange**: provides `clientInfo`/`serverInfo` for debugging

---

## JSON-RPC Protocol

MCP uses JSON-RPC 2.0 as its underlying protocol. Key message types:

### Request
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "method/name",
  "params": { ... }
}
```

### Response
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": { ... }
}
```

### Error Response
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32602,
    "message": "Invalid params",
    "data": { ... }
  }
}
```

### Notification (no response expected, no `id` field)
```json
{
  "jsonrpc": "2.0",
  "method": "notifications/tools/list_changed"
}
```

### Common Error Codes
| Code | Meaning |
|------|---------|
| `-32700` | Parse error |
| `-32600` | Invalid request |
| `-32601` | Method not found |
| `-32602` | Invalid params |
| `-32603` | Internal error |
| `-32002` | Resource not found |
