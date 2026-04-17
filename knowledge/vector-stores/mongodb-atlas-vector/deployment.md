# MongoDB Atlas Vector Search -- Deployment and Operations

## Overview

This guide covers the operational aspects of deploying MongoDB Atlas Vector Search in production: Atlas cluster setup, vector search index creation via UI, CLI, and Terraform, auto-embedding pipelines with Atlas Triggers, sharding strategies, the difference between Atlas Search and Atlas Vector Search, security configuration, and client patterns with pymongo and Motor.

---

## Atlas Cluster Setup

### Minimum Requirements for Vector Search

Atlas Vector Search requires:
- Atlas cluster tier M10 or higher (not available on M0/M2/M5 free/shared tiers)
- MongoDB 6.0.11+ or 7.0+ (7.0+ recommended for $rankFusion and latest features)
- Atlas Search nodes (automatically provisioned, or dedicated search nodes for production)

### Cluster Sizing Guidelines

| Vectors | Dimensions | Quantization | Recommended Tier | Estimated Cost/mo |
|---|---|---|---|---|
| < 100K | 1536 | None | M10 (2 GB RAM) | $60 |
| 100K - 500K | 1536 | None | M30 (8 GB RAM) | $300 |
| 500K - 2M | 1536 | Scalar | M40 (16 GB RAM) | $600 |
| 2M - 10M | 1536 | Scalar | M50 (32 GB RAM) | $1,200 |
| 10M - 50M | 1536 | Scalar | M60 (64 GB RAM) | $2,800 |
| 50M+ | 1536 | Binary | M80+ or dedicated search nodes | $5,000+ |

### Dedicated Search Nodes

For production workloads, Atlas offers dedicated search nodes that run the mongot process on separate infrastructure from the database nodes. This prevents vector search from competing with CRUD operations for resources.

```
Atlas Configuration (via UI or API):
  Cluster Tier: M50
  Search Nodes: 2x S30 (dedicated)
  
Benefits:
  - Independent scaling of search and database
  - Search workload does not affect database latency
  - Separate memory for vector indexes
```

---

## Vector Search Index Creation

### Via Atlas UI

1. Navigate to your cluster in the Atlas UI.
2. Click "Atlas Search" in the left sidebar.
3. Click "Create Search Index".
4. Select "JSON Editor" (not Visual Editor for vector search).
5. Paste the index definition:

```json
{
  "name": "vector_index",
  "type": "vectorSearch",
  "definition": {
    "fields": [
      {
        "type": "vector",
        "path": "embedding",
        "numDimensions": 1536,
        "similarity": "cosine",
        "quantization": "scalar"
      },
      {
        "type": "filter",
        "path": "metadata.tenant_id"
      },
      {
        "type": "filter",
        "path": "metadata.category"
      },
      {
        "type": "filter",
        "path": "created_at"
      }
    ]
  }
}
```

6. Select the target database and collection.
7. Click "Create Search Index".

The index builds asynchronously. Monitor progress in the Atlas UI -- it shows percentage complete and estimated time remaining.

### Via Atlas CLI (atlascli)

```bash
# Install Atlas CLI
brew install mongodb-atlas-cli  # macOS
# or download from https://www.mongodb.com/docs/atlas/cli/stable/install-atlas-cli/

# Authenticate
atlas auth login

# Create vector search index from JSON file
atlas clusters search indexes create \
    --clusterName my-cluster \
    --file vector-index-definition.json \
    --projectId 64a1b2c3d4e5f6a7b8c9d0e1

# List search indexes
atlas clusters search indexes list \
    --clusterName my-cluster \
    --db mydb \
    --collection documents \
    --projectId 64a1b2c3d4e5f6a7b8c9d0e1

# Describe a specific index
atlas clusters search indexes describe vector_index \
    --clusterName my-cluster \
    --db mydb \
    --collection documents \
    --projectId 64a1b2c3d4e5f6a7b8c9d0e1

# Delete an index
atlas clusters search indexes delete vector_index \
    --clusterName my-cluster \
    --db mydb \
    --collection documents \
    --projectId 64a1b2c3d4e5f6a7b8c9d0e1 \
    --force
```

