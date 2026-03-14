# trpc-openapi Basics

trpc-openapi exposes tRPC procedures as REST endpoints with OpenAPI documentation.

## Installation

```bash
npm install trpc-openapi
```

## Setup Router

```typescript
import { initTRPC } from '@trpc/server';
import { OpenApiMeta } from 'trpc-openapi';
import { z } from 'zod';

// Initialize with OpenAPI meta
const t = initTRPC.meta<OpenApiMeta>().create();

export const appRouter = t.router({
  getUser: t.procedure
    .meta({
      openapi: {
        method: 'GET',
        path: '/users/{id}',
        tags: ['users'],
        summary: 'Get user by ID',
      },
    })
    .input(z.object({ id: z.string() }))
    .output(z.object({
      id: z.string(),
      name: z.string(),
      email: z.string(),
    }))
    .query(({ input }) => getUserById(input.id)),

  createUser: t.procedure
    .meta({
      openapi: {
        method: 'POST',
        path: '/users',
        tags: ['users'],
      },
    })
    .input(z.object({
      name: z.string(),
      email: z.string().email(),
    }))
    .output(z.object({
      id: z.string(),
      name: z.string(),
      email: z.string(),
    }))
    .mutation(({ input }) => createUser(input)),
});
```

## OpenAPI Metadata

```typescript
.meta({
  openapi: {
    method: 'GET',           // HTTP method
    path: '/users/{id}',     // Path with params
    tags: ['users'],         // OpenAPI tags
    summary: 'Short desc',   // Summary
    description: 'Long desc', // Description
    protect: true,           // Requires auth
    deprecated: false,
  },
})
```

## Generate OpenAPI Document

```typescript
import { generateOpenApiDocument } from 'trpc-openapi';

const doc = generateOpenApiDocument(appRouter, {
  title: 'My API',
  version: '1.0.0',
  baseUrl: 'https://api.example.com',
  securitySchemes: {
    bearerAuth: {
      type: 'http',
      scheme: 'bearer',
    },
  },
});

// Save to file
fs.writeFileSync('./openapi.json', JSON.stringify(doc, null, 2));
```

## REST Handlers

### Next.js Pages Router

```typescript
// pages/api/[...trpc].ts
import { createOpenApiNextHandler } from 'trpc-openapi';

export default createOpenApiNextHandler({
  router: appRouter,
  createContext: () => ({}),
});
```

### Next.js App Router

```typescript
// app/api/[...trpc]/route.ts
import { createOpenApiNextHandler } from 'trpc-openapi';

const handler = createOpenApiNextHandler({
  router: appRouter,
  createContext: () => ({}),
});

export { handler as GET, handler as POST, handler as PUT, handler as DELETE };
```

### Express

```typescript
import { createOpenApiExpressMiddleware } from 'trpc-openapi';

app.use('/api', createOpenApiExpressMiddleware({
  router: appRouter,
  createContext: () => ({}),
}));
```

## Parameters

### Path Parameters

```typescript
// GET /users/{id}
.meta({ openapi: { method: 'GET', path: '/users/{id}' } })
.input(z.object({ id: z.string() }))
```

### Query Parameters

```typescript
// GET /users?status=active
.meta({ openapi: { method: 'GET', path: '/users' } })
.input(z.object({
  status: z.enum(['active', 'inactive']).optional(),
  page: z.number().default(1),
}))
```

### Request Body

```typescript
// POST /users
.meta({ openapi: { method: 'POST', path: '/users' } })
.input(z.object({
  name: z.string(),
  email: z.string().email(),
}))
```

## Authentication

```typescript
// Protected endpoint
.meta({
  openapi: {
    method: 'GET',
    path: '/me',
    protect: true,
  },
})

// Security in OpenAPI doc
const doc = generateOpenApiDocument(appRouter, {
  title: 'API',
  version: '1.0.0',
  baseUrl: 'https://api.example.com',
  securitySchemes: {
    bearerAuth: {
      type: 'http',
      scheme: 'bearer',
    },
  },
});
```

## Error Handling

```typescript
import { TRPCError } from '@trpc/server';

.query(({ input }) => {
  const user = findUser(input.id);
  if (!user) {
    throw new TRPCError({
      code: 'NOT_FOUND',
      message: 'User not found',
    });
  }
  return user;
})
```

## Serve Swagger UI

```typescript
// pages/api/docs.ts
export default function handler(req, res) {
  res.setHeader('Content-Type', 'text/html');
  res.send(`
    <!DOCTYPE html>
    <html>
      <head>
        <link rel="stylesheet" href="https://unpkg.com/swagger-ui-dist/swagger-ui.css" />
      </head>
      <body>
        <div id="swagger-ui"></div>
        <script src="https://unpkg.com/swagger-ui-dist/swagger-ui-bundle.js"></script>
        <script>
          SwaggerUIBundle({ url: '/api/openapi.json', dom_id: '#swagger-ui' });
        </script>
      </body>
    </html>
  `);
}
```
