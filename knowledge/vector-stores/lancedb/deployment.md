# LanceDB Deployment Guide

## Overview

This guide covers deploying LanceDB in production: pip install for embedded mode, cloud setup with S3 backend, LanceDB Cloud, pandas/polars integration, embedding function registration, indexing strategies, table management, and data ingestion patterns. LanceDB's unique embedded-first architecture means "deployment" ranges from a simple pip install to a fully managed cloud service.

---

## Embedded Mode (Zero Config)

### Installation

```bash
# Basic installation
pip install lancedb

# With optional dependencies
pip install lancedb[embeddings]      # embedding function support
pip install lancedb[full]            # all optional deps
```

### Local Filesystem

```python
import lancedb

# Connect to a local directory
db = lancedb.connect("./production_db")

# The directory structure:
# ./production_db/
#   +-- table_name.lance/
#   |     +-- _versions/         (version metadata)
#   |     +-- _indices/          (vector and FTS indexes)
#   |     +-- data/              (lance data files)
#   +-- another_table.lance/
```

### Production Configuration

```python
import lancedb

# Embedded mode with custom settings
db = lancedb.connect(
    "./production_db",
    read_consistency_interval=0,  # always read latest (0 = no cache)
)

# For read-heavy workloads, enable read caching
db = lancedb.connect(
    "./production_db",
    read_consistency_interval=5,  # cache reads for 5 seconds
)
```

### Multi-Process Access

Embedded mode supports concurrent reads from multiple processes but writes should be serialized.

```python
# Reader process (safe to have many)
db = lancedb.connect("./production_db")
table = db.open_table("documents")
results = table.search(query_vector).limit(10).to_pandas()

# Writer process (should be single writer)
db = lancedb.connect("./production_db")
table = db.open_table("documents")
table.add(new_data)
```

**Concurrency rules**:
- Multiple readers: safe
- Single writer + multiple readers: safe (readers see previous version until writer commits)
- Multiple writers: NOT safe (use external locking or cloud mode)

---

## Cloud Setup (S3 Backend)

### AWS S3

```python
import lancedb

# Using environment variables (recommended)
# AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_DEFAULT_REGION
db = lancedb.connect("s3://my-bucket/lancedb-prod/")

# Using explicit credentials
db = lancedb.connect(
    "s3://my-bucket/lancedb-prod/",
    storage_options={
        "aws_access_key_id": "AKIA...",
        "aws_secret_access_key": "...",
        "region": "us-east-1",
    },
)

# Using IAM role (EC2, ECS, Lambda)
db = lancedb.connect(
    "s3://my-bucket/lancedb-prod/",
    storage_options={"region": "us-east-1"},
)
```

### S3 Performance Optimization

```python
# S3 storage options for performance
db = lancedb.connect(
    "s3://my-bucket/lancedb-prod/",
    storage_options={
        "region": "us-east-1",
        "aws_s3_allow_unsafe_rename": "true",    # faster renames
        "connect_timeout": "5s",
        "timeout": "30s",
    },
)
```

### GCS Backend

```python
# Google Cloud Storage
db = lancedb.connect(
    "gs://my-bucket/lancedb-prod/",
    storage_options={
        "service_account": "/path/to/sa-key.json",
    },
)

# Using default credentials (GCE, Cloud Run)
db = lancedb.connect("gs://my-bucket/lancedb-prod/")
```

### Azure Blob Storage

```python
db = lancedb.connect(
    "az://my-container/lancedb-prod/",
    storage_options={
        "account_name": "myaccount",
        "account_key": "...",
    },
)
```

### S3 Cost Estimation

| Operation | S3 Cost |
|-----------|---------|
| PUT (write Lance fragment) | $0.005 per 1,000 |
| GET (read Lance fragment) | $0.0004 per 1,000 |
| Storage | $0.023 per GB/month |

**Example**: 1M vectors, 1536 dims, 10K queries/day
- Storage: ~6 GB * $0.023 = $0.14/month
- Reads: ~10K * 30 * 3 GETs = 900K GETs = $0.36/month
- Total: ~$0.50/month (dramatically cheaper than dedicated vector DB services)

---

## LanceDB Cloud

LanceDB Cloud provides a managed service with serverless compute.

```python
import lancedb

# Connect to LanceDB Cloud
db = lancedb.connect(
    "db://my-project",
    api_key="ldb_...",
    region="us-east-1",
)

# Create table
table = db.create_table("documents", data=data)

# Search
results = table.search(query_vector).limit(10).to_pandas()
```

**LanceDB Cloud tiers** (approximate as of 2025):

| Tier | Storage | Queries/Month | Price/Month |
|------|---------|--------------|-------------|
| Free | 5 GB | 1M | $0 |
| Starter | 50 GB | 10M | ~$29 |
| Pro | 500 GB | 100M | ~$199 |
| Enterprise | Custom | Custom | Contact sales |

---

## Pandas and Polars Integration

### Pandas

