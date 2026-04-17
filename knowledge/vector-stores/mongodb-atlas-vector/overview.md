# MongoDB Atlas Vector Search Deep Guide

## Overview

MongoDB Atlas Vector Search integrates vector similarity search directly into MongoDB's aggregation pipeline via the `$vectorSearch` stage. It runs on the same Atlas cluster as your operational data, eliminating the need for a separate vector database. Atlas Vector Search uses a Lucene-based HNSW implementation under the hood, supports scalar and binary quantization, filter pre/post strategies, and hybrid search combining `$vectorSearch` with full-text `$search` through `$rankFusion`. This guide covers architecture, index configuration, query patterns, hybrid retrieval, and production patterns.

For basic MongoDB operations and CRUD, see the companion MongoDB guides. This document focuses exclusively on vector search capabilities available in Atlas.

---

## Architecture

### How Atlas Vector Search Works

Atlas Vector Search runs as a separate process (mongot) alongside the mongod process on each Atlas node. The mongot process manages the Lucene-based vector indexes and handles search queries.

```
Client Request
    |
    v
mongos (Router)
    |
    v
mongod (Data Node)          mongot (Search Node)
- Stores documents           - Maintains vector indexes
- Handles CRUD               - Executes $vectorSearch
- Sends change streams        - Returns scored doc IDs
  to mongot for indexing      - Runs on same node
```

Key architectural properties:
- **Eventual consistency**: mongot indexes lag behind mongod by milliseconds to seconds. New documents are not immediately searchable.
- **Co-located**: mongot runs on the same Atlas node, so no network hop for search queries.
- **Lucene HNSW**: the underlying index is Apache Lucene's HNSW implementation, the same engine used by Elasticsearch and OpenSearch.
- **No separate infrastructure**: vector search is an Atlas feature, not a separate service.

### Index Sync via Change Streams

Atlas Vector Search uses MongoDB Change Streams internally to keep the vector index synchronized with the collection. When a document is inserted, updated, or deleted, the change stream event triggers re-indexing in mongot.

```python
# You do NOT need to manage change streams yourself for indexing.
# Atlas handles this automatically. However, you can use change streams
# for your own auto-embedding pipeline:

from pymongo import MongoClient
from openai import OpenAI

client = MongoClient("mongodb+srv://cluster0.example.mongodb.net/mydb")
openai_client = OpenAI()

db = client["mydb"]
collection = db["documents"]

# Watch for inserts and updates that lack an embedding
pipeline = [
    {"$match": {
        "operationType": {"$in": ["insert", "update", "replace"]},
        "fullDocument.embedding": {"$exists": False}
    }}
]

with collection.watch(pipeline, full_document="updateLookup") as stream:
    for change in stream:
        doc = change["fullDocument"]
        text = doc.get("content", "")
        if not text:
            continue

        # Generate embedding
        response = openai_client.embeddings.create(
            model="text-embedding-3-small",
            input=text
        )
        embedding = response.data[0].embedding

        # Update document with embedding
        collection.update_one(
            {"_id": doc["_id"]},
            {"$set": {"embedding": embedding}}
        )
```

---

## Vector Search Index Definition

### JSON Index Schema

