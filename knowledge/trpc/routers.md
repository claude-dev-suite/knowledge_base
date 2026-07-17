# tRPC Routers & Procedures

> Official Documentation: https://trpc.io/docs/server/routers

## Overview

tRPC lets you build fully type-safe APIs without schemas or code generation. On the server you define a **router** made of **procedures** (queries, mutations, and subscriptions). The router's type is exported to the client, which then gets end-to-end type inference for inputs and outputs.

This document covers tRPC **v11** (`@trpc/server` 11.x). Input validation examples use **zod** (a third-party schema library), but any supported validator works.

---

## 1. Initialization

Initialize tRPC **exactly once** per application, then export reusable helpers.

```ts
// server/trpc.ts
import { initTRPC } from '@trpc/server';

const t = initTRPC.create();

// Reusable helpers
export const router = t.router;
export const publicProcedure = t.procedure;
export const middleware = t.middleware;
export const createCallerFactory = t.createCallerFactory;
```

Only export the helpers (`router`, `procedure`, etc.) — never the `t` object — to keep the surface small.

---

## 2. Defining a Router

A router is a collection of procedures and/or nested routers.

```ts
// server/router.ts
import { router, publicProcedure } from './trpc';

export const appRouter = router({
  greeting: publicProcedure.query(() => 'hello tRPC v11!'),
});

// Export the TYPE only — never the router value — to the client.
export type AppRouter = typeof appRouter;
```

### Nested / sub-routers

Both an inline object and a nested `router()` call are equivalent:

```ts
export const appRouter = router({
  // nested router()
  user: router({
    list: publicProcedure.query(() => [/* ... */]),
  }),
  // plain object literal (also becomes a namespace)
  post: {
    list: publicProcedure.query(() => [/* ... */]),
  },
});
```

Procedures are then called by path, e.g. `user.list`, `post.list`.

---

## 3. Procedure Types

| Type           | Builder method   | Purpose                         | Client hook          |
|----------------|------------------|---------------------------------|----------------------|
| Query          | `.query()`       | Read data (idempotent)          | `useQuery`           |
| Mutation       | `.mutation()`    | Create / update / delete data   | `useMutation`        |
| Subscription   | `.subscription()`| Long-lived stream of events     | `useSubscription`    |

```ts
export const appRouter = router({
  getUser: publicProcedure
    .input(z.string())
    .query(({ input }) => db.user.find(input)),

  createUser: publicProcedure
    .input(z.object({ name: z.string() }))
    .mutation(({ input }) => db.user.create(input)),
});
```

### Subscriptions

tRPC v11 subscriptions are written as **async generators** that `yield` values. They require a compatible client transport (e.g. `httpSubscriptionLink` for SSE or a WebSocket link).

```ts
onMessage: publicProcedure
  .input(z.object({ channelId: z.string() }))
  .subscription(async function* ({ input, signal }) {
    for await (const msg of subscribeToChannel(input.channelId, { signal })) {
      yield msg;
    }
  }),
```

---

## 4. Input Validation

Attach a validator with `.input()`. The parsed, typed value is available as `input` in the resolver. Multiple `.input()` calls are merged.

```ts
import { z } from 'zod'; // third-party validator

const createPost = publicProcedure
  .input(
    z.object({
      title: z.string().min(1),
      body: z.string(),
    }),
  )
  .mutation(({ input }) => {
    //          ^ typed as { title: string; body: string }
    return db.post.create(input);
  });
```

Output can also be validated/narrowed with `.output()`. tRPC supports zod, Yup, Valibot, ArkType, Superstruct, and any function that parses/throws.

---

## 5. Context

Context is created per-request and made available to every procedure via `ctx`. Type it at init time with `.context<T>()`.

```ts
// server/context.ts
import type { CreateHTTPContextOptions } from '@trpc/server/adapters/standalone';

export function createContext({ req }: CreateHTTPContextOptions) {
  const user = getUserFromHeader(req.headers.authorization);
  return { user };
}
export type Context = Awaited<ReturnType<typeof createContext>>;

// server/trpc.ts — wire the type in at init
const t = initTRPC.context<Context>().create();
```

The context factory is passed to the adapter (see the adapters docs).

---

## 6. Middleware

Middleware wraps procedure execution. It receives `opts`, must call `opts.next()`, and can extend/narrow the context in a type-safe way.

```ts
import { TRPCError } from '@trpc/server';
import { publicProcedure } from './trpc';

// Reusable "protected" procedure
export const protectedProcedure = publicProcedure.use(async (opts) => {
  const { ctx } = opts;
  if (!ctx.user) {
    throw new TRPCError({ code: 'UNAUTHORIZED' });
  }
  return opts.next({
    ctx: {
      user: ctx.user, // now non-nullable for downstream resolvers
    },
  });
});
```

Middleware can also be created standalone with `t.middleware(...)` and attached via `.use()`. Multiple `.use()` calls chain in order.

---

## 7. Merging Routers

Compose routers by nesting them under keys when building the app router.

```ts
import { router } from './trpc';
import { userRouter } from './routers/user';
import { postRouter } from './routers/post';

export const appRouter = router({
  user: userRouter,
  post: postRouter,
});

export type AppRouter = typeof appRouter;
```

Procedures are then reachable as `user.*` and `post.*` on the client.

---

## 8. Error Handling

Throw `TRPCError` with a standardized code. Codes map to HTTP statuses automatically.

```ts
import { TRPCError } from '@trpc/server';

const getPost = publicProcedure
  .input(z.string())
  .query(async ({ input }) => {
    const post = await db.post.find(input);
    if (!post) {
      throw new TRPCError({
        code: 'NOT_FOUND',
        message: 'Post not found',
        cause: undefined, // optional original error, retains stack trace
      });
    }
    return post;
  });
```

### Common error codes

| Code                    | HTTP |
|-------------------------|------|
| `BAD_REQUEST`           | 400  |
| `UNAUTHORIZED`          | 401  |
| `FORBIDDEN`             | 403  |
| `NOT_FOUND`             | 404  |
| `TIMEOUT`               | 408  |
| `CONFLICT`              | 409  |
| `TOO_MANY_REQUESTS`     | 429  |
| `INTERNAL_SERVER_ERROR` | 500  |

Any thrown value that isn't a `TRPCError` is wrapped as `INTERNAL_SERVER_ERROR`. Validation failures from `.input()` surface as `BAD_REQUEST`.

### Formatting errors

Customize the error shape sent to the client with `errorFormatter` at init:

```ts
import { initTRPC } from '@trpc/server';
import { ZodError } from 'zod';

const t = initTRPC.create({
  errorFormatter({ shape, error }) {
    return {
      ...shape,
      data: {
        ...shape.data,
        zodError:
          error.cause instanceof ZodError ? error.cause.flatten() : null,
      },
    };
  },
});
```

Log errors centrally with the adapter's `onError` callback, which receives `{ error, type, path, input, ctx, req }`.

---

## References

- Routers / Procedures: https://trpc.io/docs/server/routers
- Middlewares: https://trpc.io/docs/server/middlewares
- Context: https://trpc.io/docs/server/context
- Error handling: https://trpc.io/docs/server/error-handling
