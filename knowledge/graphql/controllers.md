# GraphQL Serving over HTTP

> Official Documentation: https://graphql.org/learn/serving-over-http/

## Overview

A GraphQL server exposes (typically) a single endpoint — often `POST /graphql` — that accepts a query, variables, and an operation name, executes them against the schema, and returns `{ data, errors }`. The transport is usually HTTP (JSON), with subscriptions over WebSockets or SSE.

This document is framework-neutral where possible and shows the two most popular Node servers:

- **Apollo Server** (`@apollo/server`, v5) — from Apollo.
- **GraphQL Yoga** (`graphql-yoga`, v5) — from The Guild.

Both are built on `graphql-js` and share the same schema/resolver model (see `schema.md`, `resolvers.md`).

---

## 1. The HTTP Contract

A spec-compliant GraphQL-over-HTTP request body contains:

| Field           | Description                              |
|-----------------|------------------------------------------|
| `query`         | The GraphQL document string              |
| `variables`     | Optional map of variable values          |
| `operationName` | Optional; names which operation to run   |

```bash
curl -X POST http://localhost:4000/graphql \
  -H 'Content-Type: application/json' \
  -d '{"query":"query($id:ID!){ user(id:$id){ name } }","variables":{"id":"1"}}'
```

The response is JSON with `data` and/or `errors`. GraphQL commonly returns HTTP 200 even for field errors (details are in the `errors` array), though the GraphQL-over-HTTP spec also defines the `application/graphql-response+json` media type with stricter status semantics.

---

## 2. Apollo Server v5 — Standalone

The quickest way to run Apollo Server. Good for getting started or simple services.

```ts
import { ApolloServer } from '@apollo/server';
import { startStandaloneServer } from '@apollo/server/standalone';

const typeDefs = `#graphql
  type Query { hello: String }
`;

const resolvers = {
  Query: { hello: () => 'world' },
};

const server = new ApolloServer({ typeDefs, resolvers });

const { url } = await startStandaloneServer(server, {
  listen: { port: 4000 },
  context: async ({ req }) => ({
    token: req.headers.authorization,
  }),
});

console.log(`Ready at ${url}`);
```

---

## 3. Apollo Server v5 — Express Integration

For production Apollo recommends integrating with a web framework. In v5 the Express integration lives in a **separate package** (`@as-integrations/express4` or `@as-integrations/express5`) — there is no built-in `@apollo/server/express` import.

```bash
npm install @apollo/server @as-integrations/express5 express cors graphql
```

```ts
import { ApolloServer } from '@apollo/server';
import { expressMiddleware } from '@as-integrations/express5';
import express from 'express';
import cors from 'cors';

interface Context {
  token?: string;
}

const app = express();
const server = new ApolloServer<Context>({ typeDefs, resolvers });

await server.start(); // must start before applying middleware

app.use(
  '/graphql',
  cors<cors.CorsRequest>(),
  express.json(),
  expressMiddleware(server, {
    context: async ({ req, res }) => ({ token: req.headers.authorization }),
  }),
);

app.listen(4000, () => console.log('Ready at http://localhost:4000/graphql'));
```

Use `@as-integrations/express4` instead if you're on Express 4.

---

## 4. GraphQL Yoga

Yoga is a batteries-included server (built on `graphql-js` + Envelop) that runs on many runtimes. It ships an in-browser GraphiQL by default.

```bash
npm install graphql-yoga graphql
```

```ts
import { createYoga, createSchema } from 'graphql-yoga';
import { createServer } from 'node:http';

const yoga = createYoga({
  schema: createSchema({
    typeDefs: /* GraphQL */ `
      type Query { hello: String }
    `,
    resolvers: {
      Query: { hello: () => 'world' },
    },
  }),
  context: async ({ request }) => ({
    token: request.headers.get('authorization'),
  }),
});

const server = createServer(yoga);
server.listen(4000, () => console.log('Ready at http://localhost:4000/graphql'));
```

Yoga also mounts cleanly into Express — pass the `yoga` handler to `app.use('/graphql', yoga)`.

---

## 5. Context Creation

Context is a per-request object passed to every resolver. It's the standard place for authentication, data sources, and per-request DataLoaders.

```ts
// Framework-neutral shape
async function buildContext({ req }): Promise<Context> {
  const user = await authenticate(req.headers.authorization);
  return {
    user,
    db,
    loaders: createLoaders(db), // fresh per request — see resolvers.md
  };
}
```

- Apollo: pass a `context` function to `startStandaloneServer` / `expressMiddleware`.
- Yoga: pass a `context` function to `createYoga` (note Yoga uses the WHATWG `request` object, so headers are read via `request.headers.get(...)`).

Always create DataLoaders inside the context factory so their cache never leaks between users.

---

## 6. Error Handling

By default resolver errors appear in the response `errors` array and internal messages/stack traces are masked in production. Key practices:

- **Throw typed errors** for expected failures. Apollo Server ships `GraphQLError` with `extensions.code`:

```ts
import { GraphQLError } from 'graphql';

throw new GraphQLError('Not authenticated', {
  extensions: { code: 'UNAUTHENTICATED' },
});
```

- **Mask internal errors**: Apollo Server masks unexpected errors by default (shows `INTERNAL_SERVER_ERROR`). Yoga masks errors in production via its `maskedErrors` option (`useMaskedErrors`).
- **Format/log centrally**: Apollo uses `formatError` and plugins; Yoga uses Envelop plugins (`onExecute` / `maskedErrors`) to transform and log.
- **Disable introspection in production** if you don't want the schema publicly discoverable.

```ts
// Apollo: customize the outgoing error shape
const server = new ApolloServer({
  typeDefs,
  resolvers,
  formatError: (formattedError, error) => {
    logger.error(error);
    return formattedError;
  },
});
```

---

## 7. Choosing a Server

| Need                                  | Consider                          |
|---------------------------------------|-----------------------------------|
| Rich Apollo ecosystem, federation     | Apollo Server                     |
| Minimal setup, multi-runtime, Envelop | GraphQL Yoga                      |
| Fastify-first, high performance        | Mercurius                         |
| Full framework with GraphQL modules    | NestJS (`@nestjs/graphql`)        |

All are `graphql-js`-based, so your schema and resolvers port between them with minimal change.

---

## References

- Serving over HTTP: https://graphql.org/learn/serving-over-http/
- GraphQL-over-HTTP spec: https://graphql.github.io/graphql-over-http/
- Apollo Server: https://www.apollographql.com/docs/apollo-server/
- Apollo Express integration: https://www.apollographql.com/docs/apollo-server/api/express-middleware
- GraphQL Yoga: https://the-guild.dev/graphql/yoga-server/docs