Atlas Vector Search indexes are defined as JSON documents, not through MongoDB's `createIndex()` command. They are managed through the Atlas UI, Atlas CLI, or Terraform.

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
        "path": "metadata.category"
      },
      {
        "type": "filter",
        "path": "metadata.tenant_id"
      },
      {
        "type": "filter",
        "path": "created_at"
      }
    ]
  }
}
```

### Field Types

| Field Type | Purpose | Notes |
|---|---|---|
| `vector` | The embedding field to index | Specify dimensions, similarity metric |
| `filter` | Fields available for pre-filtering | Must be declared in the index definition |

### Similarity Metrics

| Metric | When to Use | Formula |
|---|---|---|
| `cosine` | Most text embeddings (default) | 1 - cosine_distance |
| `dotProduct` | Pre-normalized embeddings | dot(a, b) |
| `euclidean` | When magnitude matters | L2 distance |

### Quantization Options

Atlas Vector Search supports quantization to reduce memory usage and improve throughput.

| Quantization | Compression | Recall Impact | When to Use |
|---|---|---|---|
| None | 1x (baseline) | Baseline | Small datasets, maximum quality |
| `scalar` | ~4x | <1% loss | Default recommendation for production |
| `binary` | ~32x | 2-5% loss | Very large datasets, cost-sensitive |

```json
{
  "type": "vector",
  "path": "embedding",
  "numDimensions": 1536,
  "similarity": "cosine",
  "quantization": "scalar"
}
```

Scalar quantization converts float32 values to int8, reducing storage from 4 bytes to 1 byte per dimension. Binary quantization converts each dimension to a single bit. Both use automatic rescoring against full-precision vectors to maintain recall.

---

## $vectorSearch Aggregation Stage

### Basic Vector Search

```python
from pymongo import MongoClient
import numpy as np

client = MongoClient("mongodb+srv://cluster0.example.mongodb.net/mydb")
db = client["mydb"]
collection = db["documents"]

query_embedding = [0.1, 0.2, 0.3]  # Your 1536-dim query vector

results = collection.aggregate([
    {
        "$vectorSearch": {
            "index": "vector_index",
            "path": "embedding",
            "queryVector": query_embedding,
            "numCandidates": 150,
            "limit": 10
        }
    },
    {
        "$project": {
            "content": 1,
            "metadata": 1,
            "score": {"$meta": "vectorSearchScore"}
        }
    }
])

for doc in results:
    print(f"Score: {doc['score']:.4f} - {doc['content'][:80]}")