### Via Terraform

```hcl
# main.tf

terraform {
  required_providers {
    mongodbatlas = {
      source  = "mongodb/mongodbatlas"
      version = "~> 1.15"
    }
  }
}

provider "mongodbatlas" {
  public_key  = var.atlas_public_key
  private_key = var.atlas_private_key
}

resource "mongodbatlas_cluster" "main" {
  project_id = var.project_id
  name       = "vector-search-cluster"

  provider_name               = "AWS"
  provider_region_name        = "US_EAST_1"
  provider_instance_size_name = "M50"

  mongo_db_major_version = "7.0"

  advanced_configuration {
    javascript_enabled = false
  }
}

resource "mongodbatlas_search_index" "vector_index" {
  project_id   = var.project_id
  cluster_name = mongodbatlas_cluster.main.name

  name            = "vector_index"
  database        = "mydb"
  collection_name = "documents"
  type            = "vectorSearch"

  fields = jsonencode([
    {
      type          = "vector"
      path          = "embedding"
      numDimensions = 1536
      similarity    = "cosine"
      quantization  = "scalar"
    },
    {
      type = "filter"
      path = "metadata.tenant_id"
    },
    {
      type = "filter"
      path = "metadata.category"
    },
    {
      type = "filter"
      path = "created_at"
    }
  ])

  depends_on = [mongodbatlas_cluster.main]
}

# Also create a text search index for hybrid search
resource "mongodbatlas_search_index" "text_index" {
  project_id   = var.project_id
  cluster_name = mongodbatlas_cluster.main.name

  name            = "text_index"
  database        = "mydb"
  collection_name = "documents"
  type            = "search"

  mappings_dynamic = false
  mappings_fields = jsonencode({
    content = {
      type     = "string"
      analyzer = "lucene.standard"
    }
    title = {
      type     = "string"
      analyzer = "lucene.standard"
    }
  })

  depends_on = [mongodbatlas_cluster.main]
}

# Output the connection string
output "connection_string" {
  value     = mongodbatlas_cluster.main.connection_strings[0].standard_srv
  sensitive = true
}
```

```bash
# Deploy
terraform init
terraform plan
terraform apply
```

### Via pymongo (Atlas Admin API)

```python
from pymongo import MongoClient
from pymongo.operations import SearchIndexModel

client = MongoClient("mongodb+srv://cluster0.example.mongodb.net/mydb")
db = client["mydb"]
collection = db["documents"]

# Create vector search index
vector_index = SearchIndexModel(
    name="vector_index",
    type="vectorSearch",
    definition={
        "fields": [
            {
                "type": "vector",
                "path": "embedding",
                "numDimensions": 1536,
                "similarity": "cosine",
                "quantization": "scalar"
            },
            {
                "type": "filter",
                "path": "metadata.tenant_id"
            },
            {
                "type": "filter",
                "path": "metadata.category"
            }
        ]
    }
)

# This creates the index asynchronously
collection.create_search_index(vector_index)

# List existing search indexes
for index in collection.list_search_indexes():
    print(f"Index: {index['name']}, Status: {index['status']}")
```

---

## Atlas Triggers for Auto-Embedding

### Trigger Configuration

Atlas Triggers watch for collection changes and execute functions in response. This is the recommended pattern for automatically embedding documents on insert or update.

