# GraphQL Resolvers

> Official Documentation: https://graphql.org/learn/execution/

## Overview

A **resolver** is a function that produces the value for a single field in the schema. During execution GraphQL walks the query field by field, calling the matching resolver for each. Together the resolvers form a **resolver map** that mirrors the schema's types.

Examples use the `graphql-js` (`graphql`) resolver signature, which is shared by most JS servers (Apollo Server, GraphQL Yoga, graphql-tools). This is spec-level behavior and framework-neutral.

---

## 1. The Resolver Map

Keys are type names; nested keys are field names; values are resolver functions.

```ts
const resolvers = {
  Query: {
    user: (parent, args, context, info) => context.db.user.find(args.id),
    users: (parent, args, context) => context.db.user.findAll(),
  },
  Mutation: {
    createUser: (parent, args, context) => context.db.user.create(args.input),
  },
  User: {
    posts: (parent, args, context) => context.db.post.byAuthor(parent.id),
  },
};
```

---

## 2. Resolver Arguments

Every resolver receives four positional arguments, in this order.

| Arg        | Name      | Description                                                        |
|------------|-----------|-------------------------------------------------------------------|
| 1st        | `parent`  | Result of the parent field's resolver (a.k.a. `root`/`source`).   |
| 2nd        | `args`    | The field's arguments from the query (`{ id: "1" }`).             |
| 3rd        | `context` | Per-request shared object (auth, data sources, loaders).          |
| 4th        | `info`    | AST/execution details: field name, path, selection set, schema.   |

```ts
const resolvers = {
  Query: {
    post: (parent, args, context, info) => {
      // parent  -> undefined for root fields
      // args    -> { id: "1" }
      // context -> { user, db, loaders }
      // info    -> { fieldName: "post", path, ... }
      return context.db.post.find(args.id);
    },
  },
};
```

`context` is created once per request by the server (see `controllers.md`) and is the right place for authentication, database clients, and DataLoaders.

---

## 3. Resolver Chains

Execution is hierarchical: a field's resolver return value becomes the `parent` of its child field resolvers. This lets each type own its own resolution logic.

```graphql
query {
  user(id: "1") {   # Query.user  -> returns a User
    name            # User.name   (parent = that User)
    posts {         # User.posts  (parent = that User) -> [Post]
      title         # Post.title  (parent = each Post)
    }
  }
}
```

For a list field, GraphQL runs the child resolvers once **per item** in the list.

---

## 4. Default Resolvers

If you don't define a resolver for a field, GraphQL uses a **default resolver**: it looks for a property on `parent` with the same name as the field (or calls it if it's a function). This is why leaf fields like `name` usually need no explicit resolver.

```ts
// No User.name resolver needed — the default returns parent.name
const resolvers = {
  Query: {
    user: () => ({ id: '1', name: 'Ada' }), // name resolved by default
  },
};
```

Define an explicit resolver only when a field needs computation, renaming, or a separate data fetch.

---

## 5. Async Resolvers

Resolvers may return a value **or a Promise**. GraphQL awaits promises automatically, so `async`/`await` and returning promises both work. Sibling fields resolve concurrently.

```ts
const resolvers = {
  Query: {
    user: async (parent, args, context) => {
      const user = await context.db.user.find(args.id);
      if (!user) throw new Error('User not found');
      return user;
    },
  },
  User: {
    posts: async (user, args, context) => context.db.post.byAuthor(user.id),
  },
};
```

Thrown errors (or rejected promises) are caught by the executor and added to the response's `errors` array; the field resolves to `null` (propagating per non-null rules).

---

## 6. The N+1 Problem & DataLoader

Because child resolvers run once per parent item, a naive nested query issues one query per item — the **N+1 problem**.

```graphql
query {
  posts {        # 1 query -> N posts
    author {     # N queries -> one per post!
      name
    }
  }
}
```

**DataLoader** (a third-party library, `dataloader`, maintained by GraphQL contributors) solves this by **batching** the individual loads within a tick into one call and **caching** by key for the request.

```ts
import DataLoader from 'dataloader'; // third-party

// Batch function: given an array of keys, return values in the SAME order.
function createLoaders(db) {
  return {
    userById: new DataLoader(async (ids: readonly string[]) => {
      const users = await db.user.findByIds(ids);
      const map = new Map(users.map((u) => [u.id, u]));
      return ids.map((id) => map.get(id) ?? null);
    }),
  };
}
```

Create loaders **per request** and attach them to `context`:

```ts
// when building context (see controllers.md)
context: async () => ({ db, loaders: createLoaders(db) }),

// resolver uses the loader instead of a direct query
const resolvers = {
  Post: {
    author: (post, _args, ctx) => ctx.loaders.userById.load(post.authorId),
  },
};
```

Now the N `author` lookups collapse into a single batched query. Loaders must be recreated per request so their cache never leaks data across users.

---

## 7. The `info` Argument

`info` carries execution metadata — the field name, the response `path`, the parent type, the return type, the full schema, and the query AST. It's advanced but powers features like:

- Computing which subfields were requested (to optimize DB `SELECT`s).
- Building projection/field-selection logic.
- Libraries that map GraphQL selections to SQL joins.

Most resolvers never touch `info`; reach for it only when you need selection-aware optimization.

---

## References

- Execution & resolvers: https://graphql.org/learn/execution/
- DataLoader: https://github.com/graphql/dataloader
- graphql-tools resolvers: https://the-guild.dev/graphql/tools/docs/resolvers
- graphql-js execution: https://github.com/graphql/graphql-js
