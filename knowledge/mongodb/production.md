# MongoDB Production Guide

> Comprehensive guide for MongoDB schema design, indexes, and production deployment.

## Table of Contents

1. [Schema Design](#schema-design)
2. [Indexing Strategies](#indexing-strategies)
3. [Query Optimization](#query-optimization)
4. [Production Configuration](#production-configuration)
5. [Replication and High Availability](#replication-and-high-availability)
6. [Backup and Recovery](#backup-and-recovery)
7. [Security Hardening](#security-hardening)
8. [Monitoring and Performance](#monitoring-and-performance)

---

## Schema Design

### Document Structure Patterns

```javascript
// Embedded Documents - One-to-Few, data accessed together
const userSchema = {
  _id: ObjectId("..."),
  name: "John Doe",
  email: "john@example.com",
  // Embedded addresses (one-to-few)
  addresses: [
    {
      type: "home",
      street: "123 Main St",
      city: "New York",
      zipCode: "10001"
    },
    {
      type: "work",
      street: "456 Office Ave",
      city: "New York",
      zipCode: "10002"
    }
  ],
  // Embedded profile (one-to-one)
  profile: {
    bio: "Software developer",
    avatar: "https://...",
    socialLinks: {
      twitter: "@johndoe",
      github: "johndoe"
    }
  }
};

// References - One-to-Many, large or frequently changing data
const orderSchema = {
  _id: ObjectId("..."),
  userId: ObjectId("..."),  // Reference to user
  items: [
    {
      productId: ObjectId("..."),  // Reference to product
      name: "Product Name",        // Denormalized for display
      price: 29.99,                // Snapshot at time of order
      quantity: 2
    }
  ],
  total: 59.98,
  status: "pending",
  createdAt: ISODate("2024-01-15T10:00:00Z")
};

// Bucket Pattern - Time series, IoT data
const sensorDataSchema = {
  _id: ObjectId("..."),
  sensorId: "sensor-001",
  date: ISODate("2024-01-15"),
  readings: [
    { time: ISODate("2024-01-15T00:00:00Z"), value: 23.5 },
    { time: ISODate("2024-01-15T00:01:00Z"), value: 23.6 },
    // ... up to ~200 readings per document
  ],
  count: 200,
  sum: 4720,
  min: 22.1,
  max: 25.3
};
```

### Schema Design Patterns

```javascript
// Polymorphic Pattern - Different types in same collection
const notificationSchema = {
  _id: ObjectId("..."),
  type: "email",  // or "sms", "push"
  userId: ObjectId("..."),
  createdAt: ISODate("..."),
  // Type-specific fields
  email: {
    to: "user@example.com",
    subject: "Welcome",
    body: "..."
  }
};

// Computed Pattern - Pre-computed aggregations
const productSchema = {
  _id: ObjectId("..."),
  name: "Product Name",
  reviews: [
    { userId: ObjectId("..."), rating: 5, comment: "Great!" },
    { userId: ObjectId("..."), rating: 4, comment: "Good" }
  ],
  // Pre-computed values (updated on review changes)
  reviewStats: {
    count: 2,
    averageRating: 4.5,
    distribution: { 1: 0, 2: 0, 3: 0, 4: 1, 5: 1 }
  }
};

// Subset Pattern - Split frequently and rarely accessed data
// Main collection - hot data
const movieSchema = {
  _id: ObjectId("..."),
  title: "Movie Title",
  year: 2024,
  director: "Director Name",
  rating: 8.5,
  // Only top 10 cast members
  cast: ["Actor 1", "Actor 2", "Actor 3"],
  genres: ["Action", "Sci-Fi"]
};

// Detail collection - cold data
const movieDetailsSchema = {
  _id: ObjectId("..."),
  movieId: ObjectId("..."),
  fullCast: [/* all cast members */],
  crew: [/* full crew list */],
  plotSummary: "...",
  trivia: [/* trivia facts */]
};

// Outlier Pattern - Handle large arrays
const bookSchema = {
  _id: ObjectId("..."),
  title: "Popular Book",
  // First 50 reviews embedded
  reviews: [/* ... */],
  reviewCount: 10000,
  hasOverflow: true  // Indicates more reviews in separate collection
};

// Overflow collection
const bookReviewsOverflowSchema = {
  _id: ObjectId("..."),
  bookId: ObjectId("..."),
  reviews: [/* reviews 51-100 */],
  page: 2
};
```

### Validation Schema

```javascript
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email", "name", "createdAt"],
      properties: {
        email: {
          bsonType: "string",
          pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$",
          description: "Must be a valid email"
        },
        name: {
          bsonType: "string",
          minLength: 2,
          maxLength: 100
        },
        age: {
          bsonType: "int",
          minimum: 0,
          maximum: 150
        },
        role: {
          enum: ["user", "admin", "moderator"],
          description: "Must be a valid role"
        },
        addresses: {
          bsonType: "array",
          maxItems: 10,
          items: {
            bsonType: "object",
            required: ["street", "city"],
            properties: {
              street: { bsonType: "string" },
              city: { bsonType: "string" },
              zipCode: { bsonType: "string" }
            }
          }
        },
        createdAt: {
          bsonType: "date"
        }
      }
    }
  },
  validationLevel: "strict",
  validationAction: "error"
});
```

---

## Indexing Strategies

### Index Types

```javascript
// Single field index
db.users.createIndex({ email: 1 });  // Ascending
db.users.createIndex({ createdAt: -1 });  // Descending

// Compound index (order matters!)
db.orders.createIndex({ userId: 1, status: 1, createdAt: -1 });

// Multikey index (for arrays)
db.products.createIndex({ tags: 1 });

// Text index (full-text search)
db.articles.createIndex(
  { title: "text", content: "text" },
  { weights: { title: 10, content: 1 } }
);

// Geospatial index
db.locations.createIndex({ coordinates: "2dsphere" });

// Hashed index (for sharding)
db.users.createIndex({ _id: "hashed" });

// TTL index (auto-expire documents)
db.sessions.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 3600 }  // 1 hour
);

// Partial index (index subset of documents)
db.orders.createIndex(
  { status: 1, createdAt: -1 },
  { partialFilterExpression: { status: "pending" } }
);

// Sparse index (only documents with field)
db.users.createIndex(
  { phoneNumber: 1 },
  { sparse: true }
);

// Unique index
db.users.createIndex(
  { email: 1 },
  { unique: true }
);

// Case-insensitive index
db.users.createIndex(
  { email: 1 },
  { collation: { locale: "en", strength: 2 } }
);

// Wildcard index (flexible schema)
db.logs.createIndex({ "metadata.$**": 1 });
```

### Index Selection Guidelines

```javascript
// ESR Rule: Equality, Sort, Range
// Best index order: Fields used in equality first, then sort, then range

// Query: { status: "active", createdAt: { $gte: date } } sort: { name: 1 }
db.users.createIndex({ status: 1, name: 1, createdAt: 1 });

// Covered queries (index contains all needed fields)
db.orders.createIndex({ userId: 1, status: 1, total: 1 });
// This query can be covered:
db.orders.find(
  { userId: ObjectId("..."), status: "completed" },
  { total: 1, _id: 0 }
);

// Index intersection (MongoDB can use multiple indexes)
db.products.createIndex({ category: 1 });
db.products.createIndex({ price: 1 });
// Can combine for: { category: "electronics", price: { $lt: 100 } }
```

### Index Management

```javascript
// List indexes
db.users.getIndexes();

// Index stats
db.users.aggregate([{ $indexStats: {} }]);

// Drop index
db.users.dropIndex("email_1");

// Hide index (test impact before dropping)
db.users.hideIndex("email_1");
db.users.unhideIndex("email_1");

// Background index creation
db.users.createIndex(
  { email: 1 },
  { background: true }  // Deprecated in 4.2+, use default
);

// Check index usage
db.users.explain("executionStats").find({ email: "test@example.com" });
```

---

## Query Optimization

### Aggregation Pipeline

```javascript
// Efficient aggregation patterns
db.orders.aggregate([
  // $match first - filters early, uses indexes
  { $match: { status: "completed", createdAt: { $gte: ISODate("2024-01-01") } } },

  // $project early - reduce document size
  { $project: { userId: 1, total: 1, items: 1 } },

  // $lookup - join with other collection
  { $lookup: {
      from: "users",
      localField: "userId",
      foreignField: "_id",
      as: "user",
      pipeline: [
        { $project: { name: 1, email: 1 } }
      ]
  }},
  { $unwind: "$user" },

  // $group - aggregations
  { $group: {
      _id: "$user._id",
      userName: { $first: "$user.name" },
      totalOrders: { $sum: 1 },
      totalSpent: { $sum: "$total" },
      avgOrderValue: { $avg: "$total" }
  }},

  // $sort - sort results
  { $sort: { totalSpent: -1 } },

  // $limit - limit results
  { $limit: 10 }
]);

// $facet - multiple aggregations in one query
db.products.aggregate([
  { $match: { category: "electronics" } },
  { $facet: {
      priceRanges: [
        { $bucket: {
            groupBy: "$price",
            boundaries: [0, 100, 500, 1000, Infinity],
            default: "Other",
            output: { count: { $sum: 1 } }
        }}
      ],
      brands: [
        { $group: { _id: "$brand", count: { $sum: 1 } } },
        { $sort: { count: -1 } },
        { $limit: 10 }
      ],
      totalProducts: [
        { $count: "count" }
      ]
  }}
]);

// $graphLookup - recursive lookups
db.employees.aggregate([
  { $match: { name: "CEO" } },
  { $graphLookup: {
      from: "employees",
      startWith: "$_id",
      connectFromField: "_id",
      connectToField: "managerId",
      as: "directReports",
      maxDepth: 2,
      depthField: "level"
  }}
]);
```

### Query Performance

```javascript
// Use explain() to analyze queries
db.orders.explain("executionStats").find({
  userId: ObjectId("..."),
  status: "pending"
}).sort({ createdAt: -1 });

// Key metrics to check:
// - totalDocsExamined vs nReturned (should be close)
// - executionTimeMillis
// - stage: IXSCAN (good) vs COLLSCAN (bad)

// Avoid $where and JavaScript execution
// Bad
db.users.find({ $where: "this.firstName + ' ' + this.lastName === 'John Doe'" });
// Good
db.users.find({ firstName: "John", lastName: "Doe" });

// Use projection to limit returned fields
db.users.find(
  { status: "active" },
  { name: 1, email: 1 }  // Only return these fields
);

// Batch operations
db.orders.bulkWrite([
  { insertOne: { document: { ... } } },
  { updateOne: { filter: { _id: 1 }, update: { $set: { status: "shipped" } } } },
  { deleteOne: { filter: { _id: 2 } } }
], { ordered: false });  // Unordered for better performance
```

---

## Production Configuration

### mongod.conf Production Settings

```yaml
# /etc/mongod.conf
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true
  wiredTiger:
    engineConfig:
      cacheSizeGB: 8  # 50% of RAM for dedicated server
      journalCompressor: snappy
    collectionConfig:
      blockCompressor: snappy
    indexConfig:
      prefixCompression: true

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
  logRotate: rename
  verbosity: 0
  component:
    command:
      verbosity: 1

net:
  port: 27017
  bindIp: 0.0.0.0
  maxIncomingConnections: 65536
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/mongodb/ssl/server.pem
    CAFile: /etc/mongodb/ssl/ca.pem

security:
  authorization: enabled
  keyFile: /etc/mongodb/keyfile
  javascriptEnabled: false

operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100

replication:
  replSetName: rs0
  oplogSizeMB: 10240

setParameter:
  enableLocalhostAuthBypass: false
```

### Docker Production Setup

```yaml
# docker-compose.mongodb.yml
services:
  mongodb:
    image: mongo:7
    container_name: mongodb
    restart: unless-stopped

    command: >
      mongod
      --replSet rs0
      --bind_ip_all
      --auth
      --keyFile /etc/mongodb/keyfile

    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD_FILE: /run/secrets/mongo_password

    secrets:
      - mongo_password

    volumes:
      - mongodb-data:/data/db
      - mongodb-config:/data/configdb
      - ./keyfile:/etc/mongodb/keyfile:ro
      - ./mongod.conf:/etc/mongod.conf:ro

    deploy:
      resources:
        limits:
          memory: 16G
          cpus: "4"
        reservations:
          memory: 8G
          cpus: "2"

    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

    ulimits:
      nofile:
        soft: 64000
        hard: 64000
      nproc:
        soft: 64000
        hard: 64000

secrets:
  mongo_password:
    file: ./secrets/mongo_password.txt

volumes:
  mongodb-data:
  mongodb-config:
```

---

## Replication and High Availability

### Replica Set Configuration

```javascript
// Initialize replica set
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo1:27017", priority: 2 },
    { _id: 1, host: "mongo2:27017", priority: 1 },
    { _id: 2, host: "mongo3:27017", priority: 1 }
  ]
});

// Add arbiter (for even number of nodes)
rs.addArb("mongo-arbiter:27017");

// Configure read preference
// Primary - default, all reads from primary
// PrimaryPreferred - primary, fallback to secondary
// Secondary - all reads from secondary
// SecondaryPreferred - secondary, fallback to primary
// Nearest - lowest network latency

// In application connection string
const uri = "mongodb://mongo1:27017,mongo2:27017,mongo3:27017/mydb?replicaSet=rs0&readPreference=secondaryPreferred";

// Configure write concern
db.orders.insertOne(
  { item: "test" },
  { writeConcern: { w: "majority", j: true, wtimeout: 5000 } }
);

// Check replica set status
rs.status();
rs.printReplicationInfo();
rs.printSecondaryReplicationInfo();
```

### Sharded Cluster

```javascript
// Enable sharding on database
sh.enableSharding("mydb");

// Shard collection with hashed key
sh.shardCollection("mydb.users", { _id: "hashed" });

// Shard collection with ranged key
sh.shardCollection("mydb.orders", { userId: 1, createdAt: 1 });

// Check sharding status
sh.status();

// Balancer operations
sh.isBalancerRunning();
sh.startBalancer();
sh.stopBalancer();

// Zone sharding (data locality)
sh.addShardTag("shard0", "US");
sh.addShardTag("shard1", "EU");
sh.addTagRange(
  "mydb.users",
  { region: "US" },
  { region: "US\uffff" },
  "US"
);
```

---

## Backup and Recovery

### mongodump/mongorestore

```bash
#!/bin/bash
# backup-mongodb.sh

BACKUP_DIR="/backups/mongodb"
DATE=$(date +%Y-%m-%d_%H%M%S)
MONGO_URI="mongodb://backup:password@localhost:27017"

mkdir -p "$BACKUP_DIR"

# Full backup
mongodump \
  --uri="$MONGO_URI" \
  --authenticationDatabase=admin \
  --gzip \
  --archive="$BACKUP_DIR/full_$DATE.gz"

# Backup specific database
mongodump \
  --uri="$MONGO_URI" \
  --db=mydb \
  --gzip \
  --archive="$BACKUP_DIR/mydb_$DATE.gz"

# Backup with oplog for point-in-time recovery
mongodump \
  --uri="$MONGO_URI" \
  --oplog \
  --gzip \
  --archive="$BACKUP_DIR/full_oplog_$DATE.gz"

# Cleanup old backups
find "$BACKUP_DIR" -type f -name "*.gz" -mtime +30 -delete

echo "Backup completed: $DATE"
```

```bash
# Restore commands
# Full restore
mongorestore \
  --uri="mongodb://admin:password@localhost:27017" \
  --authenticationDatabase=admin \
  --gzip \
  --archive="/backups/mongodb/full_2024-01-15.gz"

# Restore with oplog replay
mongorestore \
  --uri="mongodb://admin:password@localhost:27017" \
  --oplogReplay \
  --gzip \
  --archive="/backups/mongodb/full_oplog_2024-01-15.gz"

# Restore specific database
mongorestore \
  --uri="mongodb://admin:password@localhost:27017" \
  --nsInclude="mydb.*" \
  --gzip \
  --archive="/backups/mongodb/full_2024-01-15.gz"
```

### Continuous Backup with Change Streams

```javascript
// Application-level continuous backup
const pipeline = [
  { $match: { operationType: { $in: ["insert", "update", "replace", "delete"] } } }
];

const changeStream = db.collection("orders").watch(pipeline, {
  fullDocument: "updateLookup"
});

changeStream.on("change", async (change) => {
  // Store change to backup system
  await backupService.recordChange({
    collection: "orders",
    operation: change.operationType,
    documentKey: change.documentKey,
    fullDocument: change.fullDocument,
    updateDescription: change.updateDescription,
    timestamp: change.clusterTime
  });
});
```

---

## Security Hardening

### User Management

```javascript
// Create admin user
use admin
db.createUser({
  user: "admin",
  pwd: passwordPrompt(),
  roles: [
    { role: "userAdminAnyDatabase", db: "admin" },
    { role: "readWriteAnyDatabase", db: "admin" },
    { role: "dbAdminAnyDatabase", db: "admin" },
    { role: "clusterAdmin", db: "admin" }
  ]
});

// Create application user with minimal privileges
use mydb
db.createUser({
  user: "app_user",
  pwd: passwordPrompt(),
  roles: [
    { role: "readWrite", db: "mydb" }
  ]
});

// Create read-only user
db.createUser({
  user: "readonly",
  pwd: passwordPrompt(),
  roles: [
    { role: "read", db: "mydb" }
  ]
});

// Create custom role
db.createRole({
  role: "orderManager",
  privileges: [
    {
      resource: { db: "mydb", collection: "orders" },
      actions: ["find", "insert", "update"]
    },
    {
      resource: { db: "mydb", collection: "products" },
      actions: ["find"]
    }
  ],
  roles: []
});

// Field-level encryption
const clientEncryption = new ClientEncryption(mongoClient, {
  keyVaultNamespace: "encryption.__keyVault",
  kmsProviders: {
    local: { key: masterKey }
  }
});

const encryptedFieldsMap = {
  "mydb.users": {
    fields: [
      {
        path: "ssn",
        bsonType: "string",
        keyId: dataKeyId,
        queries: { queryType: "equality" }
      }
    ]
  }
};
```

### Network Security

```javascript
// IP whitelist
net:
  bindIp: 10.0.1.100,127.0.0.1

// TLS/SSL configuration
net:
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/mongodb/ssl/server.pem
    CAFile: /etc/mongodb/ssl/ca.pem
    allowInvalidCertificates: false
    allowConnectionsWithoutCertificates: false
```

---

## Monitoring and Performance

### Key Metrics

```javascript
// Server status
db.serverStatus();

// Connection stats
db.serverStatus().connections;

// Memory usage
db.serverStatus().mem;
db.serverStatus().wiredTiger.cache;

// Operation counters
db.serverStatus().opcounters;

// Current operations
db.currentOp({ active: true });

// Collection stats
db.orders.stats();

// Index usage
db.orders.aggregate([{ $indexStats: {} }]);

// Profiler
db.setProfilingLevel(1, { slowms: 100 });
db.system.profile.find().sort({ ts: -1 }).limit(10);
```

### Prometheus Metrics

```yaml
# docker-compose with mongodb-exporter
services:
  mongodb-exporter:
    image: percona/mongodb_exporter:0.40
    environment:
      MONGODB_URI: "mongodb://monitor:password@mongodb:27017"
    command:
      - '--collect-all'
      - '--mongodb.direct-connect=true'
    ports:
      - "9216:9216"
```

### Alert Rules

```yaml
# prometheus/alerts/mongodb.yml
groups:
  - name: mongodb-alerts
    rules:
      - alert: MongoDBDown
        expr: mongodb_up == 0
        for: 1m
        labels:
          severity: critical

      - alert: MongoDBHighConnections
        expr: mongodb_connections{state="current"} / mongodb_connections{state="available"} > 0.8
        for: 5m
        labels:
          severity: warning

      - alert: MongoDBReplicationLag
        expr: mongodb_mongod_replset_member_replication_lag > 60
        for: 5m
        labels:
          severity: warning

      - alert: MongoDBSlowQueries
        expr: rate(mongodb_mongod_metrics_query_executor_total{state="scanned_objects"}[5m]) / rate(mongodb_mongod_metrics_query_executor_total{state="returned"}[5m]) > 100
        for: 10m
        labels:
          severity: warning
```

---

## Quick Reference

### Common Operations

```javascript
// CRUD
db.collection.insertOne({ ... });
db.collection.insertMany([{ ... }, { ... }]);
db.collection.find({ field: value });
db.collection.findOne({ _id: ObjectId("...") });
db.collection.updateOne({ _id: id }, { $set: { field: value } });
db.collection.updateMany({ status: "old" }, { $set: { status: "new" } });
db.collection.deleteOne({ _id: id });
db.collection.deleteMany({ status: "archived" });

// Update operators
$set, $unset, $inc, $push, $pull, $addToSet, $pop, $rename

// Query operators
$eq, $ne, $gt, $gte, $lt, $lte, $in, $nin
$and, $or, $not, $nor
$exists, $type, $regex
$elemMatch, $size, $all
```

### Connection String

```
mongodb://user:password@host1:27017,host2:27017,host3:27017/database?
  replicaSet=rs0
  &authSource=admin
  &readPreference=secondaryPreferred
  &w=majority
  &retryWrites=true
  &maxPoolSize=100
  &connectTimeoutMS=10000
  &socketTimeoutMS=30000
```