```python
import pandas as pd
import lancedb

db = lancedb.connect("./my_db")
table = db.open_table("documents")

# Search results as DataFrame
results_df = table.search(query_vector).limit(10).to_pandas()
print(results_df[["id", "title", "_distance"]])

# Full table as DataFrame
full_df = table.to_pandas()

# Filter as DataFrame
filtered_df = (
    table.search(query_vector)
    .where("category = 'technology' AND year >= 2023")
    .limit(100)
    .to_pandas()
)
```

### Polars

```python
import polars as pl

# Search results as Polars DataFrame
results_pl = table.search(query_vector).limit(10).to_polars()
print(results_pl.select(["id", "title", "_distance"]))

# Full table as Polars LazyFrame
lazy = table.to_polars()
filtered = lazy.filter(pl.col("category") == "technology").collect()

# Ingest from Polars
pldf = pl.DataFrame({
    "id": range(1000),
    "title": [f"Doc {i}" for i in range(1000)],
    "vector": [np.random.rand(1536).tolist() for _ in range(1000)],
})
table.add(pldf.to_arrow())
```

---

## Embedding Function Registration

### Built-in Embedding Functions

```python
from lancedb.embeddings import get_registry

# List available embedding functions
registry = get_registry()
print(registry.list())

# OpenAI
openai_embed = registry.get("openai").create(
    name="text-embedding-3-small",
    # api_key is read from OPENAI_API_KEY env var
)

# Sentence Transformers (local)
st_embed = registry.get("sentence-transformers").create(
    name="all-MiniLM-L6-v2",
    device="cpu",
)

# Cohere
cohere_embed = registry.get("cohere").create(
    name="embed-english-v3.0",
)

# Ollama (local LLM)
ollama_embed = registry.get("ollama").create(
    name="nomic-embed-text",
    host="http://localhost:11434",
)
```

### Pydantic Model with Auto-Embedding

```python
from lancedb.pydantic import LanceModel, Vector
from lancedb.embeddings import get_registry

embed_fn = get_registry().get("openai").create(name="text-embedding-3-small")

class Document(LanceModel):
    title: str
    content: str = embed_fn.SourceField()
    vector: Vector(embed_fn.ndims()) = embed_fn.VectorField()
    category: str
    year: int

# Create table with schema
table = db.create_table("auto_docs", schema=Document, mode="overwrite")

# Insert -- vectors computed automatically
table.add([
    Document(
        title="Vector Search Guide",
        content="LanceDB provides serverless vector search...",
        category="technology",
        year=2025,
    ),
])

# Search with text -- auto-embedded
results = table.search("how does vector search work").limit(10).to_pandas()
```

### Custom Embedding Function

```python
from lancedb.embeddings import EmbeddingFunction, register
import numpy as np

@register("my-custom-embedder")
class MyEmbedder(EmbeddingFunction):
    name: str = "custom"

    def ndims(self) -> int:
        return 384

    def compute_query_embeddings(self, query: str, *args, **kwargs):
        # Your embedding logic for queries
        return [np.random.rand(384).tolist()]

    def compute_source_embeddings(self, texts: list[str], *args, **kwargs):
        # Your embedding logic for documents
        return [np.random.rand(384).tolist() for _ in texts]
```

---

## Indexing Strategies

### IVF_PQ (Default, Recommended for > 50K Vectors)

```python
# Create IVF_PQ index
table.create_index(
    metric="cosine",                 # cosine, L2, dot
    num_partitions=256,              # IVF partitions
    num_sub_vectors=96,              # PQ sub-quantizers
    index_cache_size=256,            # MB of index cache
)

# Search with index
results = (
    table.search(query_vector)
    .nprobes(20)                     # search 20 of 256 partitions
    .refine_factor(10)               # fetch 10x candidates, rescore
    .limit(10)
    .to_pandas()
)
```

### IVF_HNSW_SQ (Higher Recall)

```python
# Scalar-quantized HNSW within IVF partitions
table.create_index(
    metric="cosine",
    index_type="IVF_HNSW_SQ",
    num_partitions=256,
)
```

### Index Tuning Guide

| Vectors | num_partitions | num_sub_vectors | nprobes | Expected Recall |
|---------|---------------|----------------|---------|----------------|
| 50K | 64 | 48 | 10 | 0.95 |
| 100K | 128 | 64 | 15 | 0.95 |
| 500K | 256 | 96 | 20 | 0.95 |
| 1M | 512 | 96 | 30 | 0.95 |
| 5M | 1024 | 128 | 50 | 0.95 |
| 10M | 2048 | 128 | 80 | 0.95 |

**Rules of thumb**:
- `num_partitions` = sqrt(num_vectors) * 2
- `num_sub_vectors` = dimensions / 16 (minimum 16)
- `nprobes` = num_partitions * 0.05-0.10 for ~95% recall

### Full-Text Index

```python
# Create tantivy-based full-text index
table.create_fts_index("content")

# Multi-field full-text index
table.create_fts_index(["title", "content"])
```

### Scalar Index

```python
# Create scalar indexes for filtered queries
table.create_scalar_index("category")
table.create_scalar_index("year")
```

---

## Table Management

### Compaction

```python
# Compact small files into larger ones (improves read performance)
stats = table.compact_files()
print(f"Compacted: {stats}")

# Recommendation: compact after every 10-50 add() calls
# or when the number of fragments exceeds 100
```

