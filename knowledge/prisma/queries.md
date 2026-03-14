# Prisma Client Queries - Comprehensive Guide

This documentation covers all Prisma Client query operations, from basic CRUD to advanced patterns like transactions, aggregations, and raw queries.

---

## Table of Contents

1. [CRUD Operations](#crud-operations)
2. [Bulk Operations](#bulk-operations)
3. [Upsert](#upsert)
4. [Filtering](#filtering)
5. [Sorting](#sorting)
6. [Pagination](#pagination)
7. [Select and Include](#select-and-include)
8. [Nested Queries](#nested-queries)
9. [Aggregations](#aggregations)
10. [GroupBy](#groupby)
11. [Raw Queries](#raw-queries)
12. [Transactions](#transactions)
13. [Distinct](#distinct)
14. [Full-text Search](#full-text-search)
15. [JSON Field Queries](#json-field-queries)
16. [Type-safe Query Patterns](#type-safe-query-patterns)
17. [Best Practices](#best-practices)

---

## CRUD Operations

### Create

Create a single record in the database.

```typescript
// Basic create
const user = await prisma.user.create({
  data: {
    email: 'user@example.com',
    name: 'John Doe',
  },
});

// Create with all fields
const user = await prisma.user.create({
  data: {
    email: 'user@example.com',
    name: 'John Doe',
    age: 30,
    isActive: true,
    role: 'USER',
    createdAt: new Date(),
  },
});

// Create and return specific fields
const user = await prisma.user.create({
  data: {
    email: 'user@example.com',
    name: 'John Doe',
  },
  select: {
    id: true,
    email: true,
  },
});

// Create with nested relation (one-to-one)
const user = await prisma.user.create({
  data: {
    email: 'user@example.com',
    name: 'John Doe',
    profile: {
      create: {
        bio: 'Software developer',
        avatar: 'https://example.com/avatar.jpg',
      },
    },
  },
  include: {
    profile: true,
  },
});

// Create with nested relation (one-to-many)
const user = await prisma.user.create({
  data: {
    email: 'user@example.com',
    name: 'John Doe',
    posts: {
      create: [
        { title: 'First Post', content: 'Hello World' },
        { title: 'Second Post', content: 'Another post' },
      ],
    },
  },
  include: {
    posts: true,
  },
});

// Create and connect to existing relation
const post = await prisma.post.create({
  data: {
    title: 'New Post',
    content: 'Post content',
    author: {
      connect: { id: 1 },
    },
  },
});

// Create with connectOrCreate
const post = await prisma.post.create({
  data: {
    title: 'New Post',
    content: 'Post content',
    author: {
      connectOrCreate: {
        where: { email: 'author@example.com' },
        create: {
          email: 'author@example.com',
          name: 'New Author',
        },
      },
    },
  },
});
```

### FindUnique

Find a single record by unique identifier or unique field.

```typescript
// Find by primary key
const user = await prisma.user.findUnique({
  where: { id: 1 },
});

// Find by unique field
const user = await prisma.user.findUnique({
  where: { email: 'user@example.com' },
});

// Find by compound unique key
const subscription = await prisma.subscription.findUnique({
  where: {
    userId_planId: {
      userId: 1,
      planId: 'premium',
    },
  },
});

// FindUnique with include
const user = await prisma.user.findUnique({
  where: { id: 1 },
  include: {
    posts: true,
    profile: true,
  },
});

// FindUnique with select
const user = await prisma.user.findUnique({
  where: { id: 1 },
  select: {
    id: true,
    email: true,
    name: true,
  },
});

// FindUniqueOrThrow - throws if not found
const user = await prisma.user.findUniqueOrThrow({
  where: { id: 1 },
});
```

### FindFirst

Find the first record matching the criteria.

```typescript
// Find first matching record
const user = await prisma.user.findFirst({
  where: { isActive: true },
});

// Find first with ordering
const latestUser = await prisma.user.findFirst({
  where: { isActive: true },
  orderBy: { createdAt: 'desc' },
});

// Find first with complex conditions
const user = await prisma.user.findFirst({
  where: {
    AND: [
      { isActive: true },
      { age: { gte: 18 } },
      { email: { contains: '@company.com' } },
    ],
  },
});

// FindFirstOrThrow - throws if not found
const user = await prisma.user.findFirstOrThrow({
  where: { email: 'user@example.com' },
});
```

### FindMany

Retrieve multiple records.

```typescript
// Find all records
const users = await prisma.user.findMany();

// Find with conditions
const activeUsers = await prisma.user.findMany({
  where: { isActive: true },
});

// Find with ordering
const users = await prisma.user.findMany({
  orderBy: { createdAt: 'desc' },
});

// Find with multiple ordering
const users = await prisma.user.findMany({
  orderBy: [
    { role: 'asc' },
    { name: 'asc' },
  ],
});

// Find with pagination
const users = await prisma.user.findMany({
  skip: 0,
  take: 10,
});

// Find with select
const users = await prisma.user.findMany({
  select: {
    id: true,
    email: true,
    name: true,
  },
});

// Find with include
const users = await prisma.user.findMany({
  include: {
    posts: true,
    profile: true,
  },
});

// Complex query combining multiple options
const users = await prisma.user.findMany({
  where: {
    isActive: true,
    role: 'USER',
  },
  orderBy: { createdAt: 'desc' },
  skip: 20,
  take: 10,
  select: {
    id: true,
    email: true,
    name: true,
    posts: {
      where: { published: true },
      select: {
        id: true,
        title: true,
      },
    },
  },
});
```

### Update

Update a single record.

```typescript
// Basic update
const user = await prisma.user.update({
  where: { id: 1 },
  data: { name: 'Jane Doe' },
});

// Update multiple fields
const user = await prisma.user.update({
  where: { id: 1 },
  data: {
    name: 'Jane Doe',
    email: 'jane@example.com',
    updatedAt: new Date(),
  },
});

// Update with increment/decrement
const post = await prisma.post.update({
  where: { id: 1 },
  data: {
    viewCount: { increment: 1 },
  },
});

// Update with multiply/divide
const product = await prisma.product.update({
  where: { id: 1 },
  data: {
    price: { multiply: 1.1 }, // 10% increase
  },
});

// Update with decrement
const inventory = await prisma.inventory.update({
  where: { id: 1 },
  data: {
    quantity: { decrement: 5 },
  },
});

// Update nested relation
const user = await prisma.user.update({
  where: { id: 1 },
  data: {
    profile: {
      update: {
        bio: 'Updated bio',
      },
    },
  },
  include: {
    profile: true,
  },
});

// Update with upsert on nested relation
const user = await prisma.user.update({
  where: { id: 1 },
  data: {
    profile: {
      upsert: {
        create: { bio: 'New bio' },
        update: { bio: 'Updated bio' },
      },
    },
  },
});

// Update and return specific fields
const user = await prisma.user.update({
  where: { id: 1 },
  data: { name: 'Jane Doe' },
  select: {
    id: true,
    name: true,
    email: true,
  },
});
```

### Delete

Delete a single record.

```typescript
// Basic delete
const user = await prisma.user.delete({
  where: { id: 1 },
});

// Delete by unique field
const user = await prisma.user.delete({
  where: { email: 'user@example.com' },
});

// Delete and return deleted record
const deletedUser = await prisma.user.delete({
  where: { id: 1 },
  select: {
    id: true,
    email: true,
    name: true,
  },
});

// Delete with include (returns related data before deletion)
const deletedUser = await prisma.user.delete({
  where: { id: 1 },
  include: {
    posts: true,
  },
});
```

---

## Bulk Operations

### CreateMany

Create multiple records in a single operation.

```typescript
// Basic createMany
const result = await prisma.user.createMany({
  data: [
    { email: 'user1@example.com', name: 'User One' },
    { email: 'user2@example.com', name: 'User Two' },
    { email: 'user3@example.com', name: 'User Three' },
  ],
});
// Returns: { count: 3 }

// CreateMany with skipDuplicates
const result = await prisma.user.createMany({
  data: [
    { email: 'existing@example.com', name: 'Existing User' },
    { email: 'new@example.com', name: 'New User' },
  ],
  skipDuplicates: true, // Skips records with duplicate unique fields
});

// CreateMany does NOT support:
// - Nested creates
// - Returning created records (except count)
// - Select/include

// Alternative: createManyAndReturn (Prisma 5.14.0+)
const users = await prisma.user.createManyAndReturn({
  data: [
    { email: 'user1@example.com', name: 'User One' },
    { email: 'user2@example.com', name: 'User Two' },
  ],
});
// Returns array of created records

// createManyAndReturn with select
const users = await prisma.user.createManyAndReturn({
  data: [
    { email: 'user1@example.com', name: 'User One' },
    { email: 'user2@example.com', name: 'User Two' },
  ],
  select: {
    id: true,
    email: true,
  },
});
```

### UpdateMany

Update multiple records matching the criteria.

```typescript
// Update all matching records
const result = await prisma.user.updateMany({
  where: { isActive: false },
  data: { deletedAt: new Date() },
});
// Returns: { count: number }

// Update with complex conditions
const result = await prisma.post.updateMany({
  where: {
    AND: [
      { published: false },
      { createdAt: { lt: new Date('2023-01-01') } },
    ],
  },
  data: {
    archived: true,
  },
});

// Update all records (no where clause)
const result = await prisma.user.updateMany({
  data: { newsletter: false },
});

// UpdateMany with IN filter
const result = await prisma.user.updateMany({
  where: {
    id: { in: [1, 2, 3, 4, 5] },
  },
  data: {
    role: 'PREMIUM',
  },
});

// UpdateMany does NOT support:
// - Nested updates
// - Returning updated records
// - Select/include
```

### DeleteMany

Delete multiple records matching the criteria.

```typescript
// Delete all matching records
const result = await prisma.user.deleteMany({
  where: { isActive: false },
});
// Returns: { count: number }

// Delete with complex conditions
const result = await prisma.post.deleteMany({
  where: {
    AND: [
      { published: false },
      { createdAt: { lt: new Date('2022-01-01') } },
    ],
  },
});

// Delete all records (dangerous!)
const result = await prisma.user.deleteMany();
// Or explicitly:
const result = await prisma.user.deleteMany({
  where: {},
});

// Delete with relation filter
const result = await prisma.post.deleteMany({
  where: {
    author: {
      isActive: false,
    },
  },
});
```

---

## Upsert

Create a record if it doesn't exist, otherwise update it.

```typescript
// Basic upsert
const user = await prisma.user.upsert({
  where: { email: 'user@example.com' },
  update: {
    name: 'Updated Name',
    loginCount: { increment: 1 },
  },
  create: {
    email: 'user@example.com',
    name: 'New User',
    loginCount: 1,
  },
});

// Upsert with select
const user = await prisma.user.upsert({
  where: { email: 'user@example.com' },
  update: { lastLoginAt: new Date() },
  create: {
    email: 'user@example.com',
    name: 'New User',
    lastLoginAt: new Date(),
  },
  select: {
    id: true,
    email: true,
    lastLoginAt: true,
  },
});

// Upsert with include
const user = await prisma.user.upsert({
  where: { email: 'user@example.com' },
  update: { name: 'Updated Name' },
  create: {
    email: 'user@example.com',
    name: 'New User',
    profile: {
      create: { bio: 'Hello!' },
    },
  },
  include: {
    profile: true,
  },
});

// Upsert with nested operations
const user = await prisma.user.upsert({
  where: { email: 'user@example.com' },
  update: {
    profile: {
      upsert: {
        create: { bio: 'New bio' },
        update: { bio: 'Updated bio' },
      },
    },
  },
  create: {
    email: 'user@example.com',
    name: 'New User',
    profile: {
      create: { bio: 'New bio' },
    },
  },
});

// Upsert on compound unique key
const subscription = await prisma.subscription.upsert({
  where: {
    userId_planId: {
      userId: 1,
      planId: 'premium',
    },
  },
  update: {
    renewedAt: new Date(),
  },
  create: {
    userId: 1,
    planId: 'premium',
    startedAt: new Date(),
  },
});
```

---

## Filtering

### Basic Filters

```typescript
// Equals (implicit)
const users = await prisma.user.findMany({
  where: { email: 'test@example.com' },
});

// Equals (explicit)
const users = await prisma.user.findMany({
  where: { email: { equals: 'test@example.com' } },
});

// Not equals
const users = await prisma.user.findMany({
  where: { email: { not: 'admin@example.com' } },
});

// Null check
const users = await prisma.user.findMany({
  where: { deletedAt: null },
});

// Not null
const users = await prisma.user.findMany({
  where: { deletedAt: { not: null } },
});
```

### String Filters

```typescript
// Contains (case-sensitive by default)
const users = await prisma.user.findMany({
  where: { email: { contains: 'example' } },
});

// Contains (case-insensitive)
const users = await prisma.user.findMany({
  where: {
    email: {
      contains: 'example',
      mode: 'insensitive',
    },
  },
});

// Starts with
const users = await prisma.user.findMany({
  where: { email: { startsWith: 'admin' } },
});

// Ends with
const users = await prisma.user.findMany({
  where: { email: { endsWith: '.com' } },
});

// Combined string filters
const users = await prisma.user.findMany({
  where: {
    name: {
      contains: 'john',
      mode: 'insensitive',
    },
  },
});
```

### Number Filters

```typescript
// Greater than
const users = await prisma.user.findMany({
  where: { age: { gt: 18 } },
});

// Greater than or equal
const users = await prisma.user.findMany({
  where: { age: { gte: 18 } },
});

// Less than
const users = await prisma.user.findMany({
  where: { age: { lt: 65 } },
});

// Less than or equal
const users = await prisma.user.findMany({
  where: { age: { lte: 65 } },
});

// Range (combining gt and lt)
const users = await prisma.user.findMany({
  where: {
    age: {
      gte: 18,
      lte: 65,
    },
  },
});
```

### Array Filters (in, notIn)

```typescript
// In array
const users = await prisma.user.findMany({
  where: {
    status: { in: ['ACTIVE', 'PENDING'] },
  },
});

// Not in array
const users = await prisma.user.findMany({
  where: {
    status: { notIn: ['DELETED', 'BANNED'] },
  },
});

// In with IDs
const users = await prisma.user.findMany({
  where: {
    id: { in: [1, 2, 3, 4, 5] },
  },
});
```

### Logical Operators (AND, OR, NOT)

```typescript
// AND (implicit - multiple conditions)
const users = await prisma.user.findMany({
  where: {
    isActive: true,
    role: 'USER',
    age: { gte: 18 },
  },
});

// AND (explicit)
const users = await prisma.user.findMany({
  where: {
    AND: [
      { isActive: true },
      { role: 'USER' },
      { age: { gte: 18 } },
    ],
  },
});

// OR
const users = await prisma.user.findMany({
  where: {
    OR: [
      { email: { contains: 'gmail.com' } },
      { email: { contains: 'yahoo.com' } },
      { email: { contains: 'outlook.com' } },
    ],
  },
});

// NOT
const users = await prisma.user.findMany({
  where: {
    NOT: {
      email: { contains: 'spam' },
    },
  },
});

// NOT with array (none of these)
const users = await prisma.user.findMany({
  where: {
    NOT: [
      { role: 'ADMIN' },
      { isActive: false },
    ],
  },
});

// Complex nested conditions
const users = await prisma.user.findMany({
  where: {
    AND: [
      { isActive: true },
      {
        OR: [
          { role: 'ADMIN' },
          {
            AND: [
              { role: 'USER' },
              { age: { gte: 18 } },
            ],
          },
        ],
      },
    ],
  },
});
```

### Relation Filters

```typescript
// Filter by related record existence
const usersWithPosts = await prisma.user.findMany({
  where: {
    posts: {
      some: {}, // Has at least one post
    },
  },
});

// some - at least one related record matches
const users = await prisma.user.findMany({
  where: {
    posts: {
      some: {
        published: true,
      },
    },
  },
});

// every - all related records match
const users = await prisma.user.findMany({
  where: {
    posts: {
      every: {
        published: true,
      },
    },
  },
});

// none - no related records match
const users = await prisma.user.findMany({
  where: {
    posts: {
      none: {
        published: false,
      },
    },
  },
});

// Filter on one-to-one relation
const posts = await prisma.post.findMany({
  where: {
    author: {
      isActive: true,
      role: 'ADMIN',
    },
  },
});

// is / isNot for nullable relations
const users = await prisma.user.findMany({
  where: {
    profile: {
      is: null, // No profile
    },
  },
});

const users = await prisma.user.findMany({
  where: {
    profile: {
      isNot: null, // Has profile
    },
  },
});
```

---

## Sorting

### Basic Sorting

```typescript
// Sort ascending
const users = await prisma.user.findMany({
  orderBy: { name: 'asc' },
});

// Sort descending
const users = await prisma.user.findMany({
  orderBy: { createdAt: 'desc' },
});

// Multiple sort fields
const users = await prisma.user.findMany({
  orderBy: [
    { role: 'asc' },
    { name: 'asc' },
  ],
});

// Sort with nulls handling
const users = await prisma.user.findMany({
  orderBy: {
    lastLoginAt: { sort: 'desc', nulls: 'last' },
  },
});

// Nulls first
const users = await prisma.user.findMany({
  orderBy: {
    deletedAt: { sort: 'asc', nulls: 'first' },
  },
});
```

### Sorting by Relations

```typescript
// Sort by related field count
const users = await prisma.user.findMany({
  orderBy: {
    posts: {
      _count: 'desc',
    },
  },
});

// Sort by one-to-one relation field
const posts = await prisma.post.findMany({
  orderBy: {
    author: {
      name: 'asc',
    },
  },
});

// Complex sorting with relations
const users = await prisma.user.findMany({
  orderBy: [
    {
      posts: {
        _count: 'desc',
      },
    },
    { name: 'asc' },
  ],
  include: {
    _count: {
      select: { posts: true },
    },
  },
});
```

### Sorting by Aggregation

```typescript
// Order by aggregated related field
const categories = await prisma.category.findMany({
  orderBy: {
    products: {
      _count: 'desc',
    },
  },
});
```

---

## Pagination

### Offset Pagination (skip/take)

```typescript
// Basic pagination
const users = await prisma.user.findMany({
  skip: 0,
  take: 10,
});

// Page 2
const page2 = await prisma.user.findMany({
  skip: 10,
  take: 10,
});

// Pagination function
async function getUsers(page: number, pageSize: number) {
  const users = await prisma.user.findMany({
    skip: (page - 1) * pageSize,
    take: pageSize,
    orderBy: { createdAt: 'desc' },
  });

  const total = await prisma.user.count();

  return {
    data: users,
    meta: {
      total,
      page,
      pageSize,
      totalPages: Math.ceil(total / pageSize),
    },
  };
}

// Pagination with filtering
async function searchUsers(
  query: string,
  page: number,
  pageSize: number
) {
  const where = {
    OR: [
      { name: { contains: query, mode: 'insensitive' as const } },
      { email: { contains: query, mode: 'insensitive' as const } },
    ],
  };

  const [users, total] = await Promise.all([
    prisma.user.findMany({
      where,
      skip: (page - 1) * pageSize,
      take: pageSize,
      orderBy: { name: 'asc' },
    }),
    prisma.user.count({ where }),
  ]);

  return {
    data: users,
    meta: {
      total,
      page,
      pageSize,
      totalPages: Math.ceil(total / pageSize),
    },
  };
}
```

### Cursor-based Pagination

More efficient for large datasets and real-time data.

```typescript
// First page
const firstPage = await prisma.user.findMany({
  take: 10,
  orderBy: { id: 'asc' },
});

// Next page using cursor
const nextPage = await prisma.user.findMany({
  take: 10,
  skip: 1, // Skip the cursor record
  cursor: {
    id: firstPage[firstPage.length - 1].id,
  },
  orderBy: { id: 'asc' },
});

// Cursor pagination function
async function getUsersWithCursor(
  cursor?: number,
  pageSize: number = 10
) {
  const users = await prisma.user.findMany({
    take: pageSize + 1, // Take one extra to check if there's more
    ...(cursor && {
      skip: 1,
      cursor: { id: cursor },
    }),
    orderBy: { id: 'asc' },
  });

  const hasMore = users.length > pageSize;
  const data = hasMore ? users.slice(0, -1) : users;

  return {
    data,
    nextCursor: hasMore ? data[data.length - 1].id : null,
    hasMore,
  };
}

// Cursor with custom unique field
const posts = await prisma.post.findMany({
  take: 10,
  skip: 1,
  cursor: {
    slug: 'my-post-slug',
  },
  orderBy: { createdAt: 'desc' },
});

// Bidirectional cursor pagination
async function getUsersPage(
  cursor?: number,
  direction: 'forward' | 'backward' = 'forward',
  pageSize: number = 10
) {
  const take = direction === 'forward' ? pageSize : -pageSize;

  const users = await prisma.user.findMany({
    take,
    ...(cursor && {
      skip: 1,
      cursor: { id: cursor },
    }),
    orderBy: { id: 'asc' },
  });

  return direction === 'backward' ? users.reverse() : users;
}
```

---

## Select and Include

### Select

Choose specific fields to return.

```typescript
// Basic select
const users = await prisma.user.findMany({
  select: {
    id: true,
    email: true,
    name: true,
  },
});

// Select with relation
const users = await prisma.user.findMany({
  select: {
    id: true,
    name: true,
    posts: {
      select: {
        id: true,
        title: true,
      },
    },
  },
});

// Nested select with filters
const users = await prisma.user.findMany({
  select: {
    id: true,
    name: true,
    posts: {
      where: { published: true },
      select: {
        id: true,
        title: true,
        createdAt: true,
      },
      orderBy: { createdAt: 'desc' },
      take: 5,
    },
  },
});

// Select with count
const users = await prisma.user.findMany({
  select: {
    id: true,
    name: true,
    _count: {
      select: {
        posts: true,
        comments: true,
      },
    },
  },
});
```

### Include

Include related records in the result.

```typescript
// Basic include
const users = await prisma.user.findMany({
  include: {
    posts: true,
  },
});

// Multiple includes
const users = await prisma.user.findMany({
  include: {
    posts: true,
    profile: true,
    comments: true,
  },
});

// Include with filters
const users = await prisma.user.findMany({
  include: {
    posts: {
      where: { published: true },
      orderBy: { createdAt: 'desc' },
      take: 10,
    },
  },
});

// Nested includes
const users = await prisma.user.findMany({
  include: {
    posts: {
      include: {
        comments: {
          include: {
            author: true,
          },
        },
        categories: true,
      },
    },
  },
});

// Include with select inside
const users = await prisma.user.findMany({
  include: {
    posts: {
      select: {
        id: true,
        title: true,
      },
    },
  },
});

// Include count
const users = await prisma.user.findMany({
  include: {
    _count: {
      select: {
        posts: true,
        followers: true,
        following: true,
      },
    },
  },
});
```

### Select vs Include

```typescript
// You CANNOT use both select and include at the top level
// This will cause a type error:
// const users = await prisma.user.findMany({
//   select: { id: true },
//   include: { posts: true }, // Error!
// });

// Instead, use select for everything:
const users = await prisma.user.findMany({
  select: {
    id: true,
    name: true,
    posts: true, // This includes all post fields
  },
});

// Or use include and get all user fields:
const users = await prisma.user.findMany({
  include: {
    posts: {
      select: {
        id: true,
        title: true,
      },
    },
  },
});
```

---

## Nested Queries

### Create Nested Records

```typescript
// Create with nested create (one-to-one)
const user = await prisma.user.create({
  data: {
    email: 'user@example.com',
    name: 'John',
    profile: {
      create: {
        bio: 'Developer',
        website: 'https://example.com',
      },
    },
  },
});

// Create with nested create (one-to-many)
const user = await prisma.user.create({
  data: {
    email: 'user@example.com',
    name: 'John',
    posts: {
      create: [
        { title: 'Post 1', content: 'Content 1' },
        { title: 'Post 2', content: 'Content 2' },
      ],
    },
  },
});

// Create with nested createMany
const user = await prisma.user.create({
  data: {
    email: 'user@example.com',
    name: 'John',
    posts: {
      createMany: {
        data: [
          { title: 'Post 1', content: 'Content 1' },
          { title: 'Post 2', content: 'Content 2' },
        ],
      },
    },
  },
});

// Deep nested create
const user = await prisma.user.create({
  data: {
    email: 'user@example.com',
    name: 'John',
    posts: {
      create: {
        title: 'My Post',
        content: 'Content',
        comments: {
          create: [
            { content: 'Great post!', authorId: 2 },
            { content: 'Thanks!', authorId: 3 },
          ],
        },
      },
    },
  },
});
```

### Connect Existing Records

```typescript
// Connect to existing record (one-to-one)
const post = await prisma.post.create({
  data: {
    title: 'New Post',
    content: 'Content',
    author: {
      connect: { id: 1 },
    },
  },
});

// Connect by unique field
const post = await prisma.post.create({
  data: {
    title: 'New Post',
    content: 'Content',
    author: {
      connect: { email: 'author@example.com' },
    },
  },
});

// Connect multiple records (many-to-many)
const post = await prisma.post.create({
  data: {
    title: 'New Post',
    content: 'Content',
    authorId: 1,
    categories: {
      connect: [
        { id: 1 },
        { id: 2 },
        { slug: 'technology' },
      ],
    },
  },
});

// connectOrCreate
const post = await prisma.post.create({
  data: {
    title: 'New Post',
    content: 'Content',
    author: {
      connectOrCreate: {
        where: { email: 'author@example.com' },
        create: {
          email: 'author@example.com',
          name: 'New Author',
        },
      },
    },
  },
});
```

### Disconnect Records

```typescript
// Disconnect one-to-one (set to null)
const user = await prisma.user.update({
  where: { id: 1 },
  data: {
    profile: {
      disconnect: true,
    },
  },
});

// Disconnect specific records (many-to-many)
const post = await prisma.post.update({
  where: { id: 1 },
  data: {
    categories: {
      disconnect: [
        { id: 1 },
        { id: 2 },
      ],
    },
  },
});

// Disconnect all
const post = await prisma.post.update({
  where: { id: 1 },
  data: {
    categories: {
      set: [], // Removes all connections
    },
  },
});
```

### Set Relations

```typescript
// Set replaces all existing connections
const post = await prisma.post.update({
  where: { id: 1 },
  data: {
    categories: {
      set: [
        { id: 3 },
        { id: 4 },
      ],
    },
  },
});
```

### Update Nested Records

```typescript
// Update one-to-one
const user = await prisma.user.update({
  where: { id: 1 },
  data: {
    profile: {
      update: {
        bio: 'Updated bio',
      },
    },
  },
});

// Update specific nested record
const user = await prisma.user.update({
  where: { id: 1 },
  data: {
    posts: {
      update: {
        where: { id: 5 },
        data: { published: true },
      },
    },
  },
});

// Update many nested records
const user = await prisma.user.update({
  where: { id: 1 },
  data: {
    posts: {
      updateMany: {
        where: { published: false },
        data: { archived: true },
      },
    },
  },
});

// Upsert nested record
const user = await prisma.user.update({
  where: { id: 1 },
  data: {
    profile: {
      upsert: {
        create: { bio: 'New bio', website: '' },
        update: { bio: 'Updated bio' },
      },
    },
  },
});
```

### Delete Nested Records

```typescript
// Delete one-to-one
const user = await prisma.user.update({
  where: { id: 1 },
  data: {
    profile: {
      delete: true,
    },
  },
});

// Delete specific nested record
const user = await prisma.user.update({
  where: { id: 1 },
  data: {
    posts: {
      delete: { id: 5 },
    },
  },
});

// Delete multiple nested records
const user = await prisma.user.update({
  where: { id: 1 },
  data: {
    posts: {
      delete: [
        { id: 5 },
        { id: 6 },
      ],
    },
  },
});

// Delete many with condition
const user = await prisma.user.update({
  where: { id: 1 },
  data: {
    posts: {
      deleteMany: {
        published: false,
      },
    },
  },
});
```

---

## Aggregations

### Count

```typescript
// Count all records
const userCount = await prisma.user.count();

// Count with filter
const activeUserCount = await prisma.user.count({
  where: { isActive: true },
});

// Count with select (multiple counts)
const counts = await prisma.user.count({
  select: {
    _all: true, // Total count
    email: true, // Count non-null emails
    name: true, // Count non-null names
  },
});
```

### Aggregate

```typescript
// Single aggregation
const result = await prisma.product.aggregate({
  _avg: {
    price: true,
  },
});
// result._avg.price

// Multiple aggregations
const stats = await prisma.product.aggregate({
  _count: {
    _all: true,
  },
  _avg: {
    price: true,
    rating: true,
  },
  _sum: {
    quantity: true,
  },
  _min: {
    price: true,
    createdAt: true,
  },
  _max: {
    price: true,
    createdAt: true,
  },
});

// Aggregation with filter
const stats = await prisma.product.aggregate({
  where: {
    category: 'electronics',
    isAvailable: true,
  },
  _avg: {
    price: true,
  },
  _count: {
    _all: true,
  },
});

// Aggregation with ordering and pagination
const stats = await prisma.order.aggregate({
  where: {
    createdAt: {
      gte: new Date('2024-01-01'),
    },
  },
  _sum: {
    total: true,
  },
  _avg: {
    total: true,
  },
  orderBy: {
    createdAt: 'desc',
  },
  take: 100,
});
```

### Available Aggregation Functions

```typescript
// _count - Count records
const count = await prisma.user.aggregate({
  _count: {
    _all: true, // Count all records
    email: true, // Count non-null values
  },
});

// _avg - Average (numeric fields only)
const avg = await prisma.product.aggregate({
  _avg: { price: true },
});

// _sum - Sum (numeric fields only)
const sum = await prisma.order.aggregate({
  _sum: { total: true },
});

// _min - Minimum value
const min = await prisma.product.aggregate({
  _min: {
    price: true,
    createdAt: true, // Works with dates
  },
});

// _max - Maximum value
const max = await prisma.product.aggregate({
  _max: {
    price: true,
    createdAt: true,
  },
});
```

---

## GroupBy

Group records and perform aggregations on each group.

```typescript
// Basic groupBy
const groupedUsers = await prisma.user.groupBy({
  by: ['role'],
  _count: {
    _all: true,
  },
});
// [{ role: 'USER', _count: { _all: 50 } }, { role: 'ADMIN', _count: { _all: 5 } }]

// GroupBy multiple fields
const grouped = await prisma.order.groupBy({
  by: ['status', 'paymentMethod'],
  _count: {
    _all: true,
  },
  _sum: {
    total: true,
  },
});

// GroupBy with filtering (where)
const grouped = await prisma.order.groupBy({
  by: ['status'],
  where: {
    createdAt: {
      gte: new Date('2024-01-01'),
    },
  },
  _count: {
    _all: true,
  },
  _sum: {
    total: true,
  },
});

// GroupBy with having (filter after grouping)
const grouped = await prisma.user.groupBy({
  by: ['country'],
  _count: {
    id: true,
  },
  having: {
    id: {
      _count: {
        gt: 100, // Only countries with more than 100 users
      },
    },
  },
});

// GroupBy with ordering
const grouped = await prisma.product.groupBy({
  by: ['category'],
  _count: {
    _all: true,
  },
  _avg: {
    price: true,
  },
  orderBy: {
    _count: {
      _all: 'desc',
    },
  },
});

// GroupBy with take/skip
const topCategories = await prisma.product.groupBy({
  by: ['category'],
  _count: {
    _all: true,
  },
  orderBy: {
    _count: {
      _all: 'desc',
    },
  },
  take: 5,
});

// Complex groupBy example
const salesReport = await prisma.order.groupBy({
  by: ['status', 'createdAt'],
  where: {
    createdAt: {
      gte: new Date('2024-01-01'),
      lte: new Date('2024-12-31'),
    },
  },
  _count: {
    _all: true,
  },
  _sum: {
    total: true,
    tax: true,
  },
  _avg: {
    total: true,
  },
  having: {
    total: {
      _sum: {
        gt: 1000,
      },
    },
  },
  orderBy: [
    { createdAt: 'asc' },
    {
      _sum: {
        total: 'desc',
      },
    },
  ],
});
```

---

## Raw Queries

Execute raw SQL when Prisma's query API doesn't cover your needs.

### $queryRaw

Execute raw SELECT queries and return results.

```typescript
// Basic raw query
const users = await prisma.$queryRaw`
  SELECT * FROM "User" WHERE "isActive" = true
`;

// With parameters (safe from SQL injection)
const email = 'user@example.com';
const users = await prisma.$queryRaw`
  SELECT * FROM "User" WHERE "email" = ${email}
`;

// With multiple parameters
const minAge = 18;
const maxAge = 65;
const users = await prisma.$queryRaw`
  SELECT * FROM "User"
  WHERE "age" >= ${minAge} AND "age" <= ${maxAge}
`;

// Type-safe raw query
interface UserResult {
  id: number;
  email: string;
  name: string | null;
}

const users = await prisma.$queryRaw<UserResult[]>`
  SELECT id, email, name FROM "User" WHERE "isActive" = true
`;

// Using Prisma.sql for dynamic queries
import { Prisma } from '@prisma/client';

const columns = Prisma.sql`id, email, name`;
const users = await prisma.$queryRaw`
  SELECT ${columns} FROM "User"
`;

// Dynamic table name (use with caution)
const tableName = Prisma.sql`"User"`;
const users = await prisma.$queryRaw`
  SELECT * FROM ${tableName}
`;

// Join query
const postsWithAuthors = await prisma.$queryRaw`
  SELECT
    p.id,
    p.title,
    p.content,
    u.name as author_name,
    u.email as author_email
  FROM "Post" p
  JOIN "User" u ON p."authorId" = u.id
  WHERE p.published = true
`;

// Aggregate raw query
const stats = await prisma.$queryRaw`
  SELECT
    COUNT(*) as total,
    AVG(price) as avg_price,
    SUM(quantity) as total_quantity
  FROM "Product"
  WHERE "categoryId" = ${categoryId}
`;
```

### $executeRaw

Execute raw INSERT, UPDATE, DELETE statements.

```typescript
// Insert
const result = await prisma.$executeRaw`
  INSERT INTO "User" (email, name) VALUES (${email}, ${name})
`;
// Returns number of affected rows

// Update
const updatedCount = await prisma.$executeRaw`
  UPDATE "User" SET "isActive" = false WHERE "lastLoginAt" < ${cutoffDate}
`;

// Delete
const deletedCount = await prisma.$executeRaw`
  DELETE FROM "Session" WHERE "expiresAt" < NOW()
`;

// Bulk operations
const affected = await prisma.$executeRaw`
  UPDATE "Product"
  SET price = price * 1.1
  WHERE "categoryId" = ${categoryId}
`;
```

### $queryRawUnsafe and $executeRawUnsafe

Use when you need fully dynamic queries (BE CAREFUL - SQL injection risk!).

```typescript
// WARNING: Never use with user input directly!
const tableName = 'User'; // Must be validated/sanitized
const users = await prisma.$queryRawUnsafe(
  `SELECT * FROM "${tableName}" WHERE "isActive" = $1`,
  true
);

// With parameterized values (safer)
const result = await prisma.$queryRawUnsafe(
  'SELECT * FROM "User" WHERE email = $1 AND age > $2',
  email,
  minAge
);

// Dynamic column selection (validate columns first!)
const allowedColumns = ['id', 'email', 'name'];
const requestedColumns = ['id', 'email'];
const safeColumns = requestedColumns.filter(c => allowedColumns.includes(c));
const columnList = safeColumns.join(', ');

const users = await prisma.$queryRawUnsafe(
  `SELECT ${columnList} FROM "User"`
);
```

---

## Transactions

### Sequential Transactions

Execute multiple operations in sequence, all or nothing.

```typescript
// Using $transaction with array
const [user, post] = await prisma.$transaction([
  prisma.user.create({
    data: { email: 'user@example.com', name: 'John' },
  }),
  prisma.post.create({
    data: { title: 'Hello', content: 'World', authorId: 1 },
  }),
]);

// Multiple operations
const result = await prisma.$transaction([
  prisma.user.deleteMany({ where: { isActive: false } }),
  prisma.post.deleteMany({ where: { authorId: { in: [1, 2, 3] } } }),
  prisma.comment.deleteMany({ where: { postId: { in: [1, 2, 3] } } }),
]);
```

### Interactive Transactions

More flexible transactions with business logic.

```typescript
// Basic interactive transaction
const result = await prisma.$transaction(async (tx) => {
  // Create user
  const user = await tx.user.create({
    data: { email: 'user@example.com', name: 'John' },
  });

  // Create profile using the new user's ID
  const profile = await tx.profile.create({
    data: { userId: user.id, bio: 'Hello!' },
  });

  return { user, profile };
});

// Transaction with conditional logic
const transfer = await prisma.$transaction(async (tx) => {
  // Get source account
  const source = await tx.account.findUnique({
    where: { id: sourceId },
  });

  if (!source || source.balance < amount) {
    throw new Error('Insufficient funds');
  }

  // Deduct from source
  await tx.account.update({
    where: { id: sourceId },
    data: { balance: { decrement: amount } },
  });

  // Add to destination
  await tx.account.update({
    where: { id: destinationId },
    data: { balance: { increment: amount } },
  });

  // Create transaction record
  return tx.transaction.create({
    data: {
      sourceId,
      destinationId,
      amount,
      type: 'TRANSFER',
    },
  });
});

// Transaction with error handling
async function createOrder(userId: number, items: OrderItem[]) {
  try {
    return await prisma.$transaction(async (tx) => {
      // Check inventory
      for (const item of items) {
        const product = await tx.product.findUnique({
          where: { id: item.productId },
        });

        if (!product || product.stock < item.quantity) {
          throw new Error(`Insufficient stock for product ${item.productId}`);
        }
      }

      // Create order
      const order = await tx.order.create({
        data: {
          userId,
          status: 'PENDING',
          items: {
            create: items.map(item => ({
              productId: item.productId,
              quantity: item.quantity,
              price: item.price,
            })),
          },
        },
      });

      // Update inventory
      for (const item of items) {
        await tx.product.update({
          where: { id: item.productId },
          data: { stock: { decrement: item.quantity } },
        });
      }

      return order;
    });
  } catch (error) {
    console.error('Order creation failed:', error);
    throw error;
  }
}
```

### Transaction Options

```typescript
// With timeout
const result = await prisma.$transaction(
  async (tx) => {
    // Long running operations
  },
  {
    maxWait: 5000, // Max time to wait for transaction slot (ms)
    timeout: 10000, // Max transaction duration (ms)
  }
);

// With isolation level
const result = await prisma.$transaction(
  async (tx) => {
    // Operations requiring specific isolation
  },
  {
    isolationLevel: Prisma.TransactionIsolationLevel.Serializable,
  }
);

// Available isolation levels:
// - ReadUncommitted
// - ReadCommitted
// - RepeatableRead
// - Serializable
```

### Nested Transactions

```typescript
// Prisma handles nested transactions with savepoints
const result = await prisma.$transaction(async (tx) => {
  const user = await tx.user.create({
    data: { email: 'user@example.com', name: 'John' },
  });

  try {
    // This creates a savepoint
    await tx.post.create({
      data: { title: 'Post', authorId: user.id },
    });
  } catch (error) {
    // Savepoint is rolled back, but outer transaction continues
    console.log('Post creation failed, continuing...');
  }

  return user;
});
```

---

## Distinct

Return unique values for specified fields.

```typescript
// Distinct on single field
const countries = await prisma.user.findMany({
  distinct: ['country'],
  select: {
    country: true,
  },
});

// Distinct on multiple fields
const uniqueCombinations = await prisma.user.findMany({
  distinct: ['country', 'city'],
  select: {
    country: true,
    city: true,
  },
});

// Distinct with filtering
const activeCountries = await prisma.user.findMany({
  where: { isActive: true },
  distinct: ['country'],
  select: {
    country: true,
  },
});

// Distinct with ordering
const countries = await prisma.user.findMany({
  distinct: ['country'],
  orderBy: { country: 'asc' },
  select: {
    country: true,
  },
});

// Distinct with count (alternative using groupBy)
const countryCounts = await prisma.user.groupBy({
  by: ['country'],
  _count: {
    _all: true,
  },
  orderBy: {
    _count: {
      _all: 'desc',
    },
  },
});
```

---

## Full-text Search

Full-text search is available for PostgreSQL and MySQL.

### PostgreSQL Full-text Search

```prisma
// schema.prisma - Enable preview feature
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["fullTextSearch"]
}
```

```typescript
// Basic search
const posts = await prisma.post.findMany({
  where: {
    title: {
      search: 'database',
    },
  },
});

// Search multiple fields
const posts = await prisma.post.findMany({
  where: {
    OR: [
      { title: { search: 'prisma' } },
      { content: { search: 'prisma' } },
    ],
  },
});

// Search with AND (all words must match)
const posts = await prisma.post.findMany({
  where: {
    title: {
      search: 'prisma & database', // PostgreSQL tsquery syntax
    },
  },
});

// Search with OR (any word matches)
const posts = await prisma.post.findMany({
  where: {
    title: {
      search: 'prisma | sequelize | typeorm',
    },
  },
});

// Negation (exclude words)
const posts = await prisma.post.findMany({
  where: {
    title: {
      search: 'database & !sql',
    },
  },
});

// Prefix matching
const posts = await prisma.post.findMany({
  where: {
    title: {
      search: 'data:*', // Matches "data", "database", "datastore", etc.
    },
  },
});

// Relevance ordering with raw query
const posts = await prisma.$queryRaw`
  SELECT
    id,
    title,
    content,
    ts_rank(to_tsvector('english', title || ' ' || content), plainto_tsquery('english', ${query})) as rank
  FROM "Post"
  WHERE to_tsvector('english', title || ' ' || content) @@ plainto_tsquery('english', ${query})
  ORDER BY rank DESC
`;
```

### MySQL Full-text Search

```prisma
// schema.prisma - Enable preview feature
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["fullTextSearch", "fullTextIndex"]
}

model Post {
  id      Int    @id @default(autoincrement())
  title   String @db.VarChar(255)
  content String @db.Text

  @@fulltext([title, content])
}
```

```typescript
// Basic search
const posts = await prisma.post.findMany({
  where: {
    title: {
      search: 'database',
    },
  },
});

// Search across indexed fields
const posts = await prisma.post.findMany({
  where: {
    title: {
      search: '+prisma -sequelize', // Boolean mode syntax
    },
  },
});
```

---

## JSON Field Queries

Query JSON fields in PostgreSQL, MySQL, and other databases that support JSON.

### Reading JSON Fields

```typescript
// Model with JSON field
// model User {
//   id       Int  @id
//   settings Json
// }

// Get all users
const users = await prisma.user.findMany();
// settings is automatically parsed as JavaScript object

// Access nested JSON data
const user = await prisma.user.findUnique({
  where: { id: 1 },
});
console.log(user.settings.theme); // Access JSON property
```

### Filtering JSON Fields (PostgreSQL)

```typescript
// Filter by JSON property value
const users = await prisma.user.findMany({
  where: {
    settings: {
      path: ['theme'],
      equals: 'dark',
    },
  },
});

// Nested path
const users = await prisma.user.findMany({
  where: {
    settings: {
      path: ['notifications', 'email'],
      equals: true,
    },
  },
});

// String contains in JSON
const users = await prisma.user.findMany({
  where: {
    settings: {
      path: ['bio'],
      string_contains: 'developer',
    },
  },
});

// String starts with
const users = await prisma.user.findMany({
  where: {
    settings: {
      path: ['website'],
      string_starts_with: 'https://',
    },
  },
});

// String ends with
const users = await prisma.user.findMany({
  where: {
    settings: {
      path: ['email'],
      string_ends_with: '@company.com',
    },
  },
});

// Array contains
const users = await prisma.user.findMany({
  where: {
    settings: {
      path: ['roles'],
      array_contains: ['admin'],
    },
  },
});

// Array starts with
const users = await prisma.user.findMany({
  where: {
    settings: {
      path: ['tags'],
      array_starts_with: ['featured'],
    },
  },
});

// Array ends with
const users = await prisma.user.findMany({
  where: {
    settings: {
      path: ['tags'],
      array_ends_with: ['pinned'],
    },
  },
});
```

### Updating JSON Fields

```typescript
// Replace entire JSON field
await prisma.user.update({
  where: { id: 1 },
  data: {
    settings: {
      theme: 'dark',
      notifications: {
        email: true,
        push: false,
      },
    },
  },
});

// Update specific JSON path (PostgreSQL with raw query)
await prisma.$executeRaw`
  UPDATE "User"
  SET settings = jsonb_set(settings, '{theme}', '"light"')
  WHERE id = ${userId}
`;

// Update nested JSON path
await prisma.$executeRaw`
  UPDATE "User"
  SET settings = jsonb_set(settings, '{notifications,email}', 'false')
  WHERE id = ${userId}
`;
```

### JSON with Prisma Types

```typescript
import { Prisma } from '@prisma/client';

// Define type for JSON field
type UserSettings = {
  theme: 'light' | 'dark';
  language: string;
  notifications: {
    email: boolean;
    push: boolean;
  };
};

// Create with typed JSON
await prisma.user.create({
  data: {
    email: 'user@example.com',
    settings: {
      theme: 'dark',
      language: 'en',
      notifications: {
        email: true,
        push: false,
      },
    } satisfies UserSettings,
  },
});

// Use InputJsonValue for type safety
const settings: Prisma.InputJsonValue = {
  theme: 'dark',
  notifications: { email: true },
};

await prisma.user.update({
  where: { id: 1 },
  data: { settings },
});
```

---

## Type-safe Query Patterns

### Dynamic Query Building

```typescript
import { Prisma } from '@prisma/client';

// Build where clause dynamically
function buildUserFilter(params: {
  search?: string;
  role?: string;
  isActive?: boolean;
}): Prisma.UserWhereInput {
  const where: Prisma.UserWhereInput = {};

  if (params.search) {
    where.OR = [
      { name: { contains: params.search, mode: 'insensitive' } },
      { email: { contains: params.search, mode: 'insensitive' } },
    ];
  }

  if (params.role) {
    where.role = params.role;
  }

  if (params.isActive !== undefined) {
    where.isActive = params.isActive;
  }

  return where;
}

// Usage
const users = await prisma.user.findMany({
  where: buildUserFilter({ search: 'john', isActive: true }),
});
```

### Reusable Query Components

```typescript
import { Prisma } from '@prisma/client';

// Reusable select objects
const userBasicSelect = {
  id: true,
  email: true,
  name: true,
} satisfies Prisma.UserSelect;

const userWithPostsSelect = {
  ...userBasicSelect,
  posts: {
    select: {
      id: true,
      title: true,
    },
  },
} satisfies Prisma.UserSelect;

// Reusable include objects
const postWithAuthorInclude = {
  author: {
    select: userBasicSelect,
  },
} satisfies Prisma.PostInclude;

// Usage
const users = await prisma.user.findMany({
  select: userWithPostsSelect,
});

const posts = await prisma.post.findMany({
  include: postWithAuthorInclude,
});
```

### Validated Enum Filters

```typescript
import { Prisma } from '@prisma/client';

// Use Prisma generated enums
const status: Prisma.EnumUserStatusFilter = {
  in: ['ACTIVE', 'PENDING'],
};

const users = await prisma.user.findMany({
  where: { status },
});

// Type-safe ordering
const orderBy: Prisma.UserOrderByWithRelationInput = {
  createdAt: 'desc',
};

const users = await prisma.user.findMany({ orderBy });
```

### Generic Repository Pattern

```typescript
import { Prisma, PrismaClient } from '@prisma/client';

// Generic pagination helper
async function paginate<T, A>(
  model: {
    findMany: (args: A) => Promise<T[]>;
    count: (args: { where?: any }) => Promise<number>;
  },
  args: A & { where?: any },
  options: { page: number; pageSize: number }
) {
  const { page, pageSize } = options;

  const [data, total] = await Promise.all([
    model.findMany({
      ...args,
      skip: (page - 1) * pageSize,
      take: pageSize,
    } as A),
    model.count({ where: (args as any).where }),
  ]);

  return {
    data,
    meta: {
      total,
      page,
      pageSize,
      totalPages: Math.ceil(total / pageSize),
      hasNext: page * pageSize < total,
      hasPrev: page > 1,
    },
  };
}

// Usage
const result = await paginate(
  prisma.user,
  {
    where: { isActive: true },
    orderBy: { createdAt: 'desc' },
    include: { posts: true },
  },
  { page: 1, pageSize: 10 }
);
```

### Conditional Includes

```typescript
// Include based on condition
async function getUser(id: number, includePosts: boolean) {
  return prisma.user.findUnique({
    where: { id },
    include: {
      profile: true,
      ...(includePosts && {
        posts: {
          where: { published: true },
          take: 10,
        },
      }),
    },
  });
}

// Dynamic select based on fields requested
async function getUsers(fields: string[]) {
  const select: Prisma.UserSelect = {};

  const allowedFields = ['id', 'email', 'name', 'createdAt'];
  for (const field of fields) {
    if (allowedFields.includes(field)) {
      select[field as keyof Prisma.UserSelect] = true;
    }
  }

  return prisma.user.findMany({ select });
}
```

---

## Best Practices

### 1. Connection Management

```typescript
// Singleton pattern for PrismaClient
import { PrismaClient } from '@prisma/client';

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined;
};

export const prisma = globalForPrisma.prisma ?? new PrismaClient();

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma;
}

// Proper cleanup
process.on('beforeExit', async () => {
  await prisma.$disconnect();
});
```

### 2. Error Handling

```typescript
import { Prisma } from '@prisma/client';

async function createUser(email: string, name: string) {
  try {
    return await prisma.user.create({
      data: { email, name },
    });
  } catch (error) {
    if (error instanceof Prisma.PrismaClientKnownRequestError) {
      // P2002: Unique constraint violation
      if (error.code === 'P2002') {
        throw new Error('A user with this email already exists');
      }
      // P2025: Record not found
      if (error.code === 'P2025') {
        throw new Error('Record not found');
      }
    }
    throw error;
  }
}

// Common error codes:
// P2002 - Unique constraint failed
// P2003 - Foreign key constraint failed
// P2025 - Record not found
// P2014 - Required relation violation
// P2021 - Table does not exist
// P2022 - Column does not exist
```

### 3. Soft Deletes

```typescript
// Middleware approach
prisma.$use(async (params, next) => {
  // Intercept delete and convert to soft delete
  if (params.model === 'User') {
    if (params.action === 'delete') {
      params.action = 'update';
      params.args['data'] = { deletedAt: new Date() };
    }
    if (params.action === 'deleteMany') {
      params.action = 'updateMany';
      params.args['data'] = { deletedAt: new Date() };
    }
  }

  // Auto-filter soft deleted records
  if (params.model === 'User') {
    if (params.action === 'findUnique' || params.action === 'findFirst') {
      params.action = 'findFirst';
      params.args.where = { ...params.args.where, deletedAt: null };
    }
    if (params.action === 'findMany') {
      params.args.where = { ...params.args.where, deletedAt: null };
    }
  }

  return next(params);
});
```

### 4. Logging and Debugging

```typescript
// Enable query logging
const prisma = new PrismaClient({
  log: [
    { level: 'query', emit: 'event' },
    { level: 'error', emit: 'stdout' },
    { level: 'warn', emit: 'stdout' },
  ],
});

// Log all queries
prisma.$on('query', (e) => {
  console.log('Query:', e.query);
  console.log('Params:', e.params);
  console.log('Duration:', e.duration, 'ms');
});

// Conditional logging
const prisma = new PrismaClient({
  log: process.env.NODE_ENV === 'development'
    ? ['query', 'info', 'warn', 'error']
    : ['error'],
});
```

### 5. Optimizing Queries

```typescript
// Avoid N+1 queries - use include/select
// Bad:
const users = await prisma.user.findMany();
for (const user of users) {
  const posts = await prisma.post.findMany({
    where: { authorId: user.id },
  });
}

// Good:
const users = await prisma.user.findMany({
  include: { posts: true },
});

// Select only needed fields
// Bad:
const users = await prisma.user.findMany(); // Returns all fields

// Good:
const users = await prisma.user.findMany({
  select: {
    id: true,
    email: true,
    name: true,
  },
});

// Use cursor pagination for large datasets
// Bad for large tables:
const users = await prisma.user.findMany({
  skip: 10000,
  take: 10,
});

// Better:
const users = await prisma.user.findMany({
  cursor: { id: lastSeenId },
  skip: 1,
  take: 10,
});
```

### 6. Batch Operations

```typescript
// Batch reads with Promise.all
const [users, posts, comments] = await Promise.all([
  prisma.user.findMany({ where: { isActive: true } }),
  prisma.post.findMany({ where: { published: true } }),
  prisma.comment.findMany({ where: { approved: true } }),
]);

// Use createMany for bulk inserts
await prisma.user.createMany({
  data: users,
  skipDuplicates: true,
});

// Use transactions for related operations
await prisma.$transaction([
  prisma.user.updateMany({
    where: { role: 'GUEST' },
    data: { role: 'USER' },
  }),
  prisma.auditLog.create({
    data: { action: 'BULK_ROLE_UPDATE' },
  }),
]);
```

### 7. Type Safety with Validators

```typescript
import { Prisma } from '@prisma/client';

// Validate input matches schema
function validateUserCreate(data: unknown): Prisma.UserCreateInput {
  // Use zod or similar for runtime validation
  const validated = userCreateSchema.parse(data);
  return validated;
}

// Type-safe update data
type UserUpdateData = Prisma.UserUpdateInput;

function updateUser(id: number, data: UserUpdateData) {
  return prisma.user.update({
    where: { id },
    data,
  });
}
```

### 8. Testing with Prisma

```typescript
// Use transactions for test isolation
beforeEach(async () => {
  await prisma.$transaction([
    prisma.comment.deleteMany(),
    prisma.post.deleteMany(),
    prisma.user.deleteMany(),
  ]);
});

// Or use $executeRaw for faster cleanup
beforeEach(async () => {
  await prisma.$executeRaw`TRUNCATE TABLE "User" CASCADE`;
});

// Factory functions for test data
async function createTestUser(overrides: Partial<Prisma.UserCreateInput> = {}) {
  return prisma.user.create({
    data: {
      email: `test-${Date.now()}@example.com`,
      name: 'Test User',
      ...overrides,
    },
  });
}
```

### 9. Security Considerations

```typescript
// Never interpolate user input in raw queries
// Bad:
const users = await prisma.$queryRawUnsafe(
  `SELECT * FROM "User" WHERE name = '${userInput}'` // SQL injection!
);

// Good:
const users = await prisma.$queryRaw`
  SELECT * FROM "User" WHERE name = ${userInput}
`;

// Validate and sanitize all user inputs
// Use Prisma's type system to enforce schema constraints
// Implement row-level security where needed
```

### 10. Performance Monitoring

```typescript
// Track query performance
const prisma = new PrismaClient({
  log: [{ level: 'query', emit: 'event' }],
});

const slowQueryThreshold = 100; // ms

prisma.$on('query', (e) => {
  if (e.duration > slowQueryThreshold) {
    console.warn(`Slow query (${e.duration}ms):`, e.query);
    // Send to monitoring service
  }
});

// Use connection pooling appropriately
const prisma = new PrismaClient({
  datasources: {
    db: {
      url: process.env.DATABASE_URL,
    },
  },
});
```

---

## Quick Reference

### Query Methods

| Method | Description | Returns |
|--------|-------------|---------|
| `findUnique` | Find by unique field | Record or null |
| `findUniqueOrThrow` | Find by unique field | Record (throws if not found) |
| `findFirst` | Find first matching | Record or null |
| `findFirstOrThrow` | Find first matching | Record (throws if not found) |
| `findMany` | Find all matching | Array of records |
| `create` | Create one record | Created record |
| `createMany` | Create multiple records | Count |
| `createManyAndReturn` | Create multiple records | Array of created records |
| `update` | Update one record | Updated record |
| `updateMany` | Update multiple records | Count |
| `upsert` | Create or update | Record |
| `delete` | Delete one record | Deleted record |
| `deleteMany` | Delete multiple records | Count |
| `count` | Count records | Number |
| `aggregate` | Aggregate values | Aggregation result |
| `groupBy` | Group and aggregate | Array of groups |

### Filter Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `equals` | Exact match | `{ email: { equals: 'a@b.com' } }` |
| `not` | Not equal | `{ email: { not: 'a@b.com' } }` |
| `in` | In array | `{ id: { in: [1, 2, 3] } }` |
| `notIn` | Not in array | `{ id: { notIn: [1, 2] } }` |
| `lt` | Less than | `{ age: { lt: 18 } }` |
| `lte` | Less than or equal | `{ age: { lte: 18 } }` |
| `gt` | Greater than | `{ age: { gt: 18 } }` |
| `gte` | Greater than or equal | `{ age: { gte: 18 } }` |
| `contains` | Contains string | `{ name: { contains: 'john' } }` |
| `startsWith` | Starts with | `{ name: { startsWith: 'J' } }` |
| `endsWith` | Ends with | `{ email: { endsWith: '.com' } }` |
| `AND` | All conditions true | `{ AND: [{...}, {...}] }` |
| `OR` | Any condition true | `{ OR: [{...}, {...}] }` |
| `NOT` | Negation | `{ NOT: {...} }` |

### Relation Filters

| Operator | Description |
|----------|-------------|
| `some` | At least one related record matches |
| `every` | All related records match |
| `none` | No related records match |
| `is` | Related record matches (nullable) |
| `isNot` | Related record doesn't match |

### Update Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `increment` | Add to number | `{ views: { increment: 1 } }` |
| `decrement` | Subtract from number | `{ stock: { decrement: 5 } }` |
| `multiply` | Multiply number | `{ price: { multiply: 1.1 } }` |
| `divide` | Divide number | `{ price: { divide: 2 } }` |
| `set` | Set value | `{ tags: { set: ['a', 'b'] } }` |
| `push` | Add to array | `{ tags: { push: 'new' } }` |

---

This documentation covers the essential Prisma Client query operations. For the most up-to-date information and additional features, always refer to the official Prisma documentation at https://www.prisma.io/docs.