```

### Key Parameters

| Parameter | Description | Guidance |
|---|---|---|
| `numCandidates` | Number of candidates HNSW explores | Set 10-20x `limit` for good recall |
| `limit` | Number of results returned | Your top-K |
| `filter` | Pre-filter expression | Uses indexed filter fields |

**numCandidates tuning**: higher values improve recall but increase latency. The ratio `numCandidates / limit` controls the quality-speed tradeoff.

| numCandidates / limit | Recall@10 (approx) | Relative Latency |
|---|---|---|
| 5x | 0.90-0.93 | 1.0x |
| 10x | 0.95-0.97 | 1.3x |
| 15x | 0.97-0.99 | 1.6x |
| 20x | 0.98-0.99 | 2.0x |
| 50x | 0.99+ | 3.5x |

### Filtered Vector Search

Filters are applied **before** the ANN search (pre-filtering) when the filter fields are declared in the index definition. This is critical for multi-tenant applications.

```python
results = collection.aggregate([
    {
        "$vectorSearch": {
            "index": "vector_index",
            "path": "embedding",
            "queryVector": query_embedding,
            "numCandidates": 200,
            "limit": 10,
            "filter": {
                "$and": [
                    {"metadata.tenant_id": {"$eq": "tenant_123"}},
                    {"metadata.category": {"$in": ["engineering", "science"]}},
                    {"created_at": {"$gte": "2024-01-01T00:00:00Z"}}
                ]
            }
        }
    },
    {
        "$project": {
            "content": 1,
            "score": {"$meta": "vectorSearchScore"}
        }
    }
])
```

**Filter operators available in $vectorSearch**:

| Operator | Example |
|---|---|
| `$eq` | `{"field": {"$eq": "value"}}` |
| `$ne` | `{"field": {"$ne": "value"}}` |
| `$gt`, `$gte`, `$lt`, `$lte` | Range queries on dates, numbers |
| `$in` | `{"field": {"$in": ["a", "b"]}}` |
| `$nin` | `{"field": {"$nin": ["x"]}}` |
| `$and`, `$or` | Compound filters |

---

## Hybrid Search with $rankFusion

Atlas Vector Search supports hybrid retrieval by combining `$vectorSearch` with full-text `$search` using the `$rankFusion` stage (available in Atlas 7.0.4+).

### Reciprocal Rank Fusion (RRF)

```python
results = collection.aggregate([
    {
        "$rankFusion": {
            "input": {
                "pipelines": {
                    "vector": [
                        {
                            "$vectorSearch": {
                                "index": "vector_index",
                                "path": "embedding",
                                "queryVector": query_embedding,
                                "numCandidates": 150,
                                "limit": 20
                            }
                        }
                    ],
                    "fulltext": [
                        {
                            "$search": {
                                "index": "text_index",
                                "text": {
                                    "query": "machine learning transformers",
                                    "path": "content"
                                }
                            }
                        },
                        {"$limit": 20}
                    ]
                }
            },
            "combination": "rrf",
            "limit": 10
        }
    },
    {
        "$project": {
            "content": 1,
            "score": {"$meta": "score"}
        }
    }
])
```

### How RRF Works

Reciprocal Rank Fusion combines ranked lists by assigning each document a score based on its rank position in each list:

```
RRF_score(doc) = sum( 1 / (k + rank_i) ) for each pipeline i where doc appears
```

Where `k` is a constant (default 60). Documents appearing in multiple pipelines get boosted. This is rank-based, not score-based, making it robust to different score distributions between vector and text search.

### Weighted Combination

```python
# Give more weight to vector search (0.7) vs full-text (0.3)
results = collection.aggregate([
    {
        "$rankFusion": {
            "input": {
                "pipelines": {
                    "vector": [
                        {
                            "$vectorSearch": {
                                "index": "vector_index",
                                "path": "embedding",
                                "queryVector": query_embedding,
                                "numCandidates": 150,
                                "limit": 30
                            }
                        }
                    ],
                    "fulltext": [
                        {
                            "$search": {
                                "index": "text_index",
                                "text": {
                                    "query": user_query,
                                    "path": "content"
                                }
                            }
                        },
                        {"$limit": 30}
                    ]
                },
                "weights": {
                    "vector": 0.7,
                    "fulltext": 0.3
                }
            },
            "combination": "rrf",
            "limit": 10
        }
    }
])
```

---

## Dynamic Schema Benefits

MongoDB's flexible schema is a significant advantage for vector search applications:

### Mixed Embedding Models

```python
# Store embeddings from different models in the same collection
doc_openai = {
    "content": "Machine learning overview",
    "embedding_openai": [...],       # 1536 dims
    "embedding_cohere": [...],       # 1024 dims
    "embedding_model": "text-embedding-3-small",
    "metadata": {"source": "wiki"}
}

doc_cohere = {
    "content": "Deep learning tutorial",
    "embedding_openai": [...],
    "embedding_cohere": [...],
    "embedding_model": "embed-english-v3.0",
    "metadata": {"source": "tutorial"}
}

# Create separate vector indexes for each embedding field
# Index 1: on "embedding_openai" with numDimensions=1536
# Index 2: on "embedding_cohere" with numDimensions=1024
```

### Evolving Metadata Without Migrations

```python
# Add new fields to documents without schema migration
collection.update_one(
    {"_id": doc_id},
    {"$set": {
        "metadata.sentiment": 0.85,
        "metadata.language": "en",
        "metadata.token_count": 342,
        "last_embedded_at": datetime.utcnow()
    }}
)

