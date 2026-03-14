# Prisma Schema

Comprehensive reference for Prisma Schema Language (PSL) - the declarative data modeling language used to define your database schema, models, and relationships.

---

## Table of Contents

1. [Schema Structure](#schema-structure)
2. [Data Types](#data-types)
3. [Field Modifiers](#field-modifiers)
4. [ID Strategies](#id-strategies)
5. [Enums](#enums)
6. [Indexes](#indexes)
7. [Native Database Types](#native-database-types)
8. [Composite Types](#composite-types)
9. [Multi-Schema Support](#multi-schema-support)
10. [Model Attributes](#model-attributes)
11. [Relations](#relations)
12. [Referential Actions](#referential-actions)
13. [Field-level Documentation](#field-level-documentation)
14. [Preview Features](#preview-features)
15. [Environment Variables](#environment-variables)
16. [Prisma CLI Commands](#prisma-cli-commands)
17. [Best Practices](#best-practices)

---

## Schema Structure

The Prisma schema file (`prisma/schema.prisma`) consists of three main blocks: generators, datasources, and models.

### Basic Schema Layout

```prisma
// prisma/schema.prisma

// 1. Generator - Defines what client to generate
generator client {
  provider = "prisma-client-js"
}

// 2. Datasource - Database connection configuration
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// 3. Models - Your data models
model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  posts     Post[]
}
```

### Generator Block

The generator block defines which client library Prisma generates.

```prisma
generator client {
  provider        = "prisma-client-js"
  output          = "./generated/prisma-client"  // Custom output directory
  previewFeatures = ["fullTextSearch", "metrics"]
  binaryTargets   = ["native", "linux-musl-openssl-3.0.x"]
  engineType      = "library"  // or "binary"
}
```

#### Generator Options

| Option | Description | Default |
|--------|-------------|---------|
| `provider` | Generator to use | `"prisma-client-js"` |
| `output` | Output directory for generated client | `node_modules/.prisma/client` |
| `previewFeatures` | Array of preview features to enable | `[]` |
| `binaryTargets` | Platforms to generate binaries for | `["native"]` |
| `engineType` | Query engine type | `"library"` |

#### Multiple Generators

```prisma
generator client {
  provider = "prisma-client-js"
}

generator typegraphql {
  provider = "typegraphql-prisma"
  output   = "./generated/type-graphql"
}

generator dbml {
  provider = "prisma-dbml-generator"
}
```

### Datasource Block

The datasource block specifies your database connection.

```prisma
datasource db {
  provider          = "postgresql"
  url               = env("DATABASE_URL")
  directUrl         = env("DIRECT_URL")        // For migrations (bypasses connection pooler)
  shadowDatabaseUrl = env("SHADOW_DATABASE_URL") // For development migrations
  relationMode      = "prisma"                  // For databases without foreign keys (PlanetScale)
}
```

#### Supported Database Providers

| Provider | Description |
|----------|-------------|
| `postgresql` | PostgreSQL |
| `mysql` | MySQL |
| `sqlite` | SQLite |
| `sqlserver` | Microsoft SQL Server |
| `mongodb` | MongoDB |
| `cockroachdb` | CockroachDB |

#### Connection URL Formats

```bash
# PostgreSQL
DATABASE_URL="postgresql://USER:PASSWORD@HOST:PORT/DATABASE?schema=public"

# MySQL
DATABASE_URL="mysql://USER:PASSWORD@HOST:PORT/DATABASE"

# SQLite
DATABASE_URL="file:./dev.db"

# SQL Server
DATABASE_URL="sqlserver://HOST:PORT;database=DATABASE;user=USER;password=PASSWORD;encrypt=true"

# MongoDB
DATABASE_URL="mongodb+srv://USER:PASSWORD@HOST/DATABASE?retryWrites=true&w=majority"

# CockroachDB
DATABASE_URL="postgresql://USER:PASSWORD@HOST:PORT/DATABASE?sslmode=verify-full"
```

### Model Block

Models represent tables in your database (or collections in MongoDB).

```prisma
model User {
  // Fields (columns)
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String?
  role      Role     @default(USER)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  // Relations
  posts     Post[]
  profile   Profile?

  // Model attributes
  @@index([email, name])
  @@map("users")
}
```

---

## Data Types

### Scalar Types Overview

| Prisma Type | Description |
|-------------|-------------|
| `String` | Variable length text |
| `Boolean` | True or false |
| `Int` | Integer value |
| `BigInt` | Large integer value |
| `Float` | Floating point number |
| `Decimal` | Exact decimal number |
| `DateTime` | Date and time |
| `Json` | JSON data |
| `Bytes` | Binary data |

### Type Mappings by Database

#### PostgreSQL

| Prisma Type | PostgreSQL Type | Notes |
|-------------|-----------------|-------|
| `String` | `text` | Unlimited length |
| `Boolean` | `boolean` | |
| `Int` | `integer` | 4 bytes |
| `BigInt` | `bigint` | 8 bytes |
| `Float` | `double precision` | 8 bytes |
| `Decimal` | `decimal(65,30)` | Exact precision |
| `DateTime` | `timestamp(3)` | Millisecond precision |
| `Json` | `jsonb` | Binary JSON |
| `Bytes` | `bytea` | Binary data |

#### MySQL

| Prisma Type | MySQL Type | Notes |
|-------------|------------|-------|
| `String` | `varchar(191)` | Limited for indexing |
| `Boolean` | `BOOLEAN` | Alias for TINYINT(1) |
| `Int` | `INT` | 4 bytes |
| `BigInt` | `BIGINT` | 8 bytes |
| `Float` | `DOUBLE` | 8 bytes |
| `Decimal` | `DECIMAL(65,30)` | Exact precision |
| `DateTime` | `DATETIME(3)` | Millisecond precision |
| `Json` | `JSON` | Native JSON |
| `Bytes` | `LONGBLOB` | Up to 4GB |

#### SQLite

| Prisma Type | SQLite Type | Notes |
|-------------|-------------|-------|
| `String` | `TEXT` | |
| `Boolean` | `INTEGER` | 0 or 1 |
| `Int` | `INTEGER` | |
| `BigInt` | `INTEGER` | |
| `Float` | `REAL` | |
| `Decimal` | `DECIMAL` | Stored as text |
| `DateTime` | `DATETIME` | Stored as text |
| `Json` | `TEXT` | Stored as text |
| `Bytes` | `BLOB` | |

#### SQL Server

| Prisma Type | SQL Server Type | Notes |
|-------------|-----------------|-------|
| `String` | `nvarchar(1000)` | Unicode |
| `Boolean` | `bit` | 0 or 1 |
| `Int` | `int` | 4 bytes |
| `BigInt` | `bigint` | 8 bytes |
| `Float` | `float(53)` | 8 bytes |
| `Decimal` | `decimal(32,16)` | Exact precision |
| `DateTime` | `datetime2` | High precision |
| `Json` | `nvarchar(max)` | Stored as text |
| `Bytes` | `varbinary(max)` | |

#### MongoDB

| Prisma Type | MongoDB Type | Notes |
|-------------|--------------|-------|
| `String` | `String` | |
| `Boolean` | `Bool` | |
| `Int` | `Int` | 32-bit |
| `BigInt` | `Long` | 64-bit |
| `Float` | `Double` | |
| `Decimal` | `Decimal128` | |
| `DateTime` | `Timestamp` | |
| `Json` | `Object` | |
| `Bytes` | `BinData` | |

### Arrays

Arrays are supported in PostgreSQL, CockroachDB, and MongoDB.

```prisma
model User {
  id       String   @id @default(cuid())
  tags     String[]
  scores   Int[]
  metadata Json[]
}
```

### Unsupported Types

For database-specific types not natively supported by Prisma:

```prisma
model Location {
  id       String                   @id @default(cuid())
  geometry Unsupported("geometry")  // PostGIS geometry type
  point    Unsupported("point")     // MySQL spatial type
}
```

---

## Field Modifiers

### @id

Marks a field as the primary key.

```prisma
model User {
  id String @id @default(cuid())
}

model Post {
  postId Int @id @default(autoincrement())
}
```

### @unique

Creates a unique constraint on the field.

```prisma
model User {
  id       String  @id @default(cuid())
  email    String  @unique
  username String  @unique
  ssn      String? @unique  // Nullable unique field
}
```

### @default

Sets a default value for the field.

```prisma
model User {
  id        String   @id @default(cuid())
  role      Role     @default(USER)
  active    Boolean  @default(true)
  score     Int      @default(0)
  balance   Decimal  @default(0.00)
  createdAt DateTime @default(now())
  data      Json     @default("{}")
  tags      String[] @default([])
}
```

#### Default Value Functions

| Function | Description | Types |
|----------|-------------|-------|
| `autoincrement()` | Auto-incrementing integer | `Int`, `BigInt` |
| `cuid()` | Collision-resistant unique identifier | `String` |
| `uuid()` | Universally unique identifier (v4) | `String` |
| `now()` | Current timestamp | `DateTime` |
| `dbgenerated()` | Database-generated value | Any |
| `sequence()` | CockroachDB sequence | `Int`, `BigInt` |

```prisma
model Example {
  id         String   @id @default(uuid())
  createdAt  DateTime @default(now())
  counter    Int      @default(autoincrement())
  customId   String   @default(dbgenerated("gen_random_uuid()"))
}
```

### @updatedAt

Automatically updates the timestamp when the record is modified.

```prisma
model Post {
  id        String   @id @default(cuid())
  title     String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

### @map

Maps a field name to a different column name in the database.

```prisma
model User {
  id        String   @id @default(cuid()) @map("user_id")
  email     String   @unique @map("email_address")
  firstName String   @map("first_name")
  lastName  String   @map("last_name")
  createdAt DateTime @default(now()) @map("created_at")

  @@map("users")
}
```

### @relation

Defines relationships between models.

```prisma
model User {
  id    String @id @default(cuid())
  posts Post[]
}

model Post {
  id       String @id @default(cuid())
  author   User   @relation(fields: [authorId], references: [id])
  authorId String @map("author_id")
}
```

### @ignore

Ignores a field in Prisma Client (useful for introspection).

```prisma
model User {
  id       String  @id @default(cuid())
  email    String  @unique
  password String  @ignore  // Not exposed in Prisma Client
}
```

### Optional Fields

Use `?` to make a field optional (nullable).

```prisma
model User {
  id       String  @id @default(cuid())
  email    String
  name     String?        // Optional string
  age      Int?           // Optional integer
  bio      String?        // Optional text
  avatar   String?        // Optional URL
}
```

---

## ID Strategies

### Auto-increment

Sequential integer IDs, most efficient for indexing.

```prisma
model Post {
  id Int @id @default(autoincrement())
}

model Comment {
  id BigInt @id @default(autoincrement())  // For large tables
}
```

### CUID (Recommended)

Collision-resistant unique identifiers. Good balance of uniqueness and sortability.

```prisma
model User {
  id String @id @default(cuid())
}
```

CUID characteristics:
- 25 characters long
- URL-safe
- Monotonically increasing (time-based prefix)
- Example: `clh3xxxxxxxxxxxxxxxxxxxxxx`

### UUID

Universally unique identifiers (version 4).

```prisma
model Session {
  id String @id @default(uuid())
}
```

UUID characteristics:
- 36 characters long (with hyphens)
- Completely random
- Example: `550e8400-e29b-41d4-a716-446655440000`

### Database-native UUID

```prisma
model Document {
  id String @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
}
```

### Composite ID

Multi-column primary keys using `@@id`.

```prisma
model PostTag {
  postId String
  tagId  String
  post   Post @relation(fields: [postId], references: [id])
  tag    Tag  @relation(fields: [tagId], references: [id])

  @@id([postId, tagId])
}

model OrderItem {
  orderId   String
  productId String
  quantity  Int
  order     Order   @relation(fields: [orderId], references: [id])
  product   Product @relation(fields: [productId], references: [id])

  @@id([orderId, productId])
}
```

### Named Composite ID

```prisma
model UserRole {
  userId String
  roleId String

  @@id(name: "userRoleId", fields: [userId, roleId])
}
```

### MongoDB ObjectId

```prisma
model User {
  id String @id @default(auto()) @map("_id") @db.ObjectId
}
```

---

## Enums

Enums define a set of allowed values for a field.

### Basic Enum Definition

```prisma
enum Role {
  USER
  ADMIN
  MODERATOR
  SUPER_ADMIN
}

enum Status {
  DRAFT
  PENDING
  PUBLISHED
  ARCHIVED
}

enum Priority {
  LOW
  MEDIUM
  HIGH
  URGENT
}
```

### Using Enums in Models

```prisma
model User {
  id   String @id @default(cuid())
  role Role   @default(USER)
}

model Post {
  id       String   @id @default(cuid())
  status   Status   @default(DRAFT)
  priority Priority @default(MEDIUM)
}
```

### Enum with Database Mapping

```prisma
enum Role {
  USER      @map("user")
  ADMIN     @map("admin")
  MODERATOR @map("moderator")

  @@map("user_role")  // Map enum name to database
}
```

### Optional Enum Fields

```prisma
model Task {
  id       String    @id @default(cuid())
  priority Priority?  // Can be null
}
```

### Enum Arrays (PostgreSQL)

```prisma
model User {
  id          String   @id @default(cuid())
  permissions Role[]   // Array of enum values
}
```

---

## Indexes

Indexes improve query performance by creating optimized data structures.

### Single-Column Index

```prisma
model User {
  id    String @id @default(cuid())
  email String
  name  String

  @@index([email])
  @@index([name])
}
```

### Composite Index

```prisma
model Post {
  id        String   @id @default(cuid())
  authorId  String
  status    Status
  createdAt DateTime @default(now())

  @@index([authorId, status])      // Queries filtering by both
  @@index([authorId, createdAt])   // Queries sorting by date per author
}
```

### Named Index

```prisma
model User {
  id    String @id @default(cuid())
  email String
  name  String

  @@index([email, name], map: "idx_user_email_name")
}
```

### Unique Constraint (@@unique)

```prisma
model Post {
  id     String @id @default(cuid())
  slug   String
  userId String

  @@unique([slug, userId])  // Unique slug per user
}

model TeamMember {
  id     String @id @default(cuid())
  teamId String
  userId String

  @@unique([teamId, userId])  // User can only be in a team once
}
```

### Named Unique Constraint

```prisma
model Subscription {
  id     String @id @default(cuid())
  userId String
  planId String

  @@unique(name: "userPlanUnique", fields: [userId, planId])
}
```

### Full-Text Index (PostgreSQL/MySQL)

Requires `fullTextIndex` preview feature.

```prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["fullTextIndex"]
}

model Post {
  id      String @id @default(cuid())
  title   String
  content String

  @@fulltext([title, content])
}
```

### Full-Text Index with Custom Name

```prisma
model Article {
  id    String @id @default(cuid())
  title String
  body  String

  @@fulltext([title, body], map: "article_fulltext_idx")
}
```

### Index Sort Order

```prisma
model Event {
  id        String   @id @default(cuid())
  startDate DateTime
  endDate   DateTime

  @@index([startDate(sort: Desc)])
  @@index([startDate(sort: Asc), endDate(sort: Desc)])
}
```

### Index Length (MySQL)

```prisma
model Post {
  id      String @id @default(cuid())
  content String @db.Text

  @@index([content(length: 100)])  // Index first 100 characters
}
```

### Index Type (PostgreSQL)

```prisma
model Location {
  id   String @id @default(cuid())
  tags String[]

  @@index([tags], type: Gin)  // GIN index for array searches
}
```

#### Supported Index Types

| Type | Use Case |
|------|----------|
| `BTree` | Default, general purpose |
| `Hash` | Equality comparisons |
| `Gin` | Array/JSONB contains |
| `Gist` | Geometric/full-text |
| `SpGist` | Partitioned data |
| `Brin` | Large sequential data |

### Clustered Index (SQL Server)

```prisma
model User {
  id    String @id @default(cuid())
  email String

  @@index([email], clustered: true)
}
```

---

## Native Database Types

Use `@db.*` to specify exact database column types.

### PostgreSQL Native Types

```prisma
model Product {
  id          String   @id @default(uuid()) @db.Uuid
  name        String   @db.VarChar(255)
  description String   @db.Text
  price       Decimal  @db.Decimal(10, 2)
  quantity    Int      @db.SmallInt
  bigNumber   BigInt   @db.BigInt
  data        Json     @db.JsonB
  raw         Json     @db.Json
  binary      Bytes    @db.ByteA
  char        String   @db.Char(10)
  time        DateTime @db.Time(3)
  timestamp   DateTime @db.Timestamp(6)
  timestamptz DateTime @db.Timestamptz(6)
  date        DateTime @db.Date
  money       Decimal  @db.Money
  inet        String   @db.Inet
  citext      String   @db.Citext
  xml         String   @db.Xml
  oid         Int      @db.Oid
}
```

#### PostgreSQL Type Reference

| Prisma Type | Native Types |
|-------------|--------------|
| `String` | `@db.Text`, `@db.VarChar(n)`, `@db.Char(n)`, `@db.Uuid`, `@db.Inet`, `@db.Citext`, `@db.Xml` |
| `Boolean` | `@db.Boolean` |
| `Int` | `@db.Integer`, `@db.SmallInt`, `@db.Oid` |
| `BigInt` | `@db.BigInt` |
| `Float` | `@db.DoublePrecision`, `@db.Real` |
| `Decimal` | `@db.Decimal(p,s)`, `@db.Money` |
| `DateTime` | `@db.Timestamp(n)`, `@db.Timestamptz(n)`, `@db.Date`, `@db.Time(n)`, `@db.Timetz(n)` |
| `Json` | `@db.Json`, `@db.JsonB` |
| `Bytes` | `@db.ByteA` |

### MySQL Native Types

```prisma
model Product {
  id          String   @id @default(uuid()) @db.VarChar(36)
  name        String   @db.VarChar(255)
  description String   @db.Text
  longText    String   @db.LongText
  mediumText  String   @db.MediumText
  tinyText    String   @db.TinyText
  price       Decimal  @db.Decimal(10, 2)
  quantity    Int      @db.SmallInt
  tinyNum     Int      @db.TinyInt
  mediumNum   Int      @db.MediumInt
  bigNum      BigInt   @db.BigInt
  unsigned    Int      @db.UnsignedInt
  float       Float    @db.Float
  double      Float    @db.Double
  bit         Boolean  @db.Bit(1)
  binary      Bytes    @db.Binary(16)
  varbinary   Bytes    @db.VarBinary(255)
  blob        Bytes    @db.Blob
  date        DateTime @db.Date
  time        DateTime @db.Time(3)
  datetime    DateTime @db.DateTime(3)
  timestamp   DateTime @db.Timestamp(3)
  year        Int      @db.Year
}
```

### SQL Server Native Types

```prisma
model Product {
  id          String   @id @default(uuid())
  name        String   @db.NVarChar(255)
  ascii       String   @db.VarChar(255)
  fixed       String   @db.NChar(10)
  description String   @db.NVarChar(Max)
  text        String   @db.Text
  price       Decimal  @db.Decimal(10, 2)
  money       Decimal  @db.Money
  smallMoney  Decimal  @db.SmallMoney
  quantity    Int      @db.SmallInt
  tinyNum     Int      @db.TinyInt
  bigNum      BigInt   @db.BigInt
  float       Float    @db.Real
  double      Float    @db.Float(53)
  bit         Boolean  @db.Bit
  binary      Bytes    @db.Binary(16)
  varbinary   Bytes    @db.VarBinary(Max)
  image       Bytes    @db.Image
  date        DateTime @db.Date
  time        DateTime @db.Time
  datetime    DateTime @db.DateTime
  datetime2   DateTime @db.DateTime2
  smallDt     DateTime @db.SmallDateTime
  dateTimeOff DateTime @db.DateTimeOffset
  uniqueId    String   @db.UniqueIdentifier
  xml         String   @db.Xml
}
```

### MongoDB Native Types

```prisma
model Product {
  id          String   @id @default(auto()) @map("_id") @db.ObjectId
  name        String
  description String
  price       Float    @db.Double
  quantity    Int      @db.Int
  longNum     BigInt   @db.Long
  data        Json
  binary      Bytes    @db.BinData
  timestamp   DateTime @db.Timestamp
  date        DateTime @db.Date
  objectId    String   @db.ObjectId
  precise     Decimal  @db.Decimal
}
```

---

## Composite Types

Composite types are embedded documents available in MongoDB.

### Defining Composite Types

```prisma
// Only for MongoDB
datasource db {
  provider = "mongodb"
  url      = env("DATABASE_URL")
}

type Address {
  street  String
  city    String
  state   String
  zip     String
  country String @default("USA")
}

type ContactInfo {
  email   String
  phone   String?
  address Address
}

model User {
  id      String      @id @default(auto()) @map("_id") @db.ObjectId
  name    String
  contact ContactInfo
}
```

### Nested Composite Types

```prisma
type GeoPoint {
  latitude  Float
  longitude Float
}

type Address {
  street   String
  city     String
  state    String
  zip      String
  location GeoPoint?
}

type Company {
  name    String
  address Address
}

model User {
  id       String   @id @default(auto()) @map("_id") @db.ObjectId
  name     String
  employer Company?
}
```

### Arrays of Composite Types

```prisma
type OrderItem {
  productId String @db.ObjectId
  name      String
  quantity  Int
  price     Float
}

model Order {
  id        String      @id @default(auto()) @map("_id") @db.ObjectId
  userId    String      @db.ObjectId
  items     OrderItem[]
  total     Float
  createdAt DateTime    @default(now())
}
```

### Optional Composite Fields

```prisma
type SocialLinks {
  twitter   String?
  linkedin  String?
  github    String?
  website   String?
}

model User {
  id      String       @id @default(auto()) @map("_id") @db.ObjectId
  name    String
  social  SocialLinks?
}
```

---

## Multi-Schema Support

Support for multiple database schemas (PostgreSQL, SQL Server, CockroachDB).

### Enabling Multi-Schema

```prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["multiSchema"]
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
  schemas  = ["public", "auth", "billing"]
}
```

### Assigning Models to Schemas

```prisma
model User {
  id    String @id @default(cuid())
  email String @unique

  @@schema("auth")
}

model Session {
  id        String   @id @default(cuid())
  userId    String
  expiresAt DateTime

  @@schema("auth")
}

model Invoice {
  id     String @id @default(cuid())
  amount Decimal

  @@schema("billing")
}

model Product {
  id   String @id @default(cuid())
  name String

  @@schema("public")
}
```

### Enums in Specific Schemas

```prisma
enum UserRole {
  USER
  ADMIN

  @@schema("auth")
}

enum InvoiceStatus {
  PENDING
  PAID
  CANCELLED

  @@schema("billing")
}
```

### Cross-Schema Relations

```prisma
model User {
  id       String    @id @default(cuid())
  invoices Invoice[]

  @@schema("auth")
}

model Invoice {
  id     String @id @default(cuid())
  userId String
  user   User   @relation(fields: [userId], references: [id])

  @@schema("billing")
}
```

---

## Model Attributes

### @@map

Maps the model name to a different table name in the database.

```prisma
model User {
  id    String @id @default(cuid())
  email String @unique

  @@map("users")  // Table name: users
}

model BlogPost {
  id    String @id @default(cuid())
  title String

  @@map("blog_posts")  // Table name: blog_posts
}
```

### @@schema

Assigns a model to a specific database schema.

```prisma
model User {
  id String @id @default(cuid())

  @@schema("auth")
}
```

### @@ignore

Ignores the entire model in Prisma Client (useful during introspection).

```prisma
model LegacyTable {
  id   Int    @id
  data String

  @@ignore  // Not included in Prisma Client
}
```

### @@id

Defines a composite primary key.

```prisma
model PostTag {
  postId String
  tagId  String

  @@id([postId, tagId])
}
```

### @@unique

Defines a composite unique constraint.

```prisma
model Subscription {
  id     String @id @default(cuid())
  userId String
  planId String

  @@unique([userId, planId])
}
```

### @@index

Defines an index on one or more fields.

```prisma
model Post {
  id        String   @id @default(cuid())
  authorId  String
  createdAt DateTime @default(now())

  @@index([authorId])
  @@index([authorId, createdAt])
}
```

### @@fulltext

Defines a full-text search index.

```prisma
model Article {
  id      String @id @default(cuid())
  title   String
  content String

  @@fulltext([title, content])
}
```

### Multiple Attributes Example

```prisma
model Comment {
  id        String   @id @default(cuid())
  postId    String
  authorId  String
  content   String
  createdAt DateTime @default(now())

  post   Post @relation(fields: [postId], references: [id])
  author User @relation(fields: [authorId], references: [id])

  @@unique([postId, authorId, createdAt])
  @@index([postId])
  @@index([authorId])
  @@map("comments")
}
```

---

## Relations

### One-to-One Relations

```prisma
model User {
  id      String   @id @default(cuid())
  email   String   @unique
  profile Profile?
}

model Profile {
  id     String @id @default(cuid())
  bio    String
  userId String @unique
  user   User   @relation(fields: [userId], references: [id])
}
```

#### Required One-to-One

```prisma
model User {
  id      String  @id @default(cuid())
  email   String  @unique
  profile Profile
}

model Profile {
  id     String @id @default(cuid())
  bio    String
  userId String @unique
  user   User   @relation(fields: [userId], references: [id])
}
```

### One-to-Many Relations

```prisma
model User {
  id    String @id @default(cuid())
  email String @unique
  posts Post[]
}

model Post {
  id       String @id @default(cuid())
  title    String
  authorId String
  author   User   @relation(fields: [authorId], references: [id])
}
```

#### With Optional Foreign Key

```prisma
model Post {
  id       String  @id @default(cuid())
  title    String
  authorId String?
  author   User?   @relation(fields: [authorId], references: [id])
}
```

### Many-to-Many Relations

#### Implicit Many-to-Many

Prisma automatically creates a join table.

```prisma
model Post {
  id         String     @id @default(cuid())
  title      String
  categories Category[]
}

model Category {
  id    String @id @default(cuid())
  name  String @unique
  posts Post[]
}
```

#### Explicit Many-to-Many

Define your own join table for additional fields.

```prisma
model Post {
  id         String         @id @default(cuid())
  title      String
  categories CategoriesOnPosts[]
}

model Category {
  id    String              @id @default(cuid())
  name  String              @unique
  posts CategoriesOnPosts[]
}

model CategoriesOnPosts {
  postId     String
  categoryId String
  assignedAt DateTime @default(now())
  assignedBy String

  post     Post     @relation(fields: [postId], references: [id])
  category Category @relation(fields: [categoryId], references: [id])

  @@id([postId, categoryId])
}
```

### Self-Relations

#### One-to-One Self-Relation

```prisma
model User {
  id          String  @id @default(cuid())
  name        String
  successorId String? @unique
  successor   User?   @relation("Succession", fields: [successorId], references: [id])
  predecessor User?   @relation("Succession")
}
```

#### One-to-Many Self-Relation

```prisma
model User {
  id         String  @id @default(cuid())
  name       String
  managerId  String?
  manager    User?   @relation("Management", fields: [managerId], references: [id])
  employees  User[]  @relation("Management")
}
```

#### Many-to-Many Self-Relation

```prisma
model User {
  id          String @id @default(cuid())
  name        String
  followedBy  User[] @relation("Follows")
  following   User[] @relation("Follows")
}
```

### Named Relations

Required when you have multiple relations between the same models.

```prisma
model User {
  id            String @id @default(cuid())
  name          String
  writtenPosts  Post[] @relation("WrittenPosts")
  editedPosts   Post[] @relation("EditedPosts")
}

model Post {
  id       String @id @default(cuid())
  title    String
  authorId String
  editorId String?
  author   User   @relation("WrittenPosts", fields: [authorId], references: [id])
  editor   User?  @relation("EditedPosts", fields: [editorId], references: [id])
}
```

### Relation Fields with Custom Foreign Key Names

```prisma
model Post {
  id       String @id @default(cuid())
  title    String
  authorId String @map("author_id")
  author   User   @relation(fields: [authorId], references: [id])

  @@map("posts")
}
```

### Multi-Field Relations

```prisma
model Post {
  id        String @id @default(cuid())
  title     String
  authorFirstName String
  authorLastName  String
  author    User   @relation(fields: [authorFirstName, authorLastName], references: [firstName, lastName])
}

model User {
  id        String @id @default(cuid())
  firstName String
  lastName  String
  posts     Post[]

  @@unique([firstName, lastName])
}
```

---

## Referential Actions

Control what happens when referenced records are updated or deleted.

### onDelete Actions

```prisma
model Post {
  id       String @id @default(cuid())
  authorId String
  author   User   @relation(fields: [authorId], references: [id], onDelete: Cascade)
}
```

#### Available onDelete Actions

| Action | Description |
|--------|-------------|
| `Cascade` | Delete the child records when parent is deleted |
| `Restrict` | Prevent deletion if child records exist |
| `NoAction` | Similar to Restrict (database-level) |
| `SetNull` | Set foreign key to null (field must be optional) |
| `SetDefault` | Set foreign key to default value |

### onUpdate Actions

```prisma
model Post {
  id       String @id @default(cuid())
  authorId String
  author   User   @relation(fields: [authorId], references: [id], onUpdate: Cascade)
}
```

#### Available onUpdate Actions

| Action | Description |
|--------|-------------|
| `Cascade` | Update child foreign keys when parent key changes |
| `Restrict` | Prevent update if child records exist |
| `NoAction` | Similar to Restrict (database-level) |
| `SetNull` | Set foreign key to null |
| `SetDefault` | Set foreign key to default value |

### Combined Example

```prisma
model User {
  id       String    @id @default(cuid())
  email    String    @unique
  posts    Post[]
  comments Comment[]
  profile  Profile?
}

model Profile {
  id     String @id @default(cuid())
  bio    String
  userId String @unique
  user   User   @relation(fields: [userId], references: [id], onDelete: Cascade, onUpdate: Cascade)
}

model Post {
  id       String    @id @default(cuid())
  title    String
  authorId String
  author   User      @relation(fields: [authorId], references: [id], onDelete: Cascade)
  comments Comment[]
}

model Comment {
  id       String @id @default(cuid())
  content  String
  postId   String
  authorId String
  post     Post   @relation(fields: [postId], references: [id], onDelete: Cascade)
  author   User   @relation(fields: [authorId], references: [id], onDelete: SetNull)
}
```

### SetNull with Optional Field

```prisma
model Comment {
  id       String  @id @default(cuid())
  content  String
  authorId String?  // Must be optional for SetNull
  author   User?    @relation(fields: [authorId], references: [id], onDelete: SetNull)
}
```

---

## Field-level Documentation

### Triple-slash Comments (///)

Triple-slash comments are preserved in the generated Prisma Client as JSDoc comments.

```prisma
/// Represents a user in the system
/// @deprecated Use Customer model instead
model User {
  /// Unique identifier for the user
  id String @id @default(cuid())

  /// User's email address - must be unique
  email String @unique

  /// User's display name - optional
  name String?

  /// When the user account was created
  createdAt DateTime @default(now())

  /// When the user last updated their profile
  updatedAt DateTime @updatedAt

  /// User's role in the system
  /// @default USER
  role Role @default(USER)
}

/// Available user roles
enum Role {
  /// Standard user with basic permissions
  USER

  /// Administrator with full access
  ADMIN

  /// Content moderator
  MODERATOR
}
```

### Double-slash Comments (//)

Regular comments are NOT included in the generated client.

```prisma
// This is a regular comment - not in generated client
model User {
  id String @id @default(cuid())  // inline comment
}
```

### Documentation Best Practices

```prisma
/// Customer entity representing a paying user
///
/// Customers are created when a user completes their first purchase.
/// They have associated billing information and subscription details.
///
/// @example
/// const customer = await prisma.customer.create({
///   data: { userId: '...', stripeCustomerId: '...' }
/// })
model Customer {
  /// Primary key - uses CUID for better distribution
  id String @id @default(cuid())

  /// Reference to the user account
  /// @see User
  userId String @unique

  /// Stripe customer ID for payment processing
  /// Format: cus_xxxxxxxxxxxxx
  stripeCustomerId String @unique

  /// Current subscription plan
  /// NULL if customer has no active subscription
  subscriptionId String?

  /// ISO 4217 currency code for billing
  /// @default "USD"
  currency String @default("USD")

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
}
```

---

## Preview Features

Preview features are experimental and may change.

### Enabling Preview Features

```prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["fullTextSearch", "fullTextIndex", "metrics", "tracing"]
}
```

### Available Preview Features

| Feature | Description |
|---------|-------------|
| `driverAdapters` | Use custom database drivers |
| `fullTextIndex` | Full-text search indexes |
| `fullTextSearch` | Full-text search queries |
| `metrics` | Prisma Client metrics |
| `multiSchema` | Multiple database schemas |
| `postgresqlExtensions` | PostgreSQL extensions |
| `tracing` | OpenTelemetry tracing |
| `views` | Database views support |
| `relationJoins` | JOIN-based relation loading |
| `omitApi` | Omit fields from queries |
| `prismaSchemaFolder` | Split schema into multiple files |

### Full-Text Search Example

```prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["fullTextSearch", "fullTextIndex"]
}

model Post {
  id      String @id @default(cuid())
  title   String
  content String

  @@fulltext([title, content])
}
```

### PostgreSQL Extensions

```prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["postgresqlExtensions"]
}

datasource db {
  provider   = "postgresql"
  url        = env("DATABASE_URL")
  extensions = [uuid_ossp(schema: "public"), pgcrypto]
}
```

### Views Support

```prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["views"]
}

view UserStats {
  userId     String @unique
  postCount  Int
  totalLikes Int
}
```

### Multi-File Schema

```prisma
// In prisma/schema.prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["prismaSchemaFolder"]
}
```

Then organize schemas:
```
prisma/
  schema/
    main.prisma
    user.prisma
    post.prisma
    comment.prisma
```

---

## Environment Variables

### Using env() Function

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

### Multiple Environment Variables

```prisma
datasource db {
  provider          = "postgresql"
  url               = env("DATABASE_URL")
  directUrl         = env("DIRECT_URL")
  shadowDatabaseUrl = env("SHADOW_DATABASE_URL")
}
```

### .env File Format

```bash
# .env file in project root

# PostgreSQL
DATABASE_URL="postgresql://user:password@localhost:5432/mydb?schema=public"

# Direct connection (bypasses connection pooler)
DIRECT_URL="postgresql://user:password@localhost:5432/mydb?schema=public"

# Shadow database for migrations
SHADOW_DATABASE_URL="postgresql://user:password@localhost:5432/mydb_shadow?schema=public"

# MySQL
DATABASE_URL="mysql://user:password@localhost:3306/mydb"

# SQLite
DATABASE_URL="file:./dev.db"

# SQL Server
DATABASE_URL="sqlserver://localhost:1433;database=mydb;user=sa;password=Password123;encrypt=true"

# MongoDB
DATABASE_URL="mongodb+srv://user:password@cluster.mongodb.net/mydb?retryWrites=true&w=majority"
```

### Connection String Parameters

#### PostgreSQL Parameters

```bash
DATABASE_URL="postgresql://user:password@host:5432/db?schema=public&connection_limit=5&pool_timeout=10&sslmode=require&sslcert=/path/to/cert"
```

| Parameter | Description |
|-----------|-------------|
| `schema` | Database schema to use |
| `connection_limit` | Maximum connections in pool |
| `pool_timeout` | Connection pool timeout (seconds) |
| `sslmode` | SSL mode (require, verify-ca, verify-full) |
| `connect_timeout` | Connection timeout (seconds) |

#### MySQL Parameters

```bash
DATABASE_URL="mysql://user:password@host:3306/db?connection_limit=5&socket_timeout=10&ssl_mode=REQUIRED"
```

### Environment-Specific Configuration

```bash
# .env.development
DATABASE_URL="postgresql://localhost:5432/myapp_dev"

# .env.test
DATABASE_URL="postgresql://localhost:5432/myapp_test"

# .env.production
DATABASE_URL="postgresql://prod-host:5432/myapp_prod?sslmode=require"
```

---

## Prisma CLI Commands

### Schema Validation and Formatting

```bash
# Validate schema syntax and logic
npx prisma validate

# Format schema file
npx prisma format

# Output: Formatted Prisma schema to schema.prisma
```

### Database Push (Development)

Push schema changes directly to the database without migrations.

```bash
# Push schema to database
npx prisma db push

# Force push (reset data if needed)
npx prisma db push --force-reset

# Skip generators
npx prisma db push --skip-generate

# Accept data loss
npx prisma db push --accept-data-loss
```

### Database Pull (Introspection)

Generate schema from existing database.

```bash
# Pull schema from database
npx prisma db pull

# Force overwrite
npx prisma db pull --force

# Print to stdout
npx prisma db pull --print
```

### Migrations

```bash
# Create new migration
npx prisma migrate dev --name init

# Create migration without applying
npx prisma migrate dev --create-only

# Apply pending migrations (production)
npx prisma migrate deploy

# Reset database and apply all migrations
npx prisma migrate reset

# Check migration status
npx prisma migrate status

# Resolve failed migration
npx prisma migrate resolve --applied "migration_name"
npx prisma migrate resolve --rolled-back "migration_name"
```

### Generate Prisma Client

```bash
# Generate client
npx prisma generate

# Watch mode (regenerate on schema changes)
npx prisma generate --watch
```

### Database Seed

```bash
# Run seed script
npx prisma db seed

# Configure seed in package.json
# {
#   "prisma": {
#     "seed": "ts-node prisma/seed.ts"
#   }
# }
```

### Prisma Studio

```bash
# Open Prisma Studio (GUI)
npx prisma studio

# Custom port
npx prisma studio --port 5555

# Custom browser
npx prisma studio --browser firefox
```

### Database Commands

```bash
# Execute raw SQL
npx prisma db execute --file ./script.sql

# Seed the database
npx prisma db seed
```

### Debugging

```bash
# Enable query logging
DEBUG="prisma:query" npx prisma ...

# Enable all Prisma logs
DEBUG="prisma:*" npx prisma ...

# Check Prisma version
npx prisma version

# Get environment info
npx prisma debug
```

---

## Best Practices

### Naming Conventions

```prisma
// Models: PascalCase, singular
model User {}
model BlogPost {}
model OrderItem {}

// Fields: camelCase
model User {
  id        String @id
  firstName String
  lastName  String
  createdAt DateTime
}

// Enums: PascalCase for name, SCREAMING_SNAKE_CASE for values
enum UserRole {
  ADMIN
  SUPER_ADMIN
  REGULAR_USER
}

// Relations: camelCase, descriptive
model Post {
  author   User @relation(...)
  comments Comment[]
}
```

### Database Column Naming

Use `@map` for snake_case database columns while keeping camelCase in Prisma.

```prisma
model User {
  id        String   @id @default(cuid()) @map("user_id")
  firstName String   @map("first_name")
  lastName  String   @map("last_name")
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  @@map("users")
}
```

### ID Strategy Selection

```prisma
// Use CUID for distributed systems (recommended default)
model User {
  id String @id @default(cuid())
}

// Use UUID when interoperability is needed
model ExternalResource {
  id String @id @default(uuid())
}

// Use autoincrement for simple, single-database apps
model LocalRecord {
  id Int @id @default(autoincrement())
}

// Use composite IDs for join tables
model UserRole {
  userId String
  roleId String
  @@id([userId, roleId])
}
```

### Index Optimization

```prisma
model Post {
  id        String   @id @default(cuid())
  authorId  String
  status    Status
  createdAt DateTime @default(now())
  title     String

  // Index foreign keys
  @@index([authorId])

  // Index commonly filtered fields
  @@index([status])

  // Composite index for common query patterns
  @@index([authorId, status])
  @@index([authorId, createdAt])

  // Covering index for specific queries
  @@index([status, createdAt, title])
}
```

### Relation Best Practices

```prisma
// Always specify referential actions explicitly
model Post {
  id       String @id @default(cuid())
  authorId String
  author   User   @relation(fields: [authorId], references: [id], onDelete: Cascade, onUpdate: Cascade)
}

// Use explicit many-to-many when you need extra fields
model Enrollment {
  studentId  String
  courseId   String
  enrolledAt DateTime @default(now())
  grade      Float?

  student Student @relation(fields: [studentId], references: [id])
  course  Course  @relation(fields: [courseId], references: [id])

  @@id([studentId, courseId])
}
```

### Schema Organization

```prisma
// Group related models together

// ============ User Management ============

model User {
  id String @id @default(cuid())
  // ...
}

model Profile {
  id String @id @default(cuid())
  // ...
}

model Session {
  id String @id @default(cuid())
  // ...
}

// ============ Content ============

model Post {
  id String @id @default(cuid())
  // ...
}

model Comment {
  id String @id @default(cuid())
  // ...
}

// ============ Enums ============

enum Role {
  USER
  ADMIN
}

enum Status {
  DRAFT
  PUBLISHED
}
```

### Soft Deletes

```prisma
model User {
  id        String    @id @default(cuid())
  email     String    @unique
  deletedAt DateTime?  // null = not deleted

  @@index([deletedAt])
}

// Query non-deleted users
// prisma.user.findMany({ where: { deletedAt: null } })
```

### Audit Fields

```prisma
model Post {
  id        String   @id @default(cuid())
  title     String
  content   String

  // Audit fields
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  createdBy String?
  updatedBy String?
  version   Int      @default(1)
}
```

### Avoiding Common Pitfalls

```prisma
// DO: Use explicit relation names for multiple relations
model User {
  id           String @id @default(cuid())
  writtenPosts Post[] @relation("Author")
  editedPosts  Post[] @relation("Editor")
}

model Post {
  id       String @id @default(cuid())
  authorId String
  editorId String?
  author   User   @relation("Author", fields: [authorId], references: [id])
  editor   User?  @relation("Editor", fields: [editorId], references: [id])
}

// DO: Make optional fields nullable
model Post {
  id          String  @id @default(cuid())
  description String? // Can be null
}

// DON'T: Use optional on required foreign keys
// This causes runtime errors
model Post {
  authorId String? // Only if posts can exist without authors
  author   User?   @relation(fields: [authorId], references: [id])
}
```

### Type Safety with Native Types

```prisma
model Product {
  id          String  @id @default(cuid())

  // Specify exact precision for money
  price       Decimal @db.Decimal(10, 2)

  // Use appropriate string lengths
  sku         String  @db.VarChar(50)
  name        String  @db.VarChar(255)
  description String  @db.Text

  // Use appropriate integer sizes
  quantity    Int     @db.SmallInt
  viewCount   Int     @db.Integer

  // Use database-native UUID when possible
  externalId  String  @db.Uuid
}
```

### Environment-Specific Schema

```prisma
// Use environment variables for flexibility
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")

  // For serverless environments with connection pooling
  directUrl = env("DIRECT_URL")
}

// For platforms without foreign key support (PlanetScale)
datasource db {
  provider     = "mysql"
  url          = env("DATABASE_URL")
  relationMode = "prisma"
}
```

### Migration Safety

```bash
# Always review migrations before applying
npx prisma migrate dev --create-only
# Review the generated SQL in prisma/migrations/

# Use separate shadow database
# DATABASE_URL for main database
# SHADOW_DATABASE_URL for migration testing

# In production, always use deploy (not dev)
npx prisma migrate deploy
```

### Performance Considerations

```prisma
// Add indexes for:
// 1. Foreign keys (not auto-indexed in MySQL)
// 2. Fields used in WHERE clauses
// 3. Fields used in ORDER BY clauses
// 4. Fields used in JOINs

model Comment {
  id        String   @id @default(cuid())
  postId    String
  authorId  String
  createdAt DateTime @default(now())

  @@index([postId])      // Foreign key
  @@index([authorId])    // Foreign key
  @@index([createdAt])   // Sorting
}

// Consider partial indexes for large tables
// (requires raw SQL migration)

// Use appropriate field types to minimize storage
model Metric {
  id        String   @id @default(cuid())
  value     Float    @db.Real      // 4 bytes vs 8 for double
  count     Int      @db.SmallInt  // 2 bytes vs 4 for int
  timestamp DateTime @db.Timestamp // Without timezone if not needed
}
```

---

## Quick Reference

### Common Field Patterns

```prisma
// Primary key with CUID
id String @id @default(cuid())

// Primary key with auto-increment
id Int @id @default(autoincrement())

// Primary key with UUID
id String @id @default(uuid())

// Unique email
email String @unique

// Optional field
name String?

// Default value
active Boolean @default(true)

// Enum with default
role Role @default(USER)

// Timestamps
createdAt DateTime @default(now())
updatedAt DateTime @updatedAt

// Array (PostgreSQL/MongoDB)
tags String[]

// JSON data
metadata Json

// Decimal for money
price Decimal @db.Decimal(10, 2)
```

### Common Model Patterns

```prisma
// Standard model
model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

// Join table
model PostTag {
  postId String
  tagId  String
  post   Post @relation(fields: [postId], references: [id])
  tag    Tag  @relation(fields: [tagId], references: [id])
  @@id([postId, tagId])
}

// Soft delete model
model Item {
  id        String    @id @default(cuid())
  name      String
  deletedAt DateTime?
  @@index([deletedAt])
}
```

### Essential Commands

```bash
# Development workflow
npx prisma validate          # Check schema
npx prisma format            # Format schema
npx prisma generate          # Generate client
npx prisma db push           # Push to dev DB
npx prisma migrate dev       # Create migration
npx prisma studio            # Open GUI

# Production workflow
npx prisma migrate deploy    # Apply migrations
npx prisma generate          # Generate client

# Database inspection
npx prisma db pull           # Introspect DB
npx prisma migrate status    # Check status
```

---

## Additional Resources

- [Prisma Documentation](https://www.prisma.io/docs)
- [Prisma Schema Reference](https://www.prisma.io/docs/reference/api-reference/prisma-schema-reference)
- [Prisma Client API](https://www.prisma.io/docs/reference/api-reference/prisma-client-reference)
- [Prisma Migrate](https://www.prisma.io/docs/concepts/components/prisma-migrate)
- [Prisma GitHub](https://github.com/prisma/prisma)