### Version Cleanup

```python
import datetime

# Clean up versions older than 7 days
table.cleanup_old_versions(
    older_than=datetime.timedelta(days=7),
    delete_unverified=False,
)

# Check version count
versions = table.list_versions()
print(f"Active versions: {len(versions)}")
```

### Optimization Script

```python
def optimize_table(table, compact_threshold=100, version_retention_days=7):
    """Run periodic optimization on a LanceDB table."""
    import datetime

    # Check fragment count
    stats = table.stats()
    num_fragments = stats.get("num_data_files", 0)

    if num_fragments > compact_threshold:
        print(f"Compacting {num_fragments} fragments...")
        table.compact_files()

    # Clean up old versions
    table.cleanup_old_versions(
        older_than=datetime.timedelta(days=version_retention_days),
    )

    print(f"Optimization complete. Rows: {table.count_rows()}, Version: {table.version}")
```

---

## Data Ingestion Patterns

### Streaming Ingestion

```python
import time

def stream_ingest(table, data_stream, batch_size=1000, compact_every=50):
    """Ingest data from a stream in batches."""
    batch = []
    add_count = 0

    for item in data_stream:
        batch.append(item)

        if len(batch) >= batch_size:
            table.add(batch)
            batch = []
            add_count += 1

            if add_count % compact_every == 0:
                table.compact_files()
                print(f"Compacted after {add_count} batches")

    # Flush remaining
    if batch:
        table.add(batch)

    # Final compaction
    table.compact_files()
```

### ETL Pipeline

```python
import pyarrow.parquet as pq
import lancedb

def etl_parquet_to_lancedb(
    parquet_path: str,
    db_path: str,
    table_name: str,
    vector_column: str = "embedding",
):
    """Load Parquet file into LanceDB with indexing."""
    db = lancedb.connect(db_path)

    # Read Parquet
    arrow_table = pq.read_table(parquet_path)
    print(f"Loaded {arrow_table.num_rows} rows from {parquet_path}")

    # Create or overwrite table
    table = db.create_table(table_name, data=arrow_table, mode="overwrite")
    print(f"Created table '{table_name}' with {table.count_rows()} rows")

    # Create vector index
    dims = len(arrow_table.column(vector_column)[0].as_py())
    num_rows = arrow_table.num_rows
    num_partitions = max(16, int(num_rows ** 0.5) * 2)

    table.create_index(
        metric="cosine",
        num_partitions=num_partitions,
        num_sub_vectors=max(16, dims // 16),
    )
    print(f"Created IVF_PQ index with {num_partitions} partitions")

    # Compact
    table.compact_files()
    print("Compaction complete")

    return table
```

---

## Deployment on AWS Lambda

LanceDB's embedded mode works in serverless environments.

```python
# Lambda handler
import lancedb
import json

def handler(event, context):
    # Connect to S3-backed LanceDB
    db = lancedb.connect("s3://my-bucket/lancedb/")
    table = db.open_table("documents")

    # Parse query
    query_vector = event.get("vector", [])
    top_k = event.get("top_k", 10)

    # Search
    results = (
        table.search(query_vector)
        .limit(top_k)
        .to_pandas()
    )

    return {
        "statusCode": 200,
        "body": json.dumps({
            "results": [
                {"id": str(row["id"]), "distance": float(row["_distance"])}
                for _, row in results.iterrows()
            ]
        }),
    }
```

### Lambda Layer

```bash
# Create a Lambda layer with LanceDB
mkdir -p layer/python
pip install lancedb -t layer/python/
cd layer && zip -r ../lancedb-layer.zip python/
```

---

## Common Pitfalls

1. **Not compacting after bulk inserts**: each `add()` creates new data fragments. Without compaction, read performance degrades as the number of fragments grows.

2. **Forgetting to clean up old versions**: versioning is append-only. Without cleanup, storage grows unboundedly, especially on S3 where storage costs are per-GB.

3. **Running multiple writers in embedded mode**: embedded mode does not support concurrent writers. Use external locking or switch to cloud mode for multi-writer scenarios.

4. **Not creating an index**: without an index, all searches are brute-force. For > 50K vectors, always create an IVF_PQ index.

5. **Using Python lists for large ingestion**: Python dict/list serialization is slow. Use PyArrow tables for 3-5x faster ingestion.

6. **Ignoring S3 latency for real-time queries**: S3-backed databases add 10-50ms per query due to object storage latency. For sub-10ms requirements, use local filesystem or LanceDB Cloud.

7. **Not setting nprobes for indexed search**: the default nprobes is often too low. Set to 5-10% of num_partitions for ~95% recall.

---

## References

- LanceDB documentation: https://lancedb.github.io/lancedb/
- LanceDB Cloud: https://cloud.lancedb.com/
- Lance format: https://github.com/lancedb/lance
- LanceDB Python API: https://lancedb.github.io/lancedb/python/
- LanceDB embedding functions: https://lancedb.github.io/lancedb/embeddings/
- AWS Lambda with LanceDB: https://blog.lancedb.com/serverless-vector-search-with-lancedb/
