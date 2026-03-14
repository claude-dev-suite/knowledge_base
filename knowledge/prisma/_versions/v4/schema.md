# Prisma 4 → Schema Delta

## Not Available in Prisma 4

- Client extensions ($extends)
- Prisma Accelerate (connection pooling)
- Prisma Pulse (real-time)
- jsonProtocol as default
- Improved TypeScript inference

## Syntax Differences

### Client Extensions (Prisma 5+)

```typescript
// Prisma 5 - Client extensions
const prisma = new PrismaClient().$extends({
  model: {
    user: {
      async findByEmail(email: string) {
        return prisma.user.findFirst({ where: { email } });
      },
    },
  },
  query: {
    $allOperations({ operation, model, args, query }) {
      const start = performance.now();
      const result = query(args);
      const end = performance.now();
      console.log(`${model}.${operation} took ${end - start}ms`);
      return result;
    },
  },
});

// Usage
const user = await prisma.user.findByEmail('user@example.com');


// Prisma 4 - Middleware (different approach)
const prisma = new PrismaClient();

prisma.$use(async (params, next) => {
  const start = performance.now();
  const result = await next(params);
  const end = performance.now();
  console.log(`${params.model}.${params.action} took ${end - start}ms`);
  return result;
});

// Custom methods require separate module
```

### Query Engine Protocol

```typescript
// Prisma 5 - jsonProtocol is default
const prisma = new PrismaClient();
// Uses JSON protocol for better performance

// Prisma 4 - graphql protocol default
// To enable JSON protocol (preview):
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["jsonProtocol"]
}
```

### Prisma Accelerate (Prisma 5+)

```typescript
// Prisma 5 - Edge-compatible with Accelerate
import { PrismaClient } from '@prisma/client/edge';
import { withAccelerate } from '@prisma/extension-accelerate';

const prisma = new PrismaClient().$extends(withAccelerate());

// With caching
const users = await prisma.user.findMany({
  cacheStrategy: {
    ttl: 60,        // Cache for 60 seconds
    swr: 120,       // Stale-while-revalidate for 120s
  },
});


// Prisma 4 - Connection pooling via PgBouncer or external
// No built-in edge support
```

### Improved Type Inference (Prisma 5+)

```typescript
// Prisma 5 - Better inference for select/include
const user = await prisma.user.findUnique({
  where: { id: 1 },
  select: {
    id: true,
    email: true,
    posts: {
      select: { title: true },
    },
  },
});

// TypeScript knows exact shape:
// { id: number; email: string; posts: { title: string }[] }


// Prisma 4 - Similar but less precise inference
// May require explicit typing in complex cases
```

## Still Current in Prisma 4

- Schema language
- CRUD operations
- Relations
- Migrations
- Middleware
- Transactions
- Raw queries
- Aggregations

## Schema Definition (Same in Both)

```prisma
// schema.prisma - Works in Prisma 4 and 5
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String?
  posts     Post[]
  createdAt DateTime @default(now())
}

model Post {
  id        Int      @id @default(autoincrement())
  title     String
  content   String?
  published Boolean  @default(false)
  author    User     @relation(fields: [authorId], references: [id])
  authorId  Int
}
```

## Recommendations for Prisma 4 Users

1. **Use middleware** for logging/soft-delete
2. **PgBouncer** for connection pooling
3. **Explicit types** for complex queries
4. **Plan upgrade** for $extends and Accelerate

## Migration Path

1. Update Prisma packages to v5
2. Run `npx prisma generate`
3. Replace middleware with $extends if desired
4. Update any jsonProtocol preview flags
5. Consider Prisma Accelerate for edge deployments

