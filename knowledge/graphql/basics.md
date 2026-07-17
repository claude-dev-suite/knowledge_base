# GraphQL Basics

> Official Documentation: https://graphql.org/learn/

## Overview

GraphQL is a query language for APIs and a runtime for fulfilling those queries against your data. A GraphQL service exposes a single endpoint backed by a strongly typed **schema**. Clients ask for exactly the fields they need — no more, no less — and receive a predictably shaped JSON response.

GraphQL is a **specification**, not a specific library. It is implemented across many languages; the JavaScript reference implementation is `graphql-js` (imported as `graphql`). This document is spec-level and framework-neutral.

Key properties:

- **Ask for what you need** — the response mirrors the query's shape.
- **Get many resources in one request** — traverse related objects in a single round trip.
- **Strongly typed** — every field has a type, enabling validation and tooling.
- **Introspective** — the schema can be queried at runtime, powering docs and codegen.

---

## 1. The Three Operation Types

GraphQL defines three root operation types.

| Operation      | Purpose                              | Root type      |
|----------------|--------------------------------------|----------------|
| `query`        | Read data (idempotent)               | `Query`        |
| `mutation`     | Write data (create/update/delete)    | `Mutation`     |
| `subscription` | Long-lived stream of events          | `Subscription` |

### Query

```graphql
query GetUser {
  user(id: "1") {
    id
    name
    posts {
      title
    }
  }
}
```

Response mirrors the query exactly:

```json
{
  "data": {
    "user": {
      "id": "1",
      "name": "Ada",
      "posts": [{ "title": "Hello" }]
    }
  }
}
```

### Mutation

```graphql
mutation CreatePost {
  createPost(input: { title: "New", body: "..." }) {
    id
    title
  }
}
```

Fields in a mutation's top level run **in series** (queries may resolve in parallel).

### Subscription

```graphql
subscription OnCommentAdded {
  commentAdded(postId: "1") {
    id
    content
  }
}
```

Subscriptions typically stream over WebSockets (e.g. the `graphql-ws` protocol) or Server-Sent Events, emitting a new payload each time the event fires.

---

## 2. Operation Anatomy

```graphql
query GetUser($id: ID!) {   # operation type, name, variable definitions
  user(id: $id) {           # field with an argument
    id                      # scalar field
    name
    ...UserFields           # fragment spread
  }
}

fragment UserFields on User {
  email
}
```

- **Fields** select data; they can be nested to traverse relationships.
- **Arguments** parameterize fields (e.g. `user(id: ...)`).
- **Variables** (`$id`) are typed inputs supplied alongside the query.
- **Aliases** rename fields in the response: `admin: user(id: "1")`.
- **Fragments** are reusable field sets.
- **Directives** like `@include(if: ...)` and `@skip(if: ...)` conditionally include fields.

---

## 3. Type System Overview

The schema is built from a small set of type kinds. See `schema.md` for full detail.

| Kind        | Description                                             |
|-------------|--------------------------------------------------------|
| Scalar      | Leaf values: `Int`, `Float`, `String`, `Boolean`, `ID` |
| Object      | Named set of fields (e.g. `User`, `Post`)              |
| Input       | Object used as an argument/input value                 |
| Enum        | Fixed set of allowed string values                     |
| Interface   | Abstract type implemented by objects                   |
| Union       | One of several object types                            |
| List        | `[T]` — an ordered list of `T`                         |
| Non-Null    | `T!` — value is guaranteed present                     |

```graphql
type User {
  id: ID!
  name: String!
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
  author: User!
}

type Query {
  user(id: ID!): User
}
```

Every query is validated against these types before execution.

---

## 4. GraphQL vs REST

| Aspect            | REST                                   | GraphQL                                    |
|-------------------|----------------------------------------|--------------------------------------------|
| Endpoints         | Many (one per resource)                | Usually one                                |
| Data shape        | Fixed per endpoint                     | Client selects fields                      |
| Over/under-fetch  | Common                                 | Avoided — request exactly what's needed    |
| Multiple resources| Often multiple round trips             | One request can traverse the graph         |
| Typing / schema   | Optional (e.g. OpenAPI)                | Built-in, mandatory schema                 |
| Versioning        | Often `/v2/…`                          | Evolve by adding fields; deprecate old ones|
| Caching           | Leverages HTTP caching easily          | App-level caching (normalized client cache)|

GraphQL is not a replacement for REST in every case — HTTP caching, file uploads, and simple CRUD can be simpler with REST. GraphQL shines when clients have varied data needs across a richly connected graph.

---

## 5. Ecosystem

- **Servers**: Apollo Server, GraphQL Yoga, Mercurius, Envelop. (See `controllers.md`.)
- **Clients**: Apollo Client, urql, Relay, `graphql-request`. (See `data-fetching.md`.)
- **Schema tools**: `graphql-js`, `@graphql-tools` (schema stitching, `makeExecutableSchema`), Pothos and Nexus (code-first schemas).
- **Tooling**: GraphQL Code Generator (typed operations), GraphiQL / Apollo Sandbox (in-browser IDEs), ESLint plugin.
- **Federation**: Apollo Federation and GraphQL Mesh compose multiple services into one supergraph.
- **N+1 mitigation**: DataLoader (batching/caching). (See `resolvers.md`.)

---

## References

- Learn GraphQL: https://graphql.org/learn/
- Specification: https://spec.graphql.org/
- Queries & mutations: https://graphql.org/learn/queries/
- graphql-js: https://github.com/graphql/graphql-js