```javascript
// Trigger Configuration (via Atlas UI or App Services CLI):
// Name: auto-embed-documents
// Type: Database Trigger
// Collection: documents
// Operation Types: Insert, Update, Replace
// Full Document: Enabled
// Full Document Before Change: Disabled

// Trigger Function:
exports = async function(changeEvent) {
    const doc = changeEvent.fullDocument;
    const collection = context.services
        .get("mongodb-atlas")
        .db("mydb")
        .collection("documents");

    // Skip if embedding already exists and content has not changed
    if (changeEvent.operationType === "update") {
        const updatedFields = changeEvent.updateDescription?.updatedFields || {};
        const contentChanged = "content" in updatedFields || "title" in updatedFields;
        if (!contentChanged && doc.embedding && doc.embedding.length > 0) {
            return;
        }
    }

    // Build text to embed
    const parts = [];
    if (doc.title) parts.push(doc.title);
    if (doc.content) parts.push(doc.content);
    const text = parts.join("\n\n");

    if (!text) {
        console.log(`Document ${doc._id} has no text to embed, skipping.`);
        return;
    }

    try {
        // Call OpenAI embedding API
        const response = await context.http.post({
            url: "https://api.openai.com/v1/embeddings",
            headers: {
                "Authorization": [
                    `Bearer ${context.values.get("OPENAI_API_KEY")}`
                ],
                "Content-Type": ["application/json"]
            },
            body: JSON.stringify({
                model: "text-embedding-3-small",
                input: text.substring(0, 8000)  // Truncate to model limit
            })
        });

        const body = JSON.parse(response.body.text());

        if (body.error) {
            console.error(`OpenAI error: ${body.error.message}`);
            return;
        }

        const embedding = body.data[0].embedding;

        // Update document with embedding
        await collection.updateOne(
            { _id: doc._id },
            {
                $set: {
                    embedding: embedding,
                    embedded_at: new Date(),
                    embedding_model: "text-embedding-3-small",
                    embedding_dimensions: embedding.length
                }
            }
        );

        console.log(`Embedded document ${doc._id} (${embedding.length} dims)`);
    } catch (err) {
        console.error(`Failed to embed document ${doc._id}: ${err.message}`);
        // Optionally mark the document as failed
        await collection.updateOne(
            { _id: doc._id },
            { $set: { embedding_error: err.message, embedding_failed_at: new Date() } }
        );
    }
};
```

### Batch Embedding with Atlas Triggers

For high-volume inserts, individual triggers per document can be expensive (each calls the OpenAI API separately). Use a scheduled trigger for batch processing:

```javascript
// Scheduled Trigger: runs every 5 minutes
// Name: batch-embed-documents

exports = async function() {
    const collection = context.services
        .get("mongodb-atlas")
        .db("mydb")
        .collection("documents");

    const BATCH_SIZE = 100;
    const MAX_BATCHES = 10;  // Process up to 1000 docs per run

    for (let batch = 0; batch < MAX_BATCHES; batch++) {
        // Find documents without embeddings
        const docs = await collection.find(
            {
                embedding: { $exists: false },
                embedding_error: { $exists: false }
            }
        ).limit(BATCH_SIZE).toArray();

        if (docs.length === 0) break;

        // Build batch of texts
        const texts = docs.map(doc => {
            const parts = [];
            if (doc.title) parts.push(doc.title);
            if (doc.content) parts.push(doc.content);
            return parts.join("\n\n").substring(0, 8000);
        });

        try {
            // Batch embed (OpenAI supports batch input)
            const response = await context.http.post({
                url: "https://api.openai.com/v1/embeddings",
                headers: {
                    "Authorization": [
                        `Bearer ${context.values.get("OPENAI_API_KEY")}`
                    ],
                    "Content-Type": ["application/json"]
                },
                body: JSON.stringify({
                    model: "text-embedding-3-small",
                    input: texts
                })
            });

            const body = JSON.parse(response.body.text());

            if (body.error) {
                console.error(`OpenAI batch error: ${body.error.message}`);
                break;
            }

            // Bulk update with embeddings
            const bulkOps = body.data.map((embData, i) => ({
                updateOne: {
                    filter: { _id: docs[i]._id },
                    update: {
                        $set: {
                            embedding: embData.embedding,
                            embedded_at: new Date(),
                            embedding_model: "text-embedding-3-small"
                        }
                    }
                }
            }));

            await collection.bulkWrite(bulkOps);
            console.log(`Batch ${batch}: embedded ${docs.length} documents`);
        } catch (err) {
            console.error(`Batch ${batch} failed: ${err.message}`);
            break;
        }
    }
};
```

---

## Sharding

### When to Shard

Shard when a single Atlas replica set cannot handle the data volume or query throughput:
- **> 50M vectors**: single node memory becomes prohibitive
- **> 500 write ops/sec** for embedding updates
- **Regional distribution** needed for latency

### Shard Key Selection

