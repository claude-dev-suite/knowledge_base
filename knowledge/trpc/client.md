# tRPC Client & React Query

> Official Documentation: https://trpc.io/docs/client/react

## Overview

tRPC clients call your server procedures with full type inference derived from the exported `AppRouter` type — no code generation. This document covers tRPC **v11**:

- `@trpc/client` — the framework-agnostic client and **links**.
- `@trpc/react-query` — React hooks built on **TanStack Query** (`@tanstack/react-query`, a third-party library).

Import only the `AppRouter` **type** from the server, never its value.

---

## 1. Vanilla Client (`@trpc/client`)

Use `createTRPCClient` for Node scripts, non-React apps, or anywhere you want a plain proxy.

```ts
import { createTRPCClient, httpBatchLink } from '@trpc/client';
import type { AppRouter } from '../server/router';

const client = createTRPCClient<AppRouter>({
  links: [
    httpBatchLink({ url: 'http://localhost:3000/trpc' }),
  ],
});

// Call procedures directly — fully typed:
const user = await client.getUser.query('id_bilbo');
const created = await client.createUser.mutate({ name: 'Frodo' });
```

Note the method names: queries use `.query()`, mutations use `.mutate()`.

---

## 2. Links

Links form a chain that each request passes through (similar to middleware). The **terminating** link performs the actual transport.

| Link                    | Import               | Purpose                                    |
|-------------------------|----------------------|--------------------------------------------|
| `httpBatchLink`         | `@trpc/client`       | HTTP transport; batches calls into one req |
| `httpLink`              | `@trpc/client`       | HTTP transport, one request per call       |
| `httpSubscriptionLink`  | `@trpc/client`       | Subscriptions over Server-Sent Events      |
| `splitLink`             | `@trpc/client`       | Route to different links by condition       |
| `loggerLink`            | `@trpc/client`       | Dev logging (non-terminating)              |
| `wsLink`                | `@trpc/client`       | WebSocket transport (needs `createWSClient`)|

```ts
import {
  createTRPCClient,
  httpBatchLink,
  loggerLink,
} from '@trpc/client';

const client = createTRPCClient<AppRouter>({
  links: [
    loggerLink({ enabled: () => process.env.NODE_ENV === 'development' }),
    httpBatchLink({
      url: 'http://localhost:3000/trpc',
      // Send auth headers per request:
      async headers() {
        return { authorization: getAuthToken() };
      },
    }),
  ],
});
```

`httpBatchLink` collects calls fired in the same tick into a single HTTP request, reducing round trips. Use `httpLink` if you need one request per call.

---

## 3. React Setup (`@trpc/react-query`)

Create typed hooks with `createTRPCReact`, then provide both a tRPC client and a TanStack `QueryClient`.

```tsx
// utils/trpc.ts
import { createTRPCReact } from '@trpc/react-query';
import type { AppRouter } from '../server/router';

export const trpc = createTRPCReact<AppRouter>();
```

```tsx
// App.tsx
import { useState } from 'react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { httpBatchLink } from '@trpc/client';
import { trpc } from './utils/trpc';

export function App() {
  const [queryClient] = useState(() => new QueryClient());
  const [trpcClient] = useState(() =>
    trpc.createClient({
      links: [httpBatchLink({ url: 'http://localhost:3000/trpc' })],
    }),
  );

  return (
    <trpc.Provider client={trpcClient} queryClient={queryClient}>
      <QueryClientProvider client={queryClient}>
        <YourRoutes />
      </QueryClientProvider>
    </trpc.Provider>
  );
}
```

Use `useState` (not a module-level constant) so each SSR request gets its own client instance.

---

## 4. React Hooks

The hooks mirror your router paths and wrap TanStack Query, so all of TanStack's options and return fields (`data`, `isLoading`, `error`, `isPending`, etc.) are available.

### useQuery

```tsx
function UserProfile() {
  const userQuery = trpc.getUser.useQuery('id_bilbo');

  if (userQuery.isLoading) return <p>Loading…</p>;
  if (userQuery.error) return <p>{userQuery.error.message}</p>;
  return <h1>{userQuery.data.name}</h1>;
}
```

### useMutation

```tsx
function CreateUser() {
  const utils = trpc.useUtils(); // access the query cache
  const createUser = trpc.createUser.useMutation({
    onSuccess() {
      // Invalidate/refetch related queries after a write
      utils.getUser.invalidate();
    },
  });

  return (
    <button onClick={() => createUser.mutate({ name: 'Frodo' })}>
      {createUser.isPending ? 'Saving…' : 'Create'}
    </button>
  );
}
```

`trpc.useUtils()` returns helpers to `invalidate`, `prefetch`, `setData`, etc., scoped by procedure path.

### useSubscription

```tsx
trpc.onMessage.useSubscription(
  { channelId },
  {
    onData(msg) { console.log('new message', msg); },
    onError(err) { console.error(err); },
  },
);
```

Requires a subscription-capable link (`httpSubscriptionLink` or `wsLink`).

---

## 5. Server-Side Calls

To invoke procedures on the same server (e.g. from a REST route, a cron job, or tests), build a **caller** — this runs the middleware/resolvers directly without HTTP.

```ts
import { createCallerFactory } from './trpc';
import { appRouter } from './router';
import { createContext } from './context';

const createCaller = createCallerFactory(appRouter);

const caller = createCaller(await createContext(/* ... */));
const user = await caller.getUser('id_bilbo');
```

`router.createCaller(ctx)` is the equivalent shorthand. Do **not** use callers to invoke one procedure from inside another — extract shared logic into a plain function instead, to avoid re-running context and middleware.

---

## 6. Type Inference

The client infers everything from `AppRouter`. Extract input/output types with the built-in helpers.

```ts
import type { inferRouterInputs, inferRouterOutputs } from '@trpc/server';
import type { AppRouter } from '../server/router';

type RouterInputs = inferRouterInputs<AppRouter>;
type RouterOutputs = inferRouterOutputs<AppRouter>;

// Per-procedure:
type GetUserInput = RouterInputs['getUser'];   // e.g. string
type GetUserOutput = RouterOutputs['getUser']; // e.g. User
```

Because types flow straight from server to client, renaming a procedure input or changing an output type surfaces as a compile error at every call site — the core benefit of tRPC.

---

## References

- React Query integration: https://trpc.io/docs/client/react
- Setup: https://trpc.io/docs/client/react/setup
- Vanilla client: https://trpc.io/docs/client/vanilla
- Links: https://trpc.io/docs/client/links
- Server-side calls: https://trpc.io/docs/server/server-side-calls
- Inferring types: https://trpc.io/docs/client/vanilla/infer-types
