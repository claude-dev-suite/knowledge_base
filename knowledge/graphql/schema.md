# GraphQL Schema

> Official Documentation: https://graphql.org/learn/schema/

## Overview

The schema is the contract of a GraphQL API. It declares every type, field, and operation the server supports. The canonical way to express a schema is the **Schema Definition Language (SDL)** — a language-agnostic, human-readable syntax. Clients and servers both validate against it.

This document is spec-level and framework-neutral (SDL as defined by the GraphQL specification).

---

## 1. Schema Definition Language (SDL)

A schema wires the three root operation types to object types.

```graphql
schema {
  query: Query
  mutation: Mutation
  subscription: Subscription
}

type Query {
  users: [User!]!
  user(id: ID!): User
}
```

`Query` is required; `Mutation` and `Subscription` are optional. If your root types are named exactly `Query`, `Mutation`, and `Subscription`, the explicit `schema { … }` block can be omitted.

---

## 2. Scalar Types

Built-in scalars (leaf values):

| Scalar    | Description                                    |
|-----------|------------------------------------------------|
| `Int`     | Signed 32-bit integer                          |
| `Float`   | Signed double-precision floating point         |
| `String`  | UTF-8 character sequence                        |
| `Boolean` | `true` / `false`                               |
| `ID`      | Unique identifier; serialized as a String      |

### Custom scalars

Declare a scalar and provide serialization logic in your server implementation.

```graphql
scalar DateTime
scalar JSON
```

The server must define how each custom scalar parses input and serializes output (via a resolver/config in the implementation).

---

## 3. Object Types & Fields

Object types are the core building blocks — a named collection of fields.

```graphql
type User {
  id: ID!
  name: String!
  email: String
  posts: [Post!]!
}
```

Fields may take **arguments**, optionally with default values:

```graphql
type Query {
  posts(first: Int = 10, after: String): [Post!]!
}
```

---

## 4. Type Modifiers: Non-Null & List

Modifiers compose to express nullability and collections.

| Notation  | Meaning                                             |
|-----------|-----------------------------------------------------|
| `String`  | Nullable string                                     |
| `String!` | Non-null string (never returns null)                |
| `[String]`| Nullable list of nullable strings                   |
| `[String!]`| Nullable list of non-null strings                  |
| `[String!]!`| Non-null list of non-null strings                 |

If a `!` field resolves to null, the error propagates up to the nearest nullable parent. Design nullability deliberately — over-using `!` can make partial responses fail entirely.

---

## 5. Enums

A fixed set of allowed values.

```graphql
enum Role {
  ADMIN
  EDITOR
  VIEWER
}

type User {
  role: Role!
}
```

Enum values are transmitted as their name strings. Servers may map them to internal values (e.g. integers).

---

## 6. Input Types

Object arguments (especially for mutations) use `input` types. Input fields may only be scalars, enums, or other input types — never object types.

```graphql
input CreateUserInput {
  name: String!
  email: String!
  role: Role = VIEWER
}

type Mutation {
  createUser(input: CreateUserInput!): User!
}
```

---

## 7. Interfaces

An interface defines fields that implementing object types must include.

```graphql
interface Node {
  id: ID!
}

type User implements Node {
  id: ID!
  name: String!
}

type Post implements Node {
  id: ID!
  title: String!
}
```

Query interface fields, then narrow with **inline fragments**:

```graphql
query {
  node(id: "1") {
    id
    ... on User { name }
    ... on Post { title }
  }
}
```

The server must supply a type resolver (`__resolveType` / `resolveType`) so it knows which concrete type each value is.

---

## 8. Unions

A union is one of several object types that need not share fields.

```graphql
union SearchResult = User | Post | Comment

type Query {
  search(term: String!): [SearchResult!]!
}
```

Consumers must use inline fragments per member type. Like interfaces, unions require a `resolveType`.

---

## 9. Directives

Directives annotate parts of a schema or query to change behavior. The spec defines these built-ins:

| Directive        | Location        | Purpose                                  |
|------------------|-----------------|------------------------------------------|
| `@deprecated`    | Schema (fields/enum values) | Mark as deprecated with a reason |
| `@skip(if:)`     | Query            | Omit a field when the condition is true  |
| `@include(if:)`  | Query            | Include a field only when true           |
| `@specifiedBy(url:)` | Scalar        | Point to a custom scalar's spec          |

```graphql
type User {
  id: ID!
  fullName: String!
  name: String! @deprecated(reason: "Use fullName")
}
```

Custom directives can be defined with `directive @auth(role: Role!) on FIELD_DEFINITION` and given behavior in the server implementation.

---

## 10. Documentation & Descriptions

Any type or field can carry a description using string literals — these surface in introspection and IDE tooling.

```graphql
"""
A registered user of the system.
"""
type User {
  "Primary key"
  id: ID!
}
```

---

## 11. Schema Design Tips

- **Design for the client's use cases**, not your database tables.
- Prefer **`input` objects** for mutation arguments — easier to evolve than positional args.
- Return a **payload type** from mutations (e.g. `CreateUserPayload`) so you can add fields (errors, metadata) later without breaking clients.
- **Evolve by addition**: add fields and mark old ones `@deprecated` rather than versioning the endpoint.
- Use **interfaces/`Node`** for consistent global identification and pagination.
- Be intentional about **nullability** — reserve `!` for values you can always guarantee.

---

## References

- Schema & types: https://graphql.org/learn/schema/
- Type system spec: https://spec.graphql.org/October2021/#sec-Type-System
- Schema design best practices: https://graphql.org/learn/best-practices/
- graphql-tools (makeExecutableSchema): https://the-guild.dev/graphql/tools/docs/generate-schema