```python
# Shard by tenant_id (good for multi-tenant applications)
db.admin.command({
    "shardCollection": "mydb.documents",
    "key": {"metadata.tenant_id": "hashed"}
})

# Shard by category + _id (good for range queries on category)
db.admin.command({
    "shardCollection": "mydb.documents",
    "key": {"metadata.category": 1, "_id": 1}
})
```

**Important**: Atlas Vector Search works with sharded collections. The `$vectorSearch` aggregation stage is dispatched to all shards, and results are merged by the mongos router. This means:

- Each shard builds its own vector search index.
- `numCandidates` applies per shard (so with 3 shards and numCandidates=150, total candidates explored = 450).
- `limit` is applied after merge (final result count = limit).

### Chunk Distribution Monitoring

```python
# Check shard distribution
stats = db.command("collStats", "documents")
for shard_name, shard_stats in stats.get("shards", {}).items():
    print(f"Shard {shard_name}: {shard_stats['count']} documents, "
          f"{shard_stats['size'] / 1e9:.1f} GB")
```

---

## Atlas Search vs Atlas Vector Search

These are two different features in Atlas. Understanding the distinction is critical.

| Feature | Atlas Search | Atlas Vector Search |
|---|---|---|
| Purpose | Full-text search (BM25) | Vector similarity search (ANN) |
| Aggregation stage | `$search` | `$vectorSearch` |
| Index type | `search` | `vectorSearch` |
| Data indexed | Text fields | Vector (array of floats) fields |
| Underlying engine | Lucene (inverted index) | Lucene (HNSW graph) |
| Scoring | BM25, custom scoring | Cosine / dot product / L2 |
| Hybrid search | Via `$rankFusion` with `$vectorSearch` | Via `$rankFusion` with `$search` |

### Creating Both Indexes

```python
from pymongo.operations import SearchIndexModel

# Vector search index
vector_idx = SearchIndexModel(
    name="vector_index",
    type="vectorSearch",
    definition={
        "fields": [{
            "type": "vector",
            "path": "embedding",
            "numDimensions": 1536,
            "similarity": "cosine"
        }]
    }
)

# Full-text search index (for hybrid with $rankFusion)
text_idx = SearchIndexModel(
    name="text_index",
    type="search",
    definition={
        "mappings": {
            "dynamic": False,
            "fields": {
                "content": {
                    "type": "string",
                    "analyzer": "lucene.standard"
                },
                "title": {
                    "type": "string",
                    "analyzer": "lucene.standard"
                }
            }
        }
    }
)

# Create both indexes
collection.create_search_index(vector_idx)
collection.create_search_index(text_idx)
```

---

## Security

### VPC Peering

```bash
# Create VPC peering connection via Atlas CLI
atlas networking peering create aws \
    --projectId <project_id> \
    --atlasCidrBlock 192.168.248.0/21 \
    --vpcId vpc-0abc123def456 \
    --awsAccountId 123456789012 \
    --region us-east-1 \
    --routeTableCidrBlock 10.0.0.0/16
```

### Private Endpoints (AWS PrivateLink)

```hcl
# Terraform: Atlas Private Endpoint
resource "mongodbatlas_privatelink_endpoint" "main" {
  project_id    = var.project_id
  provider_name = "AWS"
  region        = "us-east-1"
}

resource "aws_vpc_endpoint" "atlas" {
  vpc_id             = aws_vpc.main.id
  service_name       = mongodbatlas_privatelink_endpoint.main.endpoint_service_name
  vpc_endpoint_type  = "Interface"
  subnet_ids         = [aws_subnet.private.id]
  security_group_ids = [aws_security_group.atlas.id]
}

resource "mongodbatlas_privatelink_endpoint_service" "main" {
  project_id          = var.project_id
  private_link_id     = mongodbatlas_privatelink_endpoint.main.id
  endpoint_service_id = aws_vpc_endpoint.atlas.id
  provider_name       = "AWS"
}
```

### Encryption

| Layer | Mechanism | Configuration |
|---|---|---|
| In-transit | TLS 1.2+ | Enforced by default on Atlas |
| At-rest | AES-256 | Enabled by default, CMK optional |
| Field-level | Client-Side Field Level Encryption | Via pymongo CSFLE |

