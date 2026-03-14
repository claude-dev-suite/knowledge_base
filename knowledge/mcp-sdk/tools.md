# MCP SDK - Tools

> Official Documentation: https://modelcontextprotocol.io/docs/concepts/tools

## Overview

Tools are executable functions that AI applications can invoke to perform actions — querying databases, calling APIs, performing computations, or interacting with the file system. Tools are **model-controlled**: the language model discovers and invokes them automatically based on context. Each tool is uniquely identified by a name and includes metadata describing its schema.

---

## Table of Contents

1. [Tool Definition and Structure](#tool-definition-and-structure)
2. [Registering Tools with McpServer](#registering-tools-with-mcpserver)
3. [Zod Schema for Input Validation](#zod-schema-for-input-validation)
4. [Output Schema (Structured Content)](#output-schema-structured-content)
5. [Tool Result Content Types](#tool-result-content-types)
6. [Tool Annotations](#tool-annotations)
7. [Error Handling in Tools](#error-handling-in-tools)
8. [Logging inside Tools](#logging-inside-tools)
9. [Sampling inside Tools](#sampling-inside-tools)
10. [Protocol Messages (ListTools / CallTool)](#protocol-messages)
11. [Best Practices for Naming and Design](#best-practices)

---

## Tool Definition and Structure

A tool definition includes:
- `name`: Unique identifier within the server's namespace
- `title`: Optional human-readable display name
- `description`: Human-readable description of what the tool does and when to use it
- `inputSchema`: JSON Schema defining expected parameters
- `outputSchema`: Optional JSON Schema for structured output validation
- `annotations`: Optional hints about tool behavior

---

## Registering Tools with McpServer

Use `server.registerTool()` to register tools on an `McpServer` instance.

### Basic tool registration
```typescript
import { McpServer } from '@modelcontextprotocol/server';
import { StdioServerTransport } from '@modelcontextprotocol/server';
import { z } from 'zod';

const server = new McpServer({ name: 'my-server', version: '1.0.0' });

server.registerTool(
    'calculate-bmi',
    {
        title: 'BMI Calculator',
        description: 'Calculate Body Mass Index given weight and height',
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

const transport = new StdioServerTransport();
await server.connect(transport);
```

### Tool with multiple return items
```typescript
server.registerTool(
    'get-system-info',
    {
        title: 'System Info',
        description: 'Returns system information including platform and memory usage',
        inputSchema: z.object({})
    },
    async () => ({
        content: [
            { type: 'text', text: `Platform: ${process.platform}` },
            { type: 'text', text: `Node version: ${process.version}` },
            { type: 'text', text: `Memory: ${JSON.stringify(process.memoryUsage())}` }
        ]
    })
);
```

---

## Zod Schema for Input Validation

The SDK uses Zod v4 for input schema validation. The `inputSchema` field accepts a Zod object schema, and the SDK automatically converts it to JSON Schema for the protocol.

### Common Zod schema patterns
```typescript
import { z } from 'zod';

// String with description
z.string().describe('The URL to fetch')

// Optional with default
z.string().optional().default('metric')

// Enum
z.enum(['metric', 'imperial', 'kelvin'])

// Number with constraints
z.number().min(0).max(100).describe('Percentage value')

// Array
z.array(z.string()).describe('List of file paths')

// Nested object
z.object({
    host: z.string(),
    port: z.number().default(5432),
    database: z.string()
})

// Union type
z.union([z.string(), z.number()])
```

### Complete example with rich schema
```typescript
server.registerTool(
    'database-query',
    {
        title: 'Database Query',
        description: 'Execute a read-only SQL query against the database',
        inputSchema: z.object({
            query: z.string().describe('SQL SELECT statement to execute'),
            database: z.enum(['users', 'orders', 'products']).describe('Target database'),
            limit: z.number().min(1).max(1000).default(100).describe('Maximum rows to return'),
            format: z.enum(['json', 'csv', 'table']).optional().default('json')
        })
    },
    async ({ query, database, limit, format }) => {
        // Validate query is read-only
        if (!query.trim().toUpperCase().startsWith('SELECT')) {
            return {
                content: [{ type: 'text', text: 'Error: Only SELECT queries are allowed' }],
                isError: true
            };
        }
        // Execute query...
        const results = await executeQuery(database, query, limit);
        return {
            content: [{ type: 'text', text: formatResults(results, format) }]
        };
    }
);
```

---

## Output Schema (Structured Content)

Tools can return both unstructured content (for the LLM) and structured content (for programmatic use by client applications). If an `outputSchema` is provided, the server **must** return `structuredContent` that conforms to it.

```typescript
server.registerTool(
    'get-weather',
    {
        title: 'Weather Data',
        description: 'Get current weather for a location',
        inputSchema: z.object({
            location: z.string().describe('City name or zip code')
        }),
        outputSchema: z.object({
            temperature: z.number().describe('Temperature in Celsius'),
            conditions: z.string().describe('Weather conditions'),
            humidity: z.number().describe('Humidity percentage')
        })
    },
    async ({ location }) => {
        const data = await fetchWeatherAPI(location);
        const structured = {
            temperature: data.temp_c,
            conditions: data.condition.text,
            humidity: data.humidity
        };
        return {
            // Text content for the LLM
            content: [{ type: 'text', text: JSON.stringify(structured) }],
            // Structured content for client programmatic use
            structuredContent: structured
        };
    }
);
```

---

## Tool Result Content Types

Tool results support multiple content types in the `content` array.

### Text content
```typescript
{ type: 'text', text: 'Result text here' }
```

### Image content
```typescript
{
    type: 'image',
    data: 'base64-encoded-data',
    mimeType: 'image/png',
    annotations: {
        audience: ['user'],
        priority: 0.9
    }
}
```

### Audio content
```typescript
{
    type: 'audio',
    data: 'base64-encoded-audio-data',
    mimeType: 'audio/wav'
}
```

### Resource links (reference large resources without embedding)
```typescript
import { ResourceLink, CallToolResult } from '@modelcontextprotocol/server';

server.registerTool(
    'list-files',
    {
        title: 'List Files',
        description: 'Returns project files as resource links without embedding content'
    },
    async (): Promise<CallToolResult> => {
        const links: ResourceLink[] = [
            {
                type: 'resource_link',
                uri: 'file:///projects/readme.md',
                name: 'README',
                mimeType: 'text/markdown'
            },
            {
                type: 'resource_link',
                uri: 'file:///projects/config.json',
                name: 'Config',
                mimeType: 'application/json'
            }
        ];
        return { content: links };
    }
);
```

### Embedded resources (inline resource content)
```typescript
{
    type: 'resource',
    resource: {
        uri: 'file:///project/src/main.ts',
        mimeType: 'text/typescript',
        text: 'export function hello() { return "world"; }',
        annotations: {
            audience: ['user', 'assistant'],
            priority: 0.7,
            lastModified: '2025-01-12T15:00:58Z'
        }
    }
}
```

---

## Tool Annotations

Annotations provide hints about tool behavior without changing execution semantics. They are **advisory** — clients may use them to decide how to present tools.

```typescript
server.registerTool(
    'read-file',
    {
        title: 'Read File',
        description: 'Read a file from the filesystem',
        inputSchema: z.object({
            path: z.string().describe('File path to read')
        }),
        annotations: {
            readOnlyHint: true,       // Does not modify state
            idempotentHint: true,     // Same result when called multiple times
            destructiveHint: false    // Will not destroy data
        }
    },
    async ({ path }) => {
        const content = await readFile(path, 'utf-8');
        return { content: [{ type: 'text', text: content }] };
    }
);

server.registerTool(
    'delete-file',
    {
        title: 'Delete File',
        description: 'Permanently delete a file from the filesystem',
        inputSchema: z.object({
            path: z.string().describe('File path to delete')
        }),
        annotations: {
            readOnlyHint: false,
            destructiveHint: true,    // This is a destructive operation
            idempotentHint: false
        }
    },
    async ({ path }) => {
        await deleteFile(path);
        return { content: [{ type: 'text', text: `Deleted: ${path}` }] };
    }
);
```

> **Security note**: Clients **must** treat tool annotations as untrusted unless they come from trusted servers.

---

## Error Handling in Tools

Tools use two error reporting mechanisms:

### 1. Tool execution errors (soft errors — tool ran but encountered a problem)
Return `isError: true` in the result. The LLM can read the error text and adapt.

```typescript
server.registerTool(
    'fetch-url',
    {
        title: 'Fetch URL',
        description: 'Fetches content from a URL',
        inputSchema: z.object({
            url: z.string().url().describe('URL to fetch')
        })
    },
    async ({ url }) => {
        try {
            const response = await fetch(url);
            if (!response.ok) {
                return {
                    content: [{
                        type: 'text',
                        text: `HTTP error: ${response.status} ${response.statusText}`
                    }],
                    isError: true
                };
            }
            const text = await response.text();
            return { content: [{ type: 'text', text }] };
        } catch (error) {
            return {
                content: [{
                    type: 'text',
                    text: `Network error: ${error instanceof Error ? error.message : String(error)}`
                }],
                isError: true
            };
        }
    }
);
```

### 2. Protocol errors (hard errors — throw an exception)
Thrown when the tool itself cannot be called (unknown tool, invalid arguments). These surface as JSON-RPC errors.

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "error": {
    "code": -32602,
    "message": "Unknown tool: invalid_tool_name"
  }
}
```

### Decision guide
| Scenario | Approach |
|----------|----------|
| API rate limit exceeded | `isError: true` |
| External service unavailable | `isError: true` |
| Invalid input data (business logic) | `isError: true` |
| Unknown tool name | Protocol error (throw) |
| Tool arguments fail schema validation | Protocol error (automatic) |

---

## Logging inside Tools

Use `ctx.mcpReq.log(level, data)` from the `ServerContext` to send structured log messages to the client. The server must declare the `logging` capability.

```typescript
import { McpServer } from '@modelcontextprotocol/server';
import { CallToolResult } from '@modelcontextprotocol/server';
import { z } from 'zod';

const server = new McpServer(
    { name: 'my-server', version: '1.0.0' },
    { capabilities: { logging: {} } }
);

server.registerTool(
    'fetch-data',
    {
        description: 'Fetch data from an API endpoint',
        inputSchema: z.object({ url: z.string().url() })
    },
    async ({ url }, ctx): Promise<CallToolResult> => {
        await ctx.mcpReq.log('info', `Fetching ${url}`);
        const res = await fetch(url);
        await ctx.mcpReq.log('debug', `Response status: ${res.status}`);

        if (!res.ok) {
            await ctx.mcpReq.log('error', `Failed to fetch: ${res.statusText}`);
            return {
                content: [{ type: 'text', text: `Error: ${res.statusText}` }],
                isError: true
            };
        }

        const text = await res.text();
        await ctx.mcpReq.log('info', `Successfully fetched ${text.length} bytes`);
        return { content: [{ type: 'text', text }] };
    }
);
```

Log levels: `'debug'`, `'info'`, `'warning'`, `'error'`

---

## Sampling inside Tools

Tools can request LLM completions from the connected client via `ctx.mcpReq.requestSampling()`. This allows the server to leverage AI without bundling an LLM SDK.

```typescript
server.registerTool(
    'summarize',
    {
        description: 'Summarize a piece of text using the client LLM',
        inputSchema: z.object({
            text: z.string().describe('Text to summarize'),
            maxWords: z.number().optional().default(100)
        })
    },
    async ({ text, maxWords }, ctx): Promise<CallToolResult> => {
        const response = await ctx.mcpReq.requestSampling({
            messages: [
                {
                    role: 'user',
                    content: {
                        type: 'text',
                        text: `Please summarize the following text in at most ${maxWords} words:\n\n${text}`
                    }
                }
            ],
            maxTokens: 500,
            modelPreferences: {
                hints: [{ name: 'claude-3-sonnet' }],
                speedPriority: 0.8,
                intelligencePriority: 0.5
            }
        });

        return {
            content: [
                {
                    type: 'text',
                    text: `Summary (model: ${response.model}): ${
                        response.content.type === 'text' ? response.content.text : '[non-text response]'
                    }`
                }
            ]
        };
    }
);
```

---

## Protocol Messages

### Listing Tools (tools/list)
```json
// Request
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list",
  "params": {
    "cursor": "optional-cursor-for-pagination"
  }
}

// Response
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "get_weather",
        "title": "Weather Information Provider",
        "description": "Get current weather information for a location",
        "inputSchema": {
          "type": "object",
          "properties": {
            "location": {
              "type": "string",
              "description": "City name or zip code"
            }
          },
          "required": ["location"]
        }
      }
    ],
    "nextCursor": "next-page-cursor"
  }
}
```

### Calling a Tool (tools/call)
```json
// Request
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": {
      "location": "New York"
    }
  }
}

// Response (success)
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Current weather in New York: 72°F, partly cloudy"
      }
    ],
    "isError": false
  }
}

