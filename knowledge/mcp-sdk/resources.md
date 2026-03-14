# MCP SDK - Resources

> Official Documentation: https://modelcontextprotocol.io/docs/concepts/resources

## Overview

Resources are data sources that provide contextual information to AI applications — file contents, database records, API responses, configuration data, and more. Resources are **application-driven**: the host application decides how to incorporate them based on its needs. Each resource is uniquely identified by a URI conforming to RFC 3986.

---

## Table of Contents

1. [Resource Definition and Structure](#resource-definition-and-structure)
2. [Capabilities](#capabilities)
3. [Registering Static Resources](#registering-static-resources)
4. [Registering Dynamic Resources (Templates)](#registering-dynamic-resources-templates)
5. [Resource URIs and Common URI Schemes](#resource-uris-and-common-uri-schemes)
6. [Resource Content Types](#resource-content-types)
7. [Annotations](#annotations)
8. [Subscriptions (Real-time Updates)](#subscriptions)
9. [Protocol Messages](#protocol-messages)
10. [Error Handling](#error-handling)
11. [Best Practices](#best-practices)

---

## Resource Definition and Structure

A resource definition includes:
- `uri`: Unique identifier (RFC 3986 URI)
- `name`: Name of the resource
- `title`: Optional human-readable display name
- `description`: Optional description
- `mimeType`: Optional MIME type
- `size`: Optional size in bytes

---

## Capabilities

Servers that support resources must declare the `resources` capability. Two optional features:
- `subscribe`: whether clients can subscribe to change notifications on individual resources
- `listChanged`: whether the server emits notifications when the overall resource list changes

```typescript
const server = new McpServer(
    { name: 'my-server', version: '1.0.0' },
    {
        capabilities: {
            resources: {
                subscribe: true,      // clients can subscribe to individual resources
                listChanged: true     // server notifies when the list changes
            }
        }
    }
);
```

Or minimal (no extra features):
```typescript
{ capabilities: { resources: {} } }
```

---

## Registering Static Resources

Static resources expose data at a fixed URI.

```typescript
import { McpServer } from '@modelcontextprotocol/server';
import { StdioServerTransport } from '@modelcontextprotocol/server';

const server = new McpServer({ name: 'my-server', version: '1.0.0' });

// Plain text resource
server.registerResource(
    'config',
    'config://app',
    {
        title: 'Application Config',
        description: 'Application configuration data',
        mimeType: 'text/plain'
    },
    async (uri) => ({
        contents: [{ uri: uri.href, text: 'App configuration here' }]
    })
);

// JSON resource
server.registerResource(
    'db-schema',
    'schema://database',
    {
        title: 'Database Schema',
        description: 'Current database schema definition',
        mimeType: 'application/json'
    },
    async (uri) => {
        const schema = await getDatabaseSchema();
        return {
            contents: [{
                uri: uri.href,
                mimeType: 'application/json',
                text: JSON.stringify(schema, null, 2)
            }]
        };
    }
);

// Binary resource
server.registerResource(
    'logo',
    'assets://logo.png',
    {
        title: 'Application Logo',
        description: 'PNG logo image',
        mimeType: 'image/png'
    },
    async (uri) => {
        const imageBuffer = await readFile('./logo.png');
        return {
            contents: [{
                uri: uri.href,
                mimeType: 'image/png',
                blob: imageBuffer.toString('base64')
            }]
        };
    }
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

---

## Registering Dynamic Resources (Templates)

Dynamic resources use URI templates (RFC 6570) with `ResourceTemplate`. The template defines parameterized URIs that can be instantiated with different values.

```typescript
import { ResourceTemplate } from '@modelcontextprotocol/server';

// User profile template: user://{userId}/profile
server.registerResource(
    'user-profile',
    new ResourceTemplate('user://{userId}/profile', {
        // list() provides the enumerable resources for this template
        list: async () => ({
            resources: [
                { uri: 'user://123/profile', name: 'Alice' },
                { uri: 'user://456/profile', name: 'Bob' },
                { uri: 'user://789/profile', name: 'Charlie' }
            ]
        })
    }),
    {
        title: 'User Profile',
        description: 'User profile data by user ID',
        mimeType: 'application/json'
    },
    async (uri, { userId }) => ({
        contents: [{
            uri: uri.href,
            mimeType: 'application/json',
            text: JSON.stringify(await getUserById(userId))
        }]
    })
);

// File template: file:///{path}
server.registerResource(
    'file',
    new ResourceTemplate('file:///{path}', {
        list: async () => ({
            resources: await listProjectFiles()
        })
    }),
    {
        title: 'Project Files',
        description: 'Access files in the project directory',
        mimeType: 'application/octet-stream'
    },
    async (uri, { path }) => {
        const content = await readFile(path, 'utf-8');
        return {
            contents: [{
                uri: uri.href,
                mimeType: detectMimeType(path),
                text: content
            }]
        };
    }
);
```

### Template with argument completions
Use `completable()` from the server package to provide autocompletion suggestions on template parameters:

```typescript
import { completable } from '@modelcontextprotocol/server';

server.registerResource(
    'document',
    new ResourceTemplate(
        completable('docs://{docId}', async (value) => {
            const docs = await searchDocuments(value);
            return docs.map(d => d.id);
        }),
        { list: async () => ({ resources: await listDocuments() }) }
    ),
    {
        title: 'Documentation',
        mimeType: 'text/markdown'
    },
    async (uri, { docId }) => ({
        contents: [{
            uri: uri.href,
            mimeType: 'text/markdown',
            text: await getDocument(docId)
        }]
    })
);
```

---

## Resource URIs and Common URI Schemes

### `file://`
For resources that behave like filesystem entries. Resources don't need to map to actual filesystem files.
```
file:///project/src/main.ts
file:///config/settings.json
```

### `https://`
For web resources that the client can fetch directly. Use only when the client can independently fetch the resource without going through the MCP server.
```
https://api.example.com/data
```

### `git://`
For git version control integration.
```
git://repo/branch/path/to/file
```

### Custom URI schemes
Must conform to RFC 3986. Define domain-specific schemes for your application:
```
config://app/settings
schema://database/public
user://123/profile
metrics://cpu/usage
```

---

## Resource Content Types

Resources return content in two formats:

### Text content
```typescript
{
    uri: 'file:///example.txt',
    mimeType: 'text/plain',
    text: 'Resource content here'
}
```

### Binary content (base64 encoded)
```typescript
{
    uri: 'file:///example.png',
    mimeType: 'image/png',
    blob: 'iVBORw0KGgoAAAANSUhEUgAA...' // base64 encoded
}
```

A single `resources/read` response can return multiple content items:
```typescript
async (uri) => ({
    contents: [
        { uri: uri.href, mimeType: 'text/plain', text: 'Main content' },
        { uri: uri.href + '#metadata', mimeType: 'application/json', text: JSON.stringify(metadata) }
    ]
})
```

---

## Annotations

Resources, resource templates, and content blocks support optional annotations that provide hints to clients:

- `audience`: `"user"` and/or `"assistant"` — intended consumers of this resource
- `priority`: `0.0` to `1.0` — importance (1 = required, 0 = optional)
- `lastModified`: ISO 8601 timestamp of last modification

```typescript
server.registerResource(
    'readme',
    'file:///project/README.md',
    {
        title: 'Project README',
        description: 'Main project documentation',
        mimeType: 'text/markdown',
        annotations: {
            audience: ['user'],
            priority: 0.8,
            lastModified: '2025-01-12T15:00:58Z'
        }
    },
    async (uri) => ({
        contents: [{
            uri: uri.href,
            mimeType: 'text/markdown',
            text: await readFile('./README.md', 'utf-8'),
            annotations: {
                audience: ['user', 'assistant'],
                priority: 0.7
            }
        }]
    })
);
```

Clients use annotations to:
- Filter resources by audience
- Prioritize which resources to include in context windows
- Sort by recency using `lastModified`

---

## Subscriptions

When the server declares `subscribe: true`, clients can subscribe to specific resources and receive notifications when they change.

### Server side: notify on resource change
```typescript
// The server can send update notifications from anywhere in its code
server.sendResourceUpdated({ uri: 'config://app' });
server.sendResourceListChanged();
```

### Client side: subscribe and handle updates
```typescript
// Subscribe to a resource
await client.subscribeResource({ uri: 'config://app' });

// Handle update notifications
client.setNotificationHandler('notifications/resources/updated', async (notification) => {
    if (notification.params.uri === 'config://app') {
        // Re-read the updated resource
        const { contents } = await client.readResource({ uri: 'config://app' });
        console.log('Config updated:', contents);
        // Update your application state
        applyNewConfig(contents);
    }
});

// Later: stop receiving updates
await client.unsubscribeResource({ uri: 'config://app' });
```

---

## Protocol Messages

### Listing Resources (resources/list)
```json
// Request
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "resources/list",
  "params": {
    "cursor": "optional-cursor-value"
  }
}

// Response
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "resources": [
      {
        "uri": "file:///project/src/main.rs",
        "name": "main.rs",
        "title": "Application Main File",
        "description": "Primary application entry point",
        "mimeType": "text/x-rust"
      }
    ],
    "nextCursor": "next-page-cursor"
  }
}
```

### Reading a Resource (resources/read)
```json
// Request
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "resources/read",
  "params": {
    "uri": "file:///project/src/main.rs"
  }
}

// Response
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "contents": [
      {
        "uri": "file:///project/src/main.rs",
        "mimeType": "text/x-rust",
        "text": "fn main() {\n    println!(\"Hello world!\");\n}"
      }
    ]
  }
}
```

### Listing Resource Templates (resources/templates/list)
```json
// Request
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "resources/templates/list"
}

// Response
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "resourceTemplates": [
      {
        "uriTemplate": "file:///{path}",
        "name": "Project Files",
        "title": "Project Files",
        "description": "Access files in the project directory",
        "mimeType": "application/octet-stream"
      }
    ]
  }
}
```

### Subscribe / Unsubscribe
```json
// Subscribe request
{
  "jsonrpc": "2.0",
  "id": 4,
  "method": "resources/subscribe",
  "params": { "uri": "file:///project/src/main.rs" }
}

// Update notification (server → client, no id)
{
  "jsonrpc": "2.0",
  "method": "notifications/resources/updated",
  "params": { "uri": "file:///project/src/main.rs" }
}

// List changed notification
{
  "jsonrpc": "2.0",
  "method": "notifications/resources/list_changed"
}
```

---

## Error Handling

```json
// Resource not found (-32002)
{
  "jsonrpc": "2.0",
  "id": 5,
  "error": {
    "code": -32002,
    "message": "Resource not found",
    "data": { "uri": "file:///nonexistent.txt" }
  }
}

// Internal error (-32603)
{
  "jsonrpc": "2.0",
  "id": 6,
  "error": {
    "code": -32603,
    "message": "Internal error reading resource"
  }
}
```

In the handler:
```typescript
server.registerResource(
    'secure-config',
    'config://secrets',
    { title: 'Secrets', mimeType: 'application/json' },
    async (uri, _params, ctx) => {
        // Check permissions
        if (!ctx.hasPermission('config:read')) {
            throw new Error('Access denied'); // becomes -32603
        }
        return {
            contents: [{ uri: uri.href, text: await getSecrets() }]
        };
    }
);
```

---

## Best Practices

1. **Use appropriate URI schemes**: `file://` for filesystem-like data, `https://` only when clients can fetch directly, custom schemes for domain-specific resources

2. **Set accurate MIME types**: helps clients decide how to display or process content

3. **Provide descriptions**: the description is used by AI to decide what context to include

4. **Use annotations wisely**:
   - Mark assistant-only content with `audience: ['assistant']`
   - Set `priority` to help clients manage context window size
   - Always include `lastModified` for frequently-changing resources

5. **Implement subscriptions for live data**: use `subscribe: true` for resources that change frequently (config files, database state, metrics)

6. **Validate URIs**: always validate resource URIs before accessing underlying data

7. **Access controls**: implement proper permission checks before returning sensitive resources

8. **Encode binary data correctly**: always base64-encode binary content in the `blob` field

9. **Pagination**: use `nextCursor` for large resource lists

10. **Don't perform heavy computation**: resources should be data retrieval, not computation — use tools for heavy processing