### Database User Authentication

```python
# Connect with X.509 certificate authentication
client = MongoClient(
    "mongodb+srv://cluster0.example.mongodb.net/mydb",
    tls=True,
    tlsCertificateKeyFile="/path/to/client.pem",
    tlsCAFile="/path/to/ca.pem",
    authMechanism="MONGODB-X509"
)

# Connect with SCRAM-SHA-256
client = MongoClient(
    "mongodb+srv://user:password@cluster0.example.mongodb.net/mydb",
    authMechanism="SCRAM-SHA-256"
)
```

---

## pymongo Client Patterns

### Connection Best Practices

```python
from pymongo import MongoClient
from pymongo.read_concern import ReadConcern
from pymongo.write_concern import WriteConcern
from pymongo.read_preferences import ReadPreference

# Production connection with recommended settings
client = MongoClient(
    "mongodb+srv://user:password@cluster0.example.mongodb.net/",
    maxPoolSize=50,
    minPoolSize=10,
    maxIdleTimeMS=30000,
    connectTimeoutMS=5000,
    serverSelectionTimeoutMS=5000,
    retryWrites=True,
    retryReads=True,
    w="majority",
    readPreference="secondaryPreferred",  # Read from secondaries for vector search
    compressors=["zstd", "snappy"]
)

db = client["mydb"]
collection = db["documents"]
```

### Retry Pattern for Vector Search

```python
import time
from pymongo.errors import ServerSelectionTimeoutError, OperationFailure


def vector_search_with_retry(
    collection,
    query_embedding: list[float],
    limit: int = 10,
    max_retries: int = 3,
    backoff_base: float = 0.5
):
    """Execute vector search with exponential backoff retry."""
    for attempt in range(max_retries):
        try:
            pipeline = [
                {
                    "$vectorSearch": {
                        "index": "vector_index",
                        "path": "embedding",
                        "queryVector": query_embedding,
                        "numCandidates": limit * 15,
                        "limit": limit
                    }
                },
                {
                    "$project": {
                        "content": 1,
                        "metadata": 1,
                        "score": {"$meta": "vectorSearchScore"}
                    }
                }
            ]
            return list(collection.aggregate(pipeline))
        except (ServerSelectionTimeoutError, OperationFailure) as e:
            if attempt == max_retries - 1:
                raise
            wait = backoff_base * (2 ** attempt)
            print(f"Retry {attempt + 1}/{max_retries} after {wait}s: {e}")
            time.sleep(wait)
```

### Embedding + Search Pipeline

```python
from pymongo import MongoClient
from openai import OpenAI

mongo_client = MongoClient("mongodb+srv://cluster0.example.mongodb.net/mydb")
openai_client = OpenAI()
collection = mongo_client["mydb"]["documents"]


def ingest_document(doc: dict) -> str:
    """Ingest a document: embed and store."""
    text = f"{doc.get('title', '')}\n\n{doc.get('content', '')}"
    response = openai_client.embeddings.create(
        model="text-embedding-3-small",
        input=text
    )
    embedding = response.data[0].embedding

    result = collection.insert_one({
        **doc,
        "embedding": embedding,
        "embedding_model": "text-embedding-3-small",
        "embedded_at": None  # Will be set by trigger or here
    })
    return str(result.inserted_id)


def search(query: str, limit: int = 10, filters: dict = None) -> list[dict]:
    """Semantic search with optional metadata filters."""
    response = openai_client.embeddings.create(
        model="text-embedding-3-small",
        input=query
    )
    query_embedding = response.data[0].embedding

    vector_search_stage = {
        "$vectorSearch": {
            "index": "vector_index",
            "path": "embedding",
            "queryVector": query_embedding,
            "numCandidates": limit * 15,
            "limit": limit
        }
    }

    if filters:
        vector_search_stage["$vectorSearch"]["filter"] = filters

    pipeline = [
        vector_search_stage,
        {
            "$project": {
                "content": 1,
                "title": 1,
                "metadata": 1,
                "score": {"$meta": "vectorSearchScore"},
                "embedding": 0
            }
        }
    ]

    return list(collection.aggregate(pipeline))
```