// Response (tool execution error)
{
  "jsonrpc": "2.0",
  "id": 4,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Failed to fetch weather data: API rate limit exceeded"
      }
    ],
    "isError": true
  }
}
```

### List Changed Notification
Sent when the tool list changes (requires `listChanged: true` capability):
```json
{
  "jsonrpc": "2.0",
  "method": "notifications/tools/list_changed"
}
```

Declare this capability on the server:
```typescript
const server = new McpServer(
    { name: 'my-server', version: '1.0.0' },
    { capabilities: { tools: { listChanged: true } } }
);
```

---

## Best Practices

### Naming conventions
- Use `verb_noun` or `verb-noun` format: `get_weather`, `read-file`, `create-user`
- Be specific: `calculator_arithmetic` rather than `calculate`
- Namespace by domain if needed: `database_query`, `database_schema`
- Avoid abbreviations: `get_user_profile` not `get_usr_prof`

### Design guidelines
1. **Single responsibility**: each tool should do one thing well
2. **Descriptive descriptions**: the LLM uses the description to decide when to call the tool — be precise about what the tool does and when to use it
3. **Validate inputs**: always validate with Zod schemas, add constraints (`.min()`, `.max()`, `.url()`, `.email()`, etc.)
4. **Return meaningful errors**: use `isError: true` with informative messages rather than throwing for expected failures
5. **Idempotent where possible**: mark idempotent tools with annotations
6. **Security**: validate all inputs, implement access controls, sanitize outputs, rate limit invocations
7. **Human in the loop**: implement confirmation prompts for destructive operations; clients should always show tool inputs to users before calling
8. **Paginate large results**: return cursors for large datasets
9. **Use outputSchema**: when returning structured data, define `outputSchema` to help clients and LLMs understand the response format