# New filter fields can be added to the vector search index
# without reindexing existing documents
```

### Nested Document Embeddings

```python
# Embed chunks within parent documents
parent_doc = {
    "title": "Research Paper on Transformers",
    "authors": ["Author A", "Author B"],
    "chunks": [
        {
            "text": "The transformer architecture...",
            "embedding": [...],  # 1536 dims
            "chunk_index": 0
        },
        {
            "text": "Self-attention allows...",
            "embedding": [...],
            "chunk_index": 1
        }
    ],
    "metadata": {
        "source": "arxiv",
        "year": 2024
    }
}
```

---

## Auto-Embedding Pipeline with Atlas Triggers

Atlas Triggers (backed by Atlas App Services) can automate the embedding generation process without external infrastructure.

```javascript
// Atlas Trigger Function: auto-embed on insert/update
exports = async function(changeEvent) {
    const doc = changeEvent.fullDocument;
    const collection = context.services.get("mongodb-atlas")
        .db("mydb").collection("documents");

    // Skip if already has embedding
    if (doc.embedding && doc.embedding.length > 0) {
        return;
    }

    const text = doc.content || doc.title || "";
    if (!text) return;

    // Call OpenAI via HTTPS
    const response = await context.http.post({
        url: "https://api.openai.com/v1/embeddings",
        headers: {
            "Authorization": [`Bearer ${context.values.get("OPENAI_API_KEY")}`],
            "Content-Type": ["application/json"]
        },
        body: JSON.stringify({
            model: "text-embedding-3-small",
            input: text
        })
    });

    const body = JSON.parse(response.body.text());
    const embedding = body.data[0].embedding;

    await collection.updateOne(
        { _id: doc._id },
        { $set: {
            embedding: embedding,
            embedded_at: new Date(),
            embedding_model: "text-embedding-3-small"
        }}
    );
};
```

---

## Query Patterns

### Semantic Search with Score Threshold

```python
def semantic_search(collection, query_embedding, threshold=0.7, limit=10):
    """Search with a minimum similarity score threshold."""
    pipeline = [
        {
            "$vectorSearch": {
                "index": "vector_index",
                "path": "embedding",
                "queryVector": query_embedding,
                "numCandidates": limit * 15,
                "limit": limit * 3  # Over-fetch to filter by score
            }
        },
        {
            "$addFields": {
                "score": {"$meta": "vectorSearchScore"}
            }
        },
        {
            "$match": {
                "score": {"$gte": threshold}
            }
        },
        {"$limit": limit},
        {
            "$project": {
                "content": 1,
                "metadata": 1,
                "score": 1
            }
        }
    ]
    return list(collection.aggregate(pipeline))
```

### Multi-Tenant RAG

```python
def tenant_search(collection, tenant_id, query_embedding, limit=10):
    """Filtered vector search scoped to a single tenant."""
    pipeline = [
        {
            "$vectorSearch": {
                "index": "vector_index",
                "path": "embedding",
                "queryVector": query_embedding,
                "numCandidates": 200,
                "limit": limit,
                "filter": {
                    "metadata.tenant_id": {"$eq": tenant_id}
                }
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
```

### Aggregation Pipeline After Vector Search

A key strength of Atlas Vector Search is the ability to chain standard aggregation stages after the vector search.

```python
# Vector search + group by category + compute stats
pipeline = [
    {
        "$vectorSearch": {
            "index": "vector_index",
            "path": "embedding",
            "queryVector": query_embedding,
            "numCandidates": 500,
            "limit": 100
        }
    },
    {
        "$addFields": {
            "score": {"$meta": "vectorSearchScore"}
        }
    },
    {
        "$group": {
            "_id": "$metadata.category",
            "count": {"$sum": 1},
            "avg_score": {"$avg": "$score"},
            "top_doc": {"$first": "$content"}
        }
    },
    {"$sort": {"avg_score": -1}}
]
```

---

## Async Client Patterns (Motor)

### Motor Async Client

```python
import asyncio
from motor.motor_asyncio import AsyncIOMotorClient
from openai import AsyncOpenAI

motor_client = AsyncIOMotorClient("mongodb+srv://cluster0.example.mongodb.net/mydb")
openai_client = AsyncOpenAI()

db = motor_client["mydb"]
collection = db["documents"]


async def async_vector_search(query: str, limit: int = 10):
    """Async vector search with Motor."""
    # Generate embedding asynchronously
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
                "score": {"$meta": "vectorSearchScore"}
            }
        }
    ]

    results = []
    async for doc in collection.aggregate(pipeline):
        results.append(doc)
    return results