### Motor Async Client Patterns

```python
import asyncio
from motor.motor_asyncio import AsyncIOMotorClient
from openai import AsyncOpenAI

motor_client = AsyncIOMotorClient(
    "mongodb+srv://user:password@cluster0.example.mongodb.net/",
    maxPoolSize=50,
    minPoolSize=10,
    retryWrites=True,
    retryReads=True
)

openai_client = AsyncOpenAI()
db = motor_client["mydb"]
collection = db["documents"]


async def async_search(query: str, limit: int = 10) -> list[dict]:
    """Async vector search."""
    response = await openai_client.embeddings.create(
        model="text-embedding-3-small",
        input=query
    )
    query_embedding = response.data[0].embedding

    pipeline = [
        {
            "$vectorSearch": {
                "index": "vector_index",
                "path": "embedding",
                "queryVector": query_embedding,
                "numCandidates": limit * 15,
                "limit": limit
            }
        },
        {
            "$project": {
                "content": 1,
                "title": 1,
                "score": {"$meta": "vectorSearchScore"}
            }
        }
    ]

    return await collection.aggregate(pipeline).to_list(length=limit)


async def concurrent_search(queries: list[str]) -> list[list[dict]]:
    """Run multiple searches concurrently."""
    tasks = [async_search(q) for q in queries]
    return await asyncio.gather(*tasks)
```

---

## Monitoring

### Key Metrics to Watch

```python
# Check vector search index status
for idx in collection.list_search_indexes():
    print(f"Index: {idx['name']}")
    print(f"  Status: {idx['status']}")        # READY, BUILDING, FAILED
    print(f"  Queryable: {idx['queryable']}")   # Can be queried during rebuild
    if 'latestDefinition' in idx:
        fields = idx['latestDefinition'].get('fields', [])
        for f in fields:
            print(f"  Field: {f['path']} ({f['type']})")
```

### Atlas Performance Advisor

Atlas provides query performance insights including:
- Slow vector search queries (> configurable threshold)
- Index utilization statistics
- Suggested index improvements
- Query shapes and patterns

Access via: Atlas UI > Cluster > Performance Advisor

---

## Common Pitfalls

1. **Using M0/M2/M5 tiers**: Atlas Vector Search is not available on free or shared tiers. You need M10+ for vector search.

2. **Not waiting for index build completion**: after creating a vector search index, it builds asynchronously. Queries return empty results until the index is `READY`. Monitor with `list_search_indexes()`.

3. **Missing filter field declarations**: filter fields must be declared in the vector search index definition. If you filter on `metadata.tenant_id` but did not include it as a `filter` type field, the filter is ignored.

4. **Not using scalar quantization for large datasets**: without quantization, each 1536-dim vector uses ~6 KB. At 10M documents, vector data alone is 60 GB. Enable `"quantization": "scalar"` for 4x reduction.

5. **Sharding without considering numCandidates**: with sharded collections, `numCandidates` is per-shard. If you have 3 shards and set numCandidates=150, each shard explores 150 candidates. Set numCandidates based on per-shard vector count, not total.

6. **Not creating a separate text index for hybrid search**: `$rankFusion` requires both a `vectorSearch` index and a `search` index. They are separate indexes on separate field types.

7. **Connection pool exhaustion**: vector search queries can be slower than CRUD operations. With high concurrency, ensure `maxPoolSize` is large enough (default 100 for pymongo).

---

## References

- Atlas Vector Search documentation: https://www.mongodb.com/docs/atlas/atlas-vector-search/
- Atlas CLI: https://www.mongodb.com/docs/atlas/cli/stable/
- MongoDB Atlas Terraform provider: https://registry.terraform.io/providers/mongodb/mongodbatlas/
- Atlas App Services Triggers: https://www.mongodb.com/docs/atlas/app-services/triggers/
- pymongo documentation: https://pymongo.readthedocs.io/
- Motor async driver: https://motor.readthedocs.io/
- Atlas Security: https://www.mongodb.com/docs/atlas/security/
