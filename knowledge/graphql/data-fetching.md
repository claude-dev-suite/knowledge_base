# GraphQL Client Data Fetching

> Official Documentation: https://graphql.org/learn/queries/

## Overview

On the client you send GraphQL operations (queries, mutations, subscriptions) to the server and consume the typed results. While you can POST raw JSON, most apps use a client library that adds a **normalized cache**, request deduplication, and React bindings.

This document covers the query mechanics plus two popular clients:

- **Apollo Client** (`@apollo/client`, v4) — from Apollo.
- **urql** (`urql`, v5) — from The Guild.

Both are third-party libraries. Note the v4 Apollo import split: core exports come from `@apollo/client`, while **React hooks/components come from `@apollo/client/react`**.

---

## 1. Queries with Variables

Parameterize operations with typed variables rather than string interpolation.

```graphql
query GetUser($id: ID!) {
  user(id: $id) {
    id
    name
    email
  }
}
```

Variables are sent alongside the query:

```json
{ "id": "1" }
```

Variables keep queries static (cacheable, analyzable) and safe from injection.

---

## 2. Fragments

Fragments are reusable field selections. They keep a component's data requirements colocated and composable.

```graphql
fragment UserFields on User {
  id
  name
  avatarUrl
}

query GetUser($id: ID!) {
  user(id: $id) {
    ...UserFields
    email
  }
}
```

Fragments are the backbone of colocated data (each component declares a fragment; a page composes them into one query).

---

## 3. Caching

Client caches are usually **normalized**: objects are stored by a stable identity (typically `__typename` + `id`) so the same entity referenced from multiple queries stays consistent. Fetching a field again updates every query that references that object.

- **Apollo**: `InMemoryCache`, keyed by `__typename` + `id`; customizable via `typePolicies` and `keyFields`.
- **urql**: a document cache by default; `@urql/exchange-graphcache` adds normalized caching.

Because of normalization, a mutation that returns updated objects can automatically refresh cached queries — no manual refetch needed when ids match.

---

## 4. Apollo Client (v4) Setup

```bash
npm install @apollo/client graphql
```

```tsx
import { ApolloClient, InMemoryCache, HttpLink } from '@apollo/client';
import { ApolloProvider } from '@apollo/client/react'; // hooks/components live here in v4

const client = new ApolloClient({
  link: new HttpLink({ uri: 'https://api.example.com/graphql' }),
  cache: new InMemoryCache(),
});

export function App() {
  return (
    <ApolloProvider client={client}>
      <Users />
    </ApolloProvider>
  );
}
```

### Querying with hooks

```tsx
import { gql } from '@apollo/client';
import { useQuery } from '@apollo/client/react';

const GET_USER = gql`
  query GetUser($id: ID!) {
    user(id: $id) { id name }
  }
`;

function User({ id }: { id: string }) {
  const { data, loading, error } = useQuery(GET_USER, { variables: { id } });

  if (loading) return <p>Loading…</p>;
  if (error) return <p>{error.message}</p>;
  return <h1>{data.user.name}</h1>;
}
```

---

## 5. Mutations & Cache Updates (Apollo)

```tsx
import { gql } from '@apollo/client';
import { useMutation } from '@apollo/client/react';

const ADD_TODO = gql`
  mutation AddTodo($text: String!) {
    addTodo(text: $text) { id text }
  }
`;

const GET_TODOS = gql`query { todos { id text } }`;

function AddTodo() {
  const [addTodo, { loading }] = useMutation(ADD_TODO, {
    // Option A: update the cache manually
    update(cache, { data }) {
      const newTodo = data.addTodo;
      cache.modify({
        fields: {
          todos(existing = []) {
            const ref = cache.writeFragment({
              data: newTodo,
              fragment: gql`fragment NewTodo on Todo { id text }`,
            });
            return [...existing, ref];
          },
        },
      });
    },
  });

  return <button onClick={() => addTodo({ variables: { text: 'New' } })}>Add</button>;
}
```

Cache update strategies:

| Strategy                          | When to use                                              |
|-----------------------------------|---------------------------------------------------------|
| Return updated object from mutation | Editing an existing entity (auto-merges by id)         |
| `refetchQueries`                  | Simplest; re-runs listed queries after the mutation     |
| `update` + `cache.modify`/`writeQuery` | Add/remove list items without a network refetch    |
| `optimisticResponse`              | Update UI instantly before the server responds          |

---

## 6. Pagination

Two common shapes:

- **Offset/limit**: `posts(offset: Int, limit: Int)` — simple, but shifts on inserts.
- **Cursor (Relay connections)**: `posts(first: Int, after: String)` returning `edges { node cursor } pageInfo { hasNextPage endCursor }` — stable for infinite scroll.

Apollo fetches more pages with `fetchMore` and merges them via a field policy:

```tsx
const { data, fetchMore } = useQuery(GET_POSTS, {
  variables: { first: 10 },
});

// load next page
fetchMore({ variables: { after: data.posts.pageInfo.endCursor } });
```

```ts
// Merge pages in the cache (Apollo ships a Relay-style helper)
import { relayStylePagination } from '@apollo/client/utilities';

const cache = new InMemoryCache({
  typePolicies: {
    Query: { fields: { posts: relayStylePagination() } },
  },
});
```

---

## 7. urql (v5) Basics

A lighter, exchange-based client.

```bash
npm install urql graphql
```

```tsx
import { createClient, Provider, useQuery, useMutation, gql,
         cacheExchange, fetchExchange } from 'urql';

const client = createClient({
  url: 'https://api.example.com/graphql',
  exchanges: [cacheExchange, fetchExchange],
});

function Users() {
  const [{ data, fetching, error }] = useQuery({
    query: gql`{ users { id name } }`,
  });
  if (fetching) return <p>Loading…</p>;
  if (error) return <p>{error.message}</p>;
  return <ul>{data.users.map((u) => <li key={u.id}>{u.name}</li>)}</ul>;
}

// Mutation:  const [, addUser] = useMutation(ADD_USER); addUser({ name: 'Ada' });
// Wrap the tree:  <Provider value={client}><App /></Provider>
```

urql's default document cache invalidates by `__typename`; add `@urql/exchange-graphcache` for normalized caching and precise cache updates.

---

## 8. Client Comparison

| Feature            | Apollo Client (v4)          | urql (v5)                       |
|--------------------|-----------------------------|---------------------------------|
| Cache              | Normalized (`InMemoryCache`)| Document by default; Graphcache add-on |
| Bundle size        | Larger, feature-rich        | Smaller, modular via exchanges  |
| Extensibility      | Links                       | Exchanges                       |
| React hooks import | `@apollo/client/react`      | `urql`                          |

For fully typed operations with either client, pair with **GraphQL Code Generator** (`@graphql-codegen/cli`) to generate typed hooks from your `.graphql` documents.

---

## References

- Queries & mutations: https://graphql.org/learn/queries/
- Pagination: https://graphql.org/learn/pagination/
- Apollo Client: https://www.apollographql.com/docs/react/
- urql: https://commerce.nearform.com/open-source/urql/docs/
