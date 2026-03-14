# Prisma Relations

This comprehensive guide covers all aspects of defining and working with relations in Prisma Schema and Prisma Client. Relations represent connections between models and are fundamental to building relational data structures.

---

## Table of Contents

1. [One-to-One Relations](#1-one-to-one-relations)
2. [One-to-Many Relations](#2-one-to-many-relations)
3. [Many-to-Many Relations](#3-many-to-many-relations)
4. [Self-Relations](#4-self-relations)
5. [@relation Attribute](#5-relation-attribute)
6. [Referential Actions](#6-referential-actions)
7. [Optional vs Required Relations](#7-optional-vs-required-relations)
8. [Named Relations](#8-named-relations)
9. [Multiple Relations Between Same Models](#9-multiple-relations-between-same-models)
10. [Querying Relations](#10-querying-relations)
11. [Nested Writes](#11-nested-writes)
12. [Filtering on Relations](#12-filtering-on-relations)
13. [Ordering by Relations](#13-ordering-by-relations)
14. [Counting Relations](#14-counting-relations)
15. [Relation Field Names and Foreign Keys](#15-relation-field-names-and-foreign-keys)
16. [Best Practices](#16-best-practices)

---

## 1. One-to-One Relations

A one-to-one (1-1) relation means that one record on each side of the relation can only be connected to one record on the other side. In Prisma, one side of the relation must define the foreign key.

### Basic One-to-One

```prisma
model User {
  id      String   @id @default(cuid())
  email   String   @unique
  profile Profile?
}

model Profile {
  id     String  @id @default(cuid())
  bio    String?
  avatar String?
  user   User    @relation(fields: [userId], references: [id])
  userId String  @unique // @unique makes it one-to-one
}
```

**Key Points:**
- The `@unique` constraint on `userId` ensures that each `Profile` can only belong to one `User`
- The side with the scalar field (`userId`) is called the "relation scalar field"
- The side with the relation field (`profile`) is the "back-relation"

### Required One-to-One

```prisma
model User {
  id      String  @id @default(cuid())
  email   String  @unique
  profile Profile // Required - every user must have a profile
}

model Profile {
  id     String @id @default(cuid())
  bio    String
  user   User   @relation(fields: [userId], references: [id])
  userId String @unique
}
```

### One-to-One with Composite Keys

```prisma
model User {
  firstName String
  lastName  String
  profile   Profile?

  @@id([firstName, lastName])
}

model Profile {
  id            Int    @id @default(autoincrement())
  bio           String
  userFirstName String
  userLastName  String
  user          User   @relation(fields: [userFirstName, userLastName], references: [firstName, lastName])

  @@unique([userFirstName, userLastName])
}
```

### Querying One-to-One Relations

```typescript
// Create user with profile
const userWithProfile = await prisma.user.create({
  data: {
    email: 'alice@example.com',
    profile: {
      create: {
        bio: 'Software developer',
        avatar: 'https://example.com/avatar.png',
      },
    },
  },
  include: {
    profile: true,
  },
});

// Find user with profile
const user = await prisma.user.findUnique({
  where: { email: 'alice@example.com' },
  include: { profile: true },
});

// Update profile through user
const updatedUser = await prisma.user.update({
  where: { email: 'alice@example.com' },
  data: {
    profile: {
      update: {
        bio: 'Senior software developer',
      },
    },
  },
});

// Create or update profile (upsert)
const userWithUpsertedProfile = await prisma.user.update({
  where: { email: 'alice@example.com' },
  data: {
    profile: {
      upsert: {
        create: { bio: 'New bio' },
        update: { bio: 'Updated bio' },
      },
    },
  },
});
```

---

## 2. One-to-Many Relations

A one-to-many (1-n) relation means that one record on one side can be connected to multiple records on the other side. This is the most common type of relation.

### Basic One-to-Many

```prisma
model User {
  id    String @id @default(cuid())
  email String @unique
  name  String
  posts Post[] // One user has many posts
}

model Post {
  id        String   @id @default(cuid())
  title     String
  content   String?
  published Boolean  @default(false)
  author    User     @relation(fields: [authorId], references: [id])
  authorId  String   // Foreign key - no @unique constraint
}
```

**Key Points:**
- The "many" side (`posts`) is represented by an array type `Post[]`
- The "one" side has the foreign key field (`authorId`)
- No `@unique` constraint on the foreign key (unlike one-to-one)

### One-to-Many with Required vs Optional

```prisma
// Required: Every post must have an author
model Post {
  id       String @id @default(cuid())
  title    String
  author   User   @relation(fields: [authorId], references: [id])
  authorId String // Required foreign key
}

// Optional: Posts can exist without an author
model Post {
  id       String  @id @default(cuid())
  title    String
  author   User?   @relation(fields: [authorId], references: [id])
  authorId String? // Optional foreign key
}
```

### Nested One-to-Many

```prisma
model User {
  id       String    @id @default(cuid())
  email    String    @unique
  posts    Post[]
  comments Comment[]
}

model Post {
  id       String    @id @default(cuid())
  title    String
  author   User      @relation(fields: [authorId], references: [id])
  authorId String
  comments Comment[] // Post has many comments
}

model Comment {
  id       String @id @default(cuid())
  content  String
  author   User   @relation(fields: [authorId], references: [id])
  authorId String
  post     Post   @relation(fields: [postId], references: [id])
  postId   String
}
```

### Querying One-to-Many Relations

```typescript
// Create user with multiple posts
const userWithPosts = await prisma.user.create({
  data: {
    email: 'bob@example.com',
    name: 'Bob',
    posts: {
      create: [
        { title: 'First Post', content: 'Hello World' },
        { title: 'Second Post', content: 'More content' },
        { title: 'Third Post', published: true },
      ],
    },
  },
  include: {
    posts: true,
  },
});

// Find user with filtered posts
const userWithPublishedPosts = await prisma.user.findUnique({
  where: { email: 'bob@example.com' },
  include: {
    posts: {
      where: { published: true },
      orderBy: { title: 'asc' },
    },
  },
});

// Find user with limited posts
const userWithRecentPosts = await prisma.user.findUnique({
  where: { email: 'bob@example.com' },
  include: {
    posts: {
      take: 5,
      orderBy: { createdAt: 'desc' },
    },
  },
});

// Get all posts for a user
const posts = await prisma.post.findMany({
  where: {
    author: {
      email: 'bob@example.com',
    },
  },
});
```

---

## 3. Many-to-Many Relations

A many-to-many (m-n) relation means that records on both sides can be connected to multiple records on the other side. Prisma supports two types: implicit and explicit.

### Implicit Many-to-Many

Prisma automatically creates and manages the join table. Use this when you don't need extra fields on the relation.

```prisma
model Post {
  id         String     @id @default(cuid())
  title      String
  content    String?
  categories Category[]
}

model Category {
  id    String @id @default(cuid())
  name  String @unique
  posts Post[]
}
```

**How Prisma handles implicit m-n:**
- Creates a join table named `_CategoryToPost` (alphabetical order)
- Join table has two columns: `A` and `B` referencing the primary keys
- Join table is hidden from your Prisma Client API

### Implicit Many-to-Many with @relation Name

```prisma
model Post {
  id         String     @id @default(cuid())
  title      String
  categories Category[] @relation("PostCategories")
}

model Category {
  id    String @id @default(cuid())
  name  String @unique
  posts Post[] @relation("PostCategories")
}
```

### Explicit Many-to-Many

Use explicit join tables when you need:
- Additional fields on the relation (e.g., timestamps, roles)
- More control over the join table structure
- Custom naming for the join table

```prisma
model Post {
  id       String       @id @default(cuid())
  title    String
  content  String?
  tags     PostTag[]
}

model Tag {
  id    String    @id @default(cuid())
  name  String    @unique
  posts PostTag[]
}

model PostTag {
  id         String   @id @default(cuid())
  post       Post     @relation(fields: [postId], references: [id], onDelete: Cascade)
  postId     String
  tag        Tag      @relation(fields: [tagId], references: [id], onDelete: Cascade)
  tagId      String
  assignedAt DateTime @default(now())
  assignedBy String?
  order      Int      @default(0)

  @@unique([postId, tagId])
  @@index([tagId])
}
```

### Alternative: Composite Primary Key for Join Table

```prisma
model PostTag {
  post       Post     @relation(fields: [postId], references: [id], onDelete: Cascade)
  postId     String
  tag        Tag      @relation(fields: [tagId], references: [id], onDelete: Cascade)
  tagId      String
  assignedAt DateTime @default(now())

  @@id([postId, tagId]) // Composite primary key instead of separate id
}
```

### Querying Implicit Many-to-Many

```typescript
// Create post with categories
const post = await prisma.post.create({
  data: {
    title: 'My Post',
    categories: {
      create: [
        { name: 'Technology' },
        { name: 'Programming' },
      ],
    },
  },
  include: { categories: true },
});

// Connect existing categories to post
const updatedPost = await prisma.post.update({
  where: { id: post.id },
  data: {
    categories: {
      connect: [
        { name: 'JavaScript' },
        { name: 'Web Development' },
      ],
    },
  },
});

// Set specific categories (replaces all existing)
const postWithNewCategories = await prisma.post.update({
  where: { id: post.id },
  data: {
    categories: {
      set: [
        { name: 'React' },
        { name: 'Node.js' },
      ],
    },
  },
});

// Disconnect categories
const postWithoutCategory = await prisma.post.update({
  where: { id: post.id },
  data: {
    categories: {
      disconnect: [{ name: 'React' }],
    },
  },
});
```

### Querying Explicit Many-to-Many

```typescript
// Create post with tags through join table
const post = await prisma.post.create({
  data: {
    title: 'Tagged Post',
    tags: {
      create: [
        {
          tag: { create: { name: 'prisma' } },
          assignedBy: 'admin',
        },
        {
          tag: { connect: { name: 'database' } },
          assignedBy: 'admin',
          order: 1,
        },
      ],
    },
  },
  include: {
    tags: {
      include: { tag: true },
    },
  },
});

// Query through join table
const postsWithTag = await prisma.postTag.findMany({
  where: {
    tag: { name: 'prisma' },
  },
  include: {
    post: true,
    tag: true,
  },
});

// Get all tags for a post with join table data
const postWithTags = await prisma.post.findUnique({
  where: { id: 'post-id' },
  include: {
    tags: {
      include: { tag: true },
      orderBy: { order: 'asc' },
    },
  },
});

// Add tag to post (explicit)
const newPostTag = await prisma.postTag.create({
  data: {
    post: { connect: { id: 'post-id' } },
    tag: { connect: { id: 'tag-id' } },
    assignedBy: 'user-123',
  },
});

// Remove tag from post (explicit)
await prisma.postTag.delete({
  where: {
    postId_tagId: {
      postId: 'post-id',
      tagId: 'tag-id',
    },
  },
});
```

---

## 4. Self-Relations

Self-relations are relations where a model references itself. Common use cases include hierarchies, trees, followers/following, and organizational structures.

### One-to-One Self-Relation

```prisma
model User {
  id          String  @id @default(cuid())
  name        String
  successor   User?   @relation("Succession", fields: [successorId], references: [id])
  successorId String? @unique
  predecessor User?   @relation("Succession")
}
```

### One-to-Many Self-Relation (Tree/Hierarchy)

```prisma
model Category {
  id       String     @id @default(cuid())
  name     String
  parent   Category?  @relation("CategoryHierarchy", fields: [parentId], references: [id])
  parentId String?
  children Category[] @relation("CategoryHierarchy")
}
```

### Many-to-Many Self-Relation (Followers)

```prisma
model User {
  id         String @id @default(cuid())
  name       String
  email      String @unique
  followers  User[] @relation("UserFollows")
  following  User[] @relation("UserFollows")
}
```

### Explicit Many-to-Many Self-Relation

```prisma
model User {
  id         String   @id @default(cuid())
  name       String
  followers  Follow[] @relation("Followers")
  following  Follow[] @relation("Following")
}

model Follow {
  id          String   @id @default(cuid())
  follower    User     @relation("Following", fields: [followerId], references: [id])
  followerId  String
  following   User     @relation("Followers", fields: [followingId], references: [id])
  followingId String
  createdAt   DateTime @default(now())

  @@unique([followerId, followingId])
  @@index([followingId])
}
```

### Querying Self-Relations

```typescript
// Create category hierarchy
const electronics = await prisma.category.create({
  data: {
    name: 'Electronics',
    children: {
      create: [
        {
          name: 'Computers',
          children: {
            create: [
              { name: 'Laptops' },
              { name: 'Desktops' },
            ],
          },
        },
        {
          name: 'Phones',
          children: {
            create: [
              { name: 'Smartphones' },
              { name: 'Feature Phones' },
            ],
          },
        },
      ],
    },
  },
  include: {
    children: {
      include: {
        children: true,
      },
    },
  },
});

// Get category with all descendants (limited depth)
const categoryWithChildren = await prisma.category.findUnique({
  where: { name: 'Electronics' },
  include: {
    children: {
      include: {
        children: {
          include: {
            children: true,
          },
        },
      },
    },
  },
});

// Get category with parent
const categoryWithParent = await prisma.category.findUnique({
  where: { name: 'Laptops' },
  include: {
    parent: {
      include: {
        parent: true,
      },
    },
  },
});

// Follow a user (implicit m-n)
const user = await prisma.user.update({
  where: { email: 'alice@example.com' },
  data: {
    following: {
      connect: { email: 'bob@example.com' },
    },
  },
});

// Get user with followers and following
const userWithFollows = await prisma.user.findUnique({
  where: { email: 'alice@example.com' },
  include: {
    followers: true,
    following: true,
  },
});

// Check if user follows another user
const isFollowing = await prisma.user.findFirst({
  where: {
    email: 'alice@example.com',
    following: {
      some: { email: 'bob@example.com' },
    },
  },
});
```

---

## 5. @relation Attribute

The `@relation` attribute defines how relations work between models. It's used to configure relation names, foreign keys, and referential actions.

### Syntax

```prisma
@relation(name?, fields?, references?, onDelete?, onUpdate?, map?)
```

### Arguments

| Argument     | Required | Description                                                      |
|--------------|----------|------------------------------------------------------------------|
| `name`       | No       | Defines the name of the relation                                 |
| `fields`     | Yes*     | List of scalar fields in the current model that store the FK     |
| `references` | Yes*     | List of fields in the related model that the FK references       |
| `onDelete`   | No       | Referential action when related record is deleted                |
| `onUpdate`   | No       | Referential action when related record's identifier is updated   |
| `map`        | No       | Custom name for the foreign key constraint in the database       |

*Required on the side that stores the foreign key

### Examples

```prisma
// Basic relation with foreign key
model Post {
  id       String @id @default(cuid())
  author   User   @relation(fields: [authorId], references: [id])
  authorId String
}

// Named relation
model Post {
  author   User @relation("PostAuthor", fields: [authorId], references: [id])
  authorId String
}

// With referential actions
model Post {
  author   User   @relation(fields: [authorId], references: [id], onDelete: Cascade, onUpdate: Cascade)
  authorId String
}

// With custom constraint name
model Post {
  author   User   @relation(fields: [authorId], references: [id], map: "fk_post_author")
  authorId String
}

// Composite foreign key
model OrderItem {
  order       Order  @relation(fields: [orderId, orderDate], references: [id, date])
  orderId     String
  orderDate   DateTime
}
```

### When @relation is Required

The `@relation` attribute is required when:
1. You need to define which fields store the foreign key
2. You have multiple relations between the same models
3. You have a self-relation
4. You want to configure referential actions
5. You want to customize the foreign key constraint name

### Implicit Relations (no @relation needed on one side)

```prisma
model User {
  id    String @id
  posts Post[] // No @relation needed here (back-relation)
}

model Post {
  id       String @id
  author   User   @relation(fields: [authorId], references: [id]) // @relation required
  authorId String
}
```

---

## 6. Referential Actions

Referential actions determine what happens to related records when the parent record is deleted or updated. They're configured with `onDelete` and `onUpdate` in the `@relation` attribute.

### Available Actions

| Action       | Description                                                    |
|--------------|----------------------------------------------------------------|
| `Cascade`    | Delete/update related records when parent is deleted/updated   |
| `Restrict`   | Prevent deletion/update if related records exist               |
| `NoAction`   | Similar to Restrict, but checked at end of transaction         |
| `SetNull`    | Set the foreign key to null (field must be optional)           |
| `SetDefault` | Set the foreign key to its default value                       |

### onDelete Examples

```prisma
// Cascade: Delete all posts when user is deleted
model Post {
  id       String @id @default(cuid())
  title    String
  author   User   @relation(fields: [authorId], references: [id], onDelete: Cascade)
  authorId String
}

// Restrict: Prevent user deletion if they have posts
model Post {
  id       String @id @default(cuid())
  title    String
  author   User   @relation(fields: [authorId], references: [id], onDelete: Restrict)
  authorId String
}

// SetNull: Set authorId to null when user is deleted
model Post {
  id       String  @id @default(cuid())
  title    String
  author   User?   @relation(fields: [authorId], references: [id], onDelete: SetNull)
  authorId String?
}

// SetDefault: Set to default value when user is deleted
model Post {
  id       String @id @default(cuid())
  title    String
  author   User   @relation(fields: [authorId], references: [id], onDelete: SetDefault)
  authorId String @default("default-user-id")
}

// NoAction: Database handles the constraint (transaction-level)
model Post {
  id       String @id @default(cuid())
  title    String
  author   User   @relation(fields: [authorId], references: [id], onDelete: NoAction)
  authorId String
}
```

### onUpdate Examples

```prisma
// Cascade: Update foreign key when user id changes
model Post {
  id       String @id @default(cuid())
  author   User   @relation(fields: [authorId], references: [id], onUpdate: Cascade)
  authorId String
}

// Restrict: Prevent user id update if posts exist
model Post {
  id       String @id @default(cuid())
  author   User   @relation(fields: [authorId], references: [id], onUpdate: Restrict)
  authorId String
}
```

### Combining onDelete and onUpdate

```prisma
model Comment {
  id       String @id @default(cuid())
  content  String
  post     Post   @relation(fields: [postId], references: [id], onDelete: Cascade, onUpdate: Cascade)
  postId   String
  author   User?  @relation(fields: [authorId], references: [id], onDelete: SetNull, onUpdate: Cascade)
  authorId String?
}
```

### Database Provider Defaults

| Database   | Default onDelete | Default onUpdate |
|------------|------------------|------------------|
| PostgreSQL | NoAction         | Cascade          |
| MySQL      | NoAction         | Cascade          |
| SQLite     | NoAction         | Cascade          |
| SQL Server | NoAction         | Cascade          |
| MongoDB    | NoAction         | Cascade          |

### Handling Referential Actions in Code

```typescript
// With Cascade, deleting user automatically deletes posts
await prisma.user.delete({
  where: { id: 'user-id' },
});

// With Restrict, this will fail if user has posts
try {
  await prisma.user.delete({
    where: { id: 'user-id' },
  });
} catch (error) {
  if (error.code === 'P2003') {
    console.log('Cannot delete user with existing posts');
  }
}

// Manual cascade (when not using Cascade action)
const deleteUser = async (userId: string) => {
  // Delete in correct order
  await prisma.comment.deleteMany({ where: { authorId: userId } });
  await prisma.post.deleteMany({ where: { authorId: userId } });
  await prisma.user.delete({ where: { id: userId } });
};

// Using transactions for manual cascade
const deleteUserTransaction = async (userId: string) => {
  await prisma.$transaction([
    prisma.comment.deleteMany({ where: { authorId: userId } }),
    prisma.post.deleteMany({ where: { authorId: userId } }),
    prisma.user.delete({ where: { id: userId } }),
  ]);
};
```

---

## 7. Optional vs Required Relations

Relations can be optional or required, affecting how you create and query data.

### Required Relations

A required relation means the foreign key field cannot be null. The related record must exist.

```prisma
model Post {
  id       String @id @default(cuid())
  title    String
  author   User   @relation(fields: [authorId], references: [id])
  authorId String // Required - cannot be null
}
```

```typescript
// Must provide author when creating post
const post = await prisma.post.create({
  data: {
    title: 'My Post',
    author: {
      connect: { id: 'user-id' },
    },
    // OR
    authorId: 'user-id',
  },
});

// This will fail - authorId is required
const invalidPost = await prisma.post.create({
  data: {
    title: 'My Post',
    // Error: authorId is required
  },
});
```

### Optional Relations

An optional relation means the foreign key field can be null. The relation can exist without a related record.

```prisma
model Post {
  id       String  @id @default(cuid())
  title    String
  author   User?   @relation(fields: [authorId], references: [id])
  authorId String? // Optional - can be null
}
```

```typescript
// Can create post without author
const post = await prisma.post.create({
  data: {
    title: 'Anonymous Post',
    // authorId is optional
  },
});

// Can set author to null
const disconnectedPost = await prisma.post.update({
  where: { id: post.id },
  data: {
    author: {
      disconnect: true,
    },
  },
});

// Or directly set to null
const nullAuthorPost = await prisma.post.update({
  where: { id: post.id },
  data: {
    authorId: null,
  },
});
```

### Optional Back-Relations

The back-relation side (without foreign key) can also be optional or required:

```prisma
// Required: Every user must have a profile
model User {
  id      String  @id
  profile Profile // Required
}

// Optional: User might not have a profile
model User {
  id      String   @id
  profile Profile? // Optional
}
```

### Working with Optional Relations

```typescript
// Filter for records with/without relations
const postsWithAuthor = await prisma.post.findMany({
  where: {
    authorId: { not: null },
  },
});

const postsWithoutAuthor = await prisma.post.findMany({
  where: {
    authorId: null,
  },
});

// Using relation filters
const postsWithAuthorFilter = await prisma.post.findMany({
  where: {
    author: {
      isNot: null,
    },
  },
});

const postsWithoutAuthorFilter = await prisma.post.findMany({
  where: {
    author: null,
  },
});

// Handling nullable relations in TypeScript
const post = await prisma.post.findUnique({
  where: { id: 'post-id' },
  include: { author: true },
});

if (post?.author) {
  console.log(post.author.name);
} else {
  console.log('Anonymous post');
}
```

---

## 8. Named Relations

Named relations allow you to explicitly name the relationship between models. This is required when you have multiple relations between the same models or self-relations.

### Why Use Named Relations

1. Multiple relations between same models
2. Self-relations
3. Better schema documentation
4. Controlling implicit many-to-many table names

### Basic Named Relation

```prisma
model User {
  id    String @id
  posts Post[] @relation("UserPosts")
}

model Post {
  id       String @id
  author   User   @relation("UserPosts", fields: [authorId], references: [id])
  authorId String
}
```

### Named Relations for Disambiguation

```prisma
model User {
  id            String @id
  writtenPosts  Post[] @relation("Author")
  favoritePosts Post[] @relation("Favorites")
}

model Post {
  id          String @id
  author      User   @relation("Author", fields: [authorId], references: [id])
  authorId    String
  favoritedBy User[] @relation("Favorites")
}
```

### Named Self-Relations

```prisma
model Person {
  id         String   @id
  name       String
  spouse     Person?  @relation("Marriage", fields: [spouseId], references: [id])
  spouseId   String?  @unique
  spouseOf   Person?  @relation("Marriage")
}

model Employee {
  id           String     @id
  name         String
  manager      Employee?  @relation("Management", fields: [managerId], references: [id])
  managerId    String?
  directReports Employee[] @relation("Management")
}
```

### Controlling Implicit Join Table Names

```prisma
// Without name: creates _CategoryToPost table
model Post {
  id         String     @id
  categories Category[]
}

model Category {
  id    String @id
  posts Post[]
}

// With name: creates _ProductCategories table
model Product {
  id         String     @id
  categories Category[] @relation("ProductCategories")
}

model Category {
  id       String    @id
  products Product[] @relation("ProductCategories")
}
```

---

## 9. Multiple Relations Between Same Models

When two models have multiple different relationships, you must use named relations to distinguish them.

### Two One-to-Many Relations

```prisma
model User {
  id           String @id @default(cuid())
  name         String
  email        String @unique
  writtenPosts Post[] @relation("WrittenPosts")
  editedPosts  Post[] @relation("EditedPosts")
}

model Post {
  id        String  @id @default(cuid())
  title     String
  content   String?
  author    User    @relation("WrittenPosts", fields: [authorId], references: [id])
  authorId  String
  editor    User?   @relation("EditedPosts", fields: [editorId], references: [id])
  editorId  String?
}
```

### Mixed Relation Types

```prisma
model User {
  id             String    @id @default(cuid())
  email          String    @unique
  createdTasks   Task[]    @relation("TaskCreator")
  assignedTasks  Task[]    @relation("TaskAssignee")
  reviewedTasks  Task[]    @relation("TaskReviewer")
}

model Task {
  id          String   @id @default(cuid())
  title       String
  creator     User     @relation("TaskCreator", fields: [creatorId], references: [id])
  creatorId   String
  assignee    User?    @relation("TaskAssignee", fields: [assigneeId], references: [id])
  assigneeId  String?
  reviewers   User[]   @relation("TaskReviewer")
}
```

### One-to-One and One-to-Many Between Same Models

```prisma
model User {
  id            String   @id @default(cuid())
  email         String   @unique
  mainAddress   Address? @relation("MainAddress")
  allAddresses  Address[] @relation("UserAddresses")
}

model Address {
  id           String  @id @default(cuid())
  street       String
  city         String
  mainFor      User?   @relation("MainAddress", fields: [mainForId], references: [id])
  mainForId    String? @unique
  belongsTo    User    @relation("UserAddresses", fields: [belongsToId], references: [id])
  belongsToId  String
}
```

### Querying Multiple Relations

```typescript
// Create post with author and editor
const post = await prisma.post.create({
  data: {
    title: 'Edited Article',
    author: { connect: { email: 'writer@example.com' } },
    editor: { connect: { email: 'editor@example.com' } },
  },
});

// Get user with both written and edited posts
const userWithPosts = await prisma.user.findUnique({
  where: { email: 'writer@example.com' },
  include: {
    writtenPosts: true,
    editedPosts: true,
  },
});

// Find posts where user is both author and editor
const selfEditedPosts = await prisma.post.findMany({
  where: {
    AND: [
      { authorId: 'user-id' },
      { editorId: 'user-id' },
    ],
  },
});

// Get all posts where user is involved (as author OR editor)
const allInvolvedPosts = await prisma.post.findMany({
  where: {
    OR: [
      { authorId: 'user-id' },
      { editorId: 'user-id' },
    ],
  },
});
```

---

## 10. Querying Relations

Prisma provides `include` and `select` for fetching related data.

### Using include

`include` loads the entire related record(s).

```typescript
// Include single relation
const user = await prisma.user.findUnique({
  where: { id: 'user-id' },
  include: { profile: true },
});

// Include multiple relations
const user = await prisma.user.findUnique({
  where: { id: 'user-id' },
  include: {
    profile: true,
    posts: true,
  },
});

// Nested include
const user = await prisma.user.findUnique({
  where: { id: 'user-id' },
  include: {
    posts: {
      include: {
        comments: {
          include: { author: true },
        },
      },
    },
  },
});

// Include with filtering
const user = await prisma.user.findUnique({
  where: { id: 'user-id' },
  include: {
    posts: {
      where: { published: true },
      orderBy: { createdAt: 'desc' },
      take: 10,
    },
  },
});
```

### Using select

`select` allows you to pick specific fields, including from relations.

```typescript
// Select specific fields
const user = await prisma.user.findUnique({
  where: { id: 'user-id' },
  select: {
    name: true,
    email: true,
  },
});

// Select with relations
const user = await prisma.user.findUnique({
  where: { id: 'user-id' },
  select: {
    name: true,
    posts: {
      select: {
        title: true,
        published: true,
      },
    },
  },
});

// Nested select
const user = await prisma.user.findUnique({
  where: { id: 'user-id' },
  select: {
    name: true,
    posts: {
      select: {
        title: true,
        comments: {
          select: {
            content: true,
            author: {
              select: { name: true },
            },
          },
        },
      },
    },
  },
});
```

### Combining include and select (within relations)

You cannot use both `include` and `select` at the top level, but you can use `select` within `include`:

```typescript
// This works
const user = await prisma.user.findUnique({
  where: { id: 'user-id' },
  include: {
    posts: {
      select: {
        title: true,
        content: true,
      },
    },
  },
});

// This does NOT work
const user = await prisma.user.findUnique({
  where: { id: 'user-id' },
  select: { name: true },
  include: { posts: true }, // Error: Cannot use both
});

// Instead, use select with nested select
const user = await prisma.user.findUnique({
  where: { id: 'user-id' },
  select: {
    name: true,
    posts: true, // This includes all post fields
  },
});
```

### Fluent API for Relations

```typescript
// Get posts for a specific user using fluent API
const userPosts = await prisma.user.findUnique({
  where: { id: 'user-id' },
}).posts();

// Chain with filtering
const publishedPosts = await prisma.user.findUnique({
  where: { id: 'user-id' },
}).posts({
  where: { published: true },
});

// Get profile through user
const userProfile = await prisma.user.findUnique({
  where: { id: 'user-id' },
}).profile();
```

---

## 11. Nested Writes

Nested writes allow you to create, update, or modify related records in a single operation.

### create

Create new related records.

```typescript
// Create user with new posts
const user = await prisma.user.create({
  data: {
    email: 'new@example.com',
    name: 'New User',
    posts: {
      create: [
        { title: 'First Post' },
        { title: 'Second Post', published: true },
      ],
    },
  },
  include: { posts: true },
});

// Create nested records (3 levels)
const user = await prisma.user.create({
  data: {
    email: 'nested@example.com',
    posts: {
      create: {
        title: 'Post with Comments',
        comments: {
          create: [
            { content: 'Great post!' },
            { content: 'Thanks for sharing' },
          ],
        },
      },
    },
  },
});
```

### connect

Connect to existing records.

```typescript
// Create post connected to existing user
const post = await prisma.post.create({
  data: {
    title: 'New Post',
    author: {
      connect: { email: 'existing@example.com' },
    },
  },
});

// Connect multiple existing records (many-to-many)
const post = await prisma.post.update({
  where: { id: 'post-id' },
  data: {
    categories: {
      connect: [
        { id: 'category-1' },
        { id: 'category-2' },
        { name: 'Technology' }, // Can use unique fields
      ],
    },
  },
});
```

### connectOrCreate

Connect to an existing record or create it if it doesn't exist.

```typescript
// Connect to existing category or create new one
const post = await prisma.post.update({
  where: { id: 'post-id' },
  data: {
    categories: {
      connectOrCreate: {
        where: { name: 'Technology' },
        create: { name: 'Technology' },
      },
    },
  },
});

// Multiple connectOrCreate
const post = await prisma.post.update({
  where: { id: 'post-id' },
  data: {
    categories: {
      connectOrCreate: [
        {
          where: { name: 'Tech' },
          create: { name: 'Tech' },
        },
        {
          where: { name: 'Programming' },
          create: { name: 'Programming' },
        },
      ],
    },
  },
});
```

### disconnect

Remove the connection between records (doesn't delete the record).

```typescript
// Disconnect single relation (one-to-one/many-to-one)
const post = await prisma.post.update({
  where: { id: 'post-id' },
  data: {
    author: {
      disconnect: true, // Sets authorId to null (field must be optional)
    },
  },
});

// Disconnect from many-to-many
const post = await prisma.post.update({
  where: { id: 'post-id' },
  data: {
    categories: {
      disconnect: [
        { id: 'category-1' },
        { name: 'Technology' },
      ],
    },
  },
});
```

### set

Replace all connected records (many relations only).

```typescript
// Replace all categories
const post = await prisma.post.update({
  where: { id: 'post-id' },
  data: {
    categories: {
      set: [
        { id: 'category-1' },
        { id: 'category-2' },
      ],
    },
  },
});

// Clear all connections
const post = await prisma.post.update({
  where: { id: 'post-id' },
  data: {
    categories: {
      set: [], // Removes all category connections
    },
  },
});
```

### delete

Delete related records.

```typescript
// Delete single related record (one-to-one)
const user = await prisma.user.update({
  where: { id: 'user-id' },
  data: {
    profile: {
      delete: true,
    },
  },
});

// Delete specific related records (one-to-many)
const user = await prisma.user.update({
  where: { id: 'user-id' },
  data: {
    posts: {
      delete: [
        { id: 'post-1' },
        { id: 'post-2' },
      ],
    },
  },
});

// Delete with condition
const user = await prisma.user.update({
  where: { id: 'user-id' },
  data: {
    posts: {
      deleteMany: {
        published: false, // Delete all unpublished posts
      },
    },
  },
});
```

### update and updateMany

Update related records.

```typescript
// Update single related record
const user = await prisma.user.update({
  where: { id: 'user-id' },
  data: {
    profile: {
      update: {
        bio: 'Updated bio',
      },
    },
  },
});

// Update specific related records
const user = await prisma.user.update({
  where: { id: 'user-id' },
  data: {
    posts: {
      update: {
        where: { id: 'post-id' },
        data: { title: 'Updated Title' },
      },
    },
  },
});

// Update multiple related records
const user = await prisma.user.update({
  where: { id: 'user-id' },
  data: {
    posts: {
      updateMany: {
        where: { published: false },
        data: { published: true },
      },
    },
  },
});
```

### upsert

Update if exists, create if not.

```typescript
// Upsert related record
const user = await prisma.user.update({
  where: { id: 'user-id' },
  data: {
    profile: {
      upsert: {
        create: { bio: 'New bio' },
        update: { bio: 'Updated bio' },
      },
    },
  },
});

// Upsert in one-to-many (with where condition)
const user = await prisma.user.update({
  where: { id: 'user-id' },
  data: {
    posts: {
      upsert: {
        where: { id: 'post-id' },
        create: { title: 'New Post' },
        update: { title: 'Updated Post' },
      },
    },
  },
});
```

---

## 12. Filtering on Relations

Prisma provides powerful relation filters: `some`, `every`, `none`, `is`, and `isNot`.

### some

Returns records where at least one related record matches.

```typescript
// Users with at least one published post
const usersWithPublishedPosts = await prisma.user.findMany({
  where: {
    posts: {
      some: {
        published: true,
      },
    },
  },
});

// Posts with at least one comment from a specific user
const postsWithUserComments = await prisma.post.findMany({
  where: {
    comments: {
      some: {
        author: {
          email: 'user@example.com',
        },
      },
    },
  },
});

// Nested some
const users = await prisma.user.findMany({
  where: {
    posts: {
      some: {
        comments: {
          some: {
            content: { contains: 'great' },
          },
        },
      },
    },
  },
});
```

### every

Returns records where all related records match (or there are no related records).

```typescript
// Users where all posts are published
const usersWithAllPublished = await prisma.user.findMany({
  where: {
    posts: {
      every: {
        published: true,
      },
    },
  },
});

// Categories where all products are in stock
const categoriesInStock = await prisma.category.findMany({
  where: {
    products: {
      every: {
        stock: { gt: 0 },
      },
    },
  },
});
```

### none

Returns records where no related records match.

```typescript
// Users with no published posts
const usersWithNoPublished = await prisma.user.findMany({
  where: {
    posts: {
      none: {
        published: true,
      },
    },
  },
});

// Users with no posts at all
const usersWithNoPosts = await prisma.user.findMany({
  where: {
    posts: {
      none: {},
    },
  },
});

// Products not in any category
const uncategorizedProducts = await prisma.product.findMany({
  where: {
    categories: {
      none: {},
    },
  },
});
```

### is

Filter on to-one relations by matching the related record.

```typescript
// Posts by a specific author
const posts = await prisma.post.findMany({
  where: {
    author: {
      is: {
        email: 'author@example.com',
      },
    },
  },
});

// Comments on published posts
const comments = await prisma.comment.findMany({
  where: {
    post: {
      is: {
        published: true,
      },
    },
  },
});

// Nested is
const comments = await prisma.comment.findMany({
  where: {
    post: {
      is: {
        author: {
          is: {
            email: 'author@example.com',
          },
        },
      },
    },
  },
});
```

### isNot

Filter on to-one relations by NOT matching the related record.

```typescript
// Posts NOT by a specific author
const posts = await prisma.post.findMany({
  where: {
    author: {
      isNot: {
        email: 'excluded@example.com',
      },
    },
  },
});

// Comments not on published posts
const draftComments = await prisma.comment.findMany({
  where: {
    post: {
      isNot: {
        published: true,
      },
    },
  },
});
```

### Filtering for null relations

```typescript
// Posts with no author (author is null)
const anonymousPosts = await prisma.post.findMany({
  where: {
    author: null,
  },
});

// Posts with an author (author is not null)
const authoredPosts = await prisma.post.findMany({
  where: {
    author: {
      isNot: null,
    },
  },
});

// Alternative using scalar field
const postsWithAuthor = await prisma.post.findMany({
  where: {
    authorId: { not: null },
  },
});
```

### Combining Filters

```typescript
// Complex filter combining relation filters
const users = await prisma.user.findMany({
  where: {
    AND: [
      {
        posts: {
          some: { published: true },
        },
      },
      {
        posts: {
          none: { title: { contains: 'draft' } },
        },
      },
    ],
  },
});

// Users with published posts but no comments
const activeAuthors = await prisma.user.findMany({
  where: {
    posts: {
      some: {
        AND: [
          { published: true },
          {
            comments: {
              none: {},
            },
          },
        ],
      },
    },
  },
});
```

---

## 13. Ordering by Relations

Prisma allows you to order query results based on related data.

### Order by To-One Relation Field

```typescript
// Order posts by author name
const posts = await prisma.post.findMany({
  orderBy: {
    author: {
      name: 'asc',
    },
  },
  include: { author: true },
});

// Order comments by post title, then by author email
const comments = await prisma.comment.findMany({
  orderBy: [
    {
      post: {
        title: 'asc',
      },
    },
    {
      author: {
        email: 'asc',
      },
    },
  ],
});

// Order by nested relation
const comments = await prisma.comment.findMany({
  orderBy: {
    post: {
      author: {
        name: 'asc',
      },
    },
  },
});
```

### Order by Relation Aggregates

```typescript
// Order users by number of posts (descending)
const usersByPostCount = await prisma.user.findMany({
  orderBy: {
    posts: {
      _count: 'desc',
    },
  },
  include: {
    _count: {
      select: { posts: true },
    },
  },
});

// Order categories by number of products
const categories = await prisma.category.findMany({
  orderBy: {
    products: {
      _count: 'desc',
    },
  },
});

// Multiple ordering with aggregates
const users = await prisma.user.findMany({
  orderBy: [
    {
      posts: {
        _count: 'desc',
      },
    },
    {
      name: 'asc',
    },
  ],
});
```

### Order Related Records

```typescript
// Get user with posts ordered by date
const user = await prisma.user.findUnique({
  where: { id: 'user-id' },
  include: {
    posts: {
      orderBy: {
        createdAt: 'desc',
      },
    },
  },
});

// Nested ordering
const user = await prisma.user.findUnique({
  where: { id: 'user-id' },
  include: {
    posts: {
      orderBy: { createdAt: 'desc' },
      include: {
        comments: {
          orderBy: { createdAt: 'asc' },
        },
      },
    },
  },
});

// Order by related field in include
const users = await prisma.user.findMany({
  include: {
    posts: {
      orderBy: {
        comments: {
          _count: 'desc',
        },
      },
    },
  },
});
```

### Null Handling in Ordering

```typescript
// Posts with null authors sorted last
const posts = await prisma.post.findMany({
  orderBy: {
    author: {
      name: { sort: 'asc', nulls: 'last' },
    },
  },
});

// Null values first
const posts = await prisma.post.findMany({
  orderBy: {
    author: {
      name: { sort: 'asc', nulls: 'first' },
    },
  },
});
```

---

## 14. Counting Relations

The `_count` feature allows you to count related records without loading them.

### Basic Count

```typescript
// Get user with post count
const user = await prisma.user.findUnique({
  where: { id: 'user-id' },
  include: {
    _count: {
      select: { posts: true },
    },
  },
});
// Result: { id: '...', name: '...', _count: { posts: 5 } }

// Multiple counts
const user = await prisma.user.findUnique({
  where: { id: 'user-id' },
  include: {
    _count: {
      select: {
        posts: true,
        comments: true,
        followers: true,
      },
    },
  },
});
// Result: { ..., _count: { posts: 5, comments: 12, followers: 100 } }
```

### Filtered Count

```typescript
// Count only published posts
const user = await prisma.user.findUnique({
  where: { id: 'user-id' },
  include: {
    _count: {
      select: {
        posts: {
          where: { published: true },
        },
      },
    },
  },
});

// Multiple filtered counts
const user = await prisma.user.findUnique({
  where: { id: 'user-id' },
  include: {
    _count: {
      select: {
        posts: {
          where: { published: true },
        },
      },
    },
    posts: true, // Also include actual posts if needed
  },
});
```

### Count in findMany

```typescript
// Get all users with their post counts
const usersWithCounts = await prisma.user.findMany({
  include: {
    _count: {
      select: { posts: true },
    },
  },
});

// Filter users by count (using having equivalent)
const activeUsers = await prisma.user.findMany({
  where: {
    posts: {
      some: {},
    },
  },
  include: {
    _count: {
      select: { posts: true },
    },
  },
});
```

### Count with Select

```typescript
// Select only specific fields plus count
const user = await prisma.user.findUnique({
  where: { id: 'user-id' },
  select: {
    name: true,
    email: true,
    _count: {
      select: { posts: true },
    },
  },
});
// Result: { name: '...', email: '...', _count: { posts: 5 } }
```

### Order by Count

```typescript
// Order users by post count
const usersByActivity = await prisma.user.findMany({
  orderBy: {
    posts: {
      _count: 'desc',
    },
  },
  include: {
    _count: {
      select: { posts: true },
    },
  },
});
```

### Count in Nested Includes

```typescript
// Get posts with comment counts
const posts = await prisma.post.findMany({
  include: {
    author: true,
    _count: {
      select: { comments: true },
    },
  },
});

// Nested counts
const users = await prisma.user.findMany({
  include: {
    posts: {
      include: {
        _count: {
          select: { comments: true },
        },
      },
    },
    _count: {
      select: { posts: true },
    },
  },
});
```

---

## 15. Relation Field Names and Foreign Keys

Understanding how to name relation fields and foreign keys properly is essential for clean, maintainable schemas.

### Relation Field Naming Conventions

```prisma
// Standard naming
model Post {
  id       String @id
  author   User   @relation(fields: [authorId], references: [id])
  authorId String
}

// Descriptive naming for clarity
model Task {
  id          String @id
  creator     User   @relation("TaskCreator", fields: [creatorId], references: [id])
  creatorId   String
  assignee    User?  @relation("TaskAssignee", fields: [assigneeId], references: [id])
  assigneeId  String?
}

// Plural for to-many relations
model User {
  id       String    @id
  posts    Post[]    // Plural
  comments Comment[] // Plural
  profile  Profile?  // Singular for to-one
}
```

### Foreign Key Naming

```prisma
// Convention: relationFieldName + Id
model Post {
  author   User   @relation(fields: [authorId], references: [id])
  authorId String // authorId follows author
}

// For composite keys
model OrderItem {
  order         Order    @relation(fields: [orderId, orderYear], references: [id, year])
  orderId       String
  orderYear     Int
}

// Custom foreign key column name with @map
model Post {
  author   User   @relation(fields: [authorId], references: [id])
  authorId String @map("author_user_id") // Database column name
}
```

### Mapping Relation Names to Database

```prisma
// Custom foreign key constraint name
model Post {
  id       String @id
  author   User   @relation(fields: [authorId], references: [id], map: "fk_post_author")
  authorId String
}

// Map model and field names
model Post {
  id       String @id @map("post_id")
  author   User   @relation(fields: [authorId], references: [id])
  authorId String @map("author_id")

  @@map("posts") // Table name
}
```

### Referencing Non-Primary Key Fields

```prisma
// Reference unique field instead of primary key
model User {
  id    String @id
  email String @unique
  posts Post[]
}

model Post {
  id          String @id
  author      User   @relation(fields: [authorEmail], references: [email])
  authorEmail String
}

// Reference composite unique
model User {
  firstName String
  lastName  String
  posts     Post[]

  @@unique([firstName, lastName])
}

model Post {
  id              String @id
  authorFirstName String
  authorLastName  String
  author          User   @relation(fields: [authorFirstName, authorLastName], references: [firstName, lastName])
}
```

### Underlying Database Structure

```prisma
// This schema:
model User {
  id    String @id @default(cuid())
  posts Post[]
}

model Post {
  id       String @id @default(cuid())
  author   User   @relation(fields: [authorId], references: [id])
  authorId String
}

// Creates these tables:
// User: id (primary key)
// Post: id (primary key), authorId (foreign key -> User.id)
```

---

## 16. Best Practices

### Schema Design

```prisma
// 1. Use meaningful relation names
model User {
  id            String @id
  writtenPosts  Post[] @relation("Author")      // Clear purpose
  editedPosts   Post[] @relation("Editor")      // Clear purpose
  // Avoid: posts1, posts2
}

// 2. Choose appropriate referential actions
model Comment {
  id     String @id
  post   Post   @relation(fields: [postId], references: [id], onDelete: Cascade)
  postId String
  // Comments should be deleted when post is deleted
}

model Post {
  id       String  @id
  author   User?   @relation(fields: [authorId], references: [id], onDelete: SetNull)
  authorId String?
  // Keep posts when author is deleted, just remove author reference
}

// 3. Use explicit many-to-many when you need extra fields
model PostTag {
  post       Post     @relation(fields: [postId], references: [id])
  postId     String
  tag        Tag      @relation(fields: [tagId], references: [id])
  tagId      String
  addedAt    DateTime @default(now())  // Extra metadata
  addedBy    String?                    // Extra metadata

  @@id([postId, tagId])
}

// 4. Add indexes for foreign keys (especially in many-to-many)
model PostTag {
  postId String
  tagId  String

  @@id([postId, tagId])
  @@index([tagId])  // Important for queries filtering by tag
}
```

### Query Optimization

```typescript
// 1. Use select instead of include when you need specific fields
// Bad: Loads all user fields
const posts = await prisma.post.findMany({
  include: { author: true },
});

// Good: Only loads what you need
const posts = await prisma.post.findMany({
  select: {
    title: true,
    author: {
      select: { name: true },
    },
  },
});

// 2. Limit nested includes to avoid over-fetching
// Bad: Deep nesting
const user = await prisma.user.findUnique({
  where: { id },
  include: {
    posts: {
      include: {
        comments: {
          include: {
            author: {
              include: {
                profile: true,
              },
            },
          },
        },
      },
    },
  },
});

// Good: Flatten or paginate
const user = await prisma.user.findUnique({
  where: { id },
  include: {
    posts: {
      take: 10,
      include: {
        _count: { select: { comments: true } },
      },
    },
  },
});

// 3. Use _count instead of loading relations just to count
// Bad
const user = await prisma.user.findUnique({
  where: { id },
  include: { posts: true },
});
const postCount = user.posts.length;

// Good
const user = await prisma.user.findUnique({
  where: { id },
  include: {
    _count: { select: { posts: true } },
  },
});
const postCount = user._count.posts;

// 4. Batch related queries with transactions
const [user, posts, comments] = await prisma.$transaction([
  prisma.user.findUnique({ where: { id } }),
  prisma.post.findMany({ where: { authorId: id } }),
  prisma.comment.findMany({ where: { authorId: id } }),
]);
```

### Error Handling

```typescript
// 1. Handle relation constraint errors
try {
  await prisma.post.create({
    data: {
      title: 'Post',
      authorId: 'non-existent-user',
    },
  });
} catch (error) {
  if (error.code === 'P2003') {
    // Foreign key constraint failed
    console.log('User does not exist');
  }
  if (error.code === 'P2025') {
    // Record not found for connect/disconnect
    console.log('Related record not found');
  }
}

// 2. Validate relations before operations
const userExists = await prisma.user.findUnique({
  where: { id: authorId },
  select: { id: true },
});

if (!userExists) {
  throw new Error('Author not found');
}

await prisma.post.create({
  data: { title: 'Post', authorId },
});

// 3. Use connectOrCreate for safety
await prisma.post.update({
  where: { id: postId },
  data: {
    tags: {
      connectOrCreate: {
        where: { name: 'prisma' },
        create: { name: 'prisma' },
      },
    },
  },
});
```

### Type Safety

```typescript
// 1. Use Prisma-generated types
import { Prisma, User, Post } from '@prisma/client';

// Type for user with posts
type UserWithPosts = Prisma.UserGetPayload<{
  include: { posts: true };
}>;

// Type for selected fields
type UserBasic = Prisma.UserGetPayload<{
  select: { id: true; name: true; email: true };
}>;

// 2. Create reusable include/select objects
const postWithAuthor = Prisma.validator<Prisma.PostInclude>()({
  author: {
    select: { id: true, name: true, email: true },
  },
});

const posts = await prisma.post.findMany({
  include: postWithAuthor,
});

// 3. Type function parameters
async function getUserPosts(
  userId: string,
  options?: Prisma.PostFindManyArgs
): Promise<Post[]> {
  return prisma.post.findMany({
    where: { authorId: userId },
    ...options,
  });
}
```

### Common Patterns

```typescript
// 1. Soft delete with relations
model Post {
  id        String    @id
  deletedAt DateTime?
  author    User      @relation(fields: [authorId], references: [id])
  authorId  String
}

// Query non-deleted posts
const activePosts = await prisma.post.findMany({
  where: { deletedAt: null },
});

// 2. Pagination with relations
const page = 1;
const pageSize = 10;

const posts = await prisma.post.findMany({
  skip: (page - 1) * pageSize,
  take: pageSize,
  include: {
    author: { select: { name: true } },
    _count: { select: { comments: true } },
  },
  orderBy: { createdAt: 'desc' },
});

// 3. Full-text search with relations
const searchResults = await prisma.post.findMany({
  where: {
    OR: [
      { title: { contains: searchTerm, mode: 'insensitive' } },
      { content: { contains: searchTerm, mode: 'insensitive' } },
      {
        author: {
          name: { contains: searchTerm, mode: 'insensitive' },
        },
      },
    ],
  },
  include: { author: true },
});

// 4. Aggregate with relations
const categoryStats = await prisma.category.findMany({
  select: {
    name: true,
    _count: { select: { products: true } },
    products: {
      select: {
        price: true,
      },
    },
  },
});

// Calculate averages in JS
const statsWithAverage = categoryStats.map((cat) => ({
  name: cat.name,
  productCount: cat._count.products,
  avgPrice:
    cat.products.reduce((sum, p) => sum + p.price, 0) / cat.products.length || 0,
}));
```

### Migration Considerations

```prisma
// 1. Adding required relation to existing table
// Step 1: Add as optional first
model Post {
  categoryId String?
  category   Category? @relation(fields: [categoryId], references: [id])
}

// Step 2: Migrate and populate data
// Step 3: Make required
model Post {
  categoryId String
  category   Category @relation(fields: [categoryId], references: [id])
}

// 2. Changing relation type requires careful migration
// From one-to-one to one-to-many: Remove @unique from FK

// 3. Adding cascade delete to existing relation
// Review existing data before adding onDelete: Cascade
// Test in development/staging first
```

---

## Summary

Prisma relations provide a powerful way to model and query related data:

- **One-to-One**: Use `@unique` on the foreign key field
- **One-to-Many**: Foreign key without `@unique`, array type on the "many" side
- **Many-to-Many**: Implicit (automatic join table) or explicit (custom join table)
- **Self-Relations**: Same model on both sides, require named relations
- **@relation**: Configure foreign keys, references, and referential actions
- **Referential Actions**: `Cascade`, `Restrict`, `NoAction`, `SetNull`, `SetDefault`
- **Querying**: Use `include` for full records, `select` for specific fields
- **Nested Writes**: `create`, `connect`, `connectOrCreate`, `disconnect`, `set`, `delete`, `update`
- **Filtering**: `some`, `every`, `none` for to-many; `is`, `isNot` for to-one
- **Counting**: Use `_count` for efficient relation counting
- **Ordering**: Order by related fields or relation aggregates

Always consider query performance, type safety, and proper error handling when working with relations.