async def batch_search(queries: list[str], limit: int = 10):
    """Run multiple vector searches concurrently."""
    tasks = [async_vector_search(q, limit) for q in queries]
    return await asyncio.gather(*tasks)
```

### Async Bulk Embedding Pipeline

```python
import asyncio
from motor.motor_asyncio import AsyncIOMotorClient
from openai import AsyncOpenAI

motor_client = AsyncIOMotorClient("mongodb+srv://cluster0.example.mongodb.net/mydb")
openai_client = AsyncOpenAI()
db = motor_client["mydb"]
collection = db["documents"]


async def embed_batch(documents: list[dict], batch_size: int = 100):
    """Embed documents in batches using async OpenAI client."""
    for i in range(0, len(documents), batch_size):
        batch = documents[i:i + batch_size]
        texts = [doc["content"] for doc in batch]

        response = await openai_client.embeddings.create(
            model="text-embedding-3-small",
            input=texts
        )

        operations = []
        for doc, emb_data in zip(batch, response.data):
            operations.append({
                "filter": {"_id": doc["_id"]},
                "update": {"$set": {"embedding": emb_data.embedding}}
            })

        # Bulk update
        if operations:
            bulk_ops = [
                pymongo.UpdateOne(op["filter"], op["update"])
                for op in operations
            ]
            await collection.bulk_write(bulk_ops)


async def embed_unembedded_documents():
    """Find and embed all documents without embeddings."""
    cursor = collection.find(
        {"embedding": {"$exists": False}},
        {"_id": 1, "content": 1}
    )
    documents = await cursor.to_list(length=None)
    print(f"Found {len(documents)} documents without embeddings")
    await embed_batch(documents)
```

---

## Common Pitfalls

1. **Not declaring filter fields in the index definition**: if a field is not declared as a `filter` type in the vector search index, using it in the `$vectorSearch` filter clause will fail silently or produce incorrect results.

2. **Setting numCandidates too low**: `numCandidates` of 100 with `limit` of 50 gives a 2x ratio, which yields poor recall (~0.85). Use at least 10-15x for production workloads.

3. **Expecting immediate consistency**: Atlas Vector Search indexing is eventually consistent. Newly inserted documents may not appear in search results for several hundred milliseconds to a few seconds. Do not use it for real-time "insert-then-search" patterns without accounting for this lag.

4. **Using $search instead of $vectorSearch**: `$search` is for Atlas Search (full-text). `$vectorSearch` is for vector similarity. They are separate aggregation stages with different syntax.

5. **Ignoring the score metric**: `$meta: "vectorSearchScore"` returns a normalized score between 0 and 1 for cosine similarity, but the scale differs for euclidean and dotProduct. Always verify the score range for your chosen similarity metric.

6. **Not using quantization for large datasets**: without scalar quantization, each 1536-dim float32 vector uses ~6 KB. At 10M documents, that is ~60 GB of vector data alone. Scalar quantization reduces this to ~15 GB with minimal recall loss.

7. **Forgetting to create the Atlas Search text index for hybrid search**: `$rankFusion` requires both a vector search index and a separate Atlas Search index. The text index must be created independently.

---

## References

- MongoDB Atlas Vector Search documentation: https://www.mongodb.com/docs/atlas/atlas-vector-search/
- $vectorSearch aggregation stage: https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-stage/
- $rankFusion stage: https://www.mongodb.com/docs/atlas/atlas-vector-search/tutorials/reciprocal-rank-fusion/
- Atlas Triggers: https://www.mongodb.com/docs/atlas/app-services/triggers/
- Motor async driver: https://motor.readthedocs.io/
- pymongo documentation: https://pymongo.readthedocs.io/
