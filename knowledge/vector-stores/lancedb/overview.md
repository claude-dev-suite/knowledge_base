# LanceDB Deep Guide

## Overview

LanceDB is an open-source vector database built on the Lance columnar format, designed for serverless and embedded use cases. It is Arrow-native, supports versioning and time travel, merge-on-read updates, full-text + vector hybrid search, and operates in both embedded mode (zero server, zero config) and cloud mode (S3/GCS-backed). The Rust performance core provides fast ingestion and search while the Python API integrates seamlessly with pandas and polars. This guide covers LanceDB's architecture, features, and production patterns.

For basic quickstart, see the LanceDB documentation at <https://lancedb.github.io/lancedb/>.

---

## Architecture

### Lance Format

LanceDB is built on the Lance columnar data format, designed specifically for ML/AI workloads.

```
Lance File Layout:
  +-- Metadata (schema, version, stats)
  +-- Column Groups
  |     +-- Scalar columns (Arrow format)
  |     +-- Vector columns (contiguous float arrays)
  |     +-- Blob columns (images, text)
  +-- Index Segments
        +-- IVF_PQ index
        +-- Full-text index (tantivy)
        +-- Scalar indexes
```

**Key properties of the Lance format**:
- Columnar: reads only the columns needed for a query
- Versioned: every write creates a new version (append-only)
- Zero-copy interop with Apache Arrow
- Supports random access (unlike Parquet, which is scan-only)
- Optimized for vector data (contiguous memory layout)

### Architecture Modes

**Embedded mode** (default):
```
Python Process
  +-- LanceDB library (Rust core via PyO3)
  +-- Lance dataset (local filesystem / S3 / GCS)
  
No server process. No network calls. Just a library.
```

**Cloud mode** (LanceDB Cloud):
```
Client --> [LanceDB Cloud API] --> [Object Storage (S3/GCS)]
                                        |
                                   [Lance files]
```

---

## Embedded Mode (Zero Config)

### Installation

```bash
pip install lancedb
```

### Creating a Table

```python
import lancedb
import numpy as np
import pyarrow as pa

# Open a database (creates directory if not exists)
db = lancedb.connect("./my_lancedb")

# Create table from list of dicts
data = [
    {
        "id": i,
        "title": f"Document {i}",
        "category": f"cat-{i % 10}",
        "year": 2020 + (i % 6),
        "vector": np.random.rand(1536).astype(np.float32).tolist(),
    }
    for i in range(1000)
]

table = db.create_table("documents", data=data, mode="overwrite")
print(f"Created table with {table.count_rows()} rows")
```

### Creating from Arrow Table

```python
import pyarrow as pa
import numpy as np

# Create from PyArrow table (zero-copy when possible)
num_rows = 100_000
schema = pa.schema([
    pa.field("id", pa.int64()),
    pa.field("title", pa.string()),
    pa.field("category", pa.string()),
    pa.field("vector", pa.list_(pa.float32(), 1536)),
])

arrays = [
    pa.array(range(num_rows)),
    pa.array([f"Document {i}" for i in range(num_rows)]),
    pa.array([f"cat-{i % 10}" for i in range(num_rows)]),
    pa.array(
        [np.random.rand(1536).astype(np.float32).tolist() for _ in range(num_rows)],
        type=pa.list_(pa.float32(), 1536),
    ),
]

arrow_table = pa.table(arrays, schema=schema)
table = db.create_table("documents_arrow", data=arrow_table)
```

### Creating from Pandas/Polars

```python
import pandas as pd

# From pandas DataFrame
df = pd.DataFrame({
    "id": range(1000),
    "title": [f"Doc {i}" for i in range(1000)],
    "vector": [np.random.rand(1536).tolist() for _ in range(1000)],
})

table = db.create_table("from_pandas", data=df)

# From polars DataFrame
import polars as pl

pldf = pl.DataFrame({
    "id": range(1000),
    "title": [f"Doc {i}" for i in range(1000)],
    "vector": [np.random.rand(1536).tolist() for _ in range(1000)],
})

table = db.create_table("from_polars", data=pldf.to_arrow())
```

---

## Versioning and Time Travel

Every write operation creates a new version of the dataset. Previous versions remain accessible for time travel queries.

### Version Management

```python
# Check current version
print(f"Current version: {table.version}")

# Add new data (creates a new version)
new_data = [
    {"id": 1001, "title": "New document", "vector": np.random.rand(1536).tolist()},
]
table.add(new_data)
print(f"After add: version {table.version}")

# List all versions
versions = table.list_versions()
for v in versions[:5]:
    print(f"Version {v['version']}: created at {v['timestamp']}")
```

### Time Travel

```python
# Open a specific version
table_v1 = db.open_table("documents", version=1)
print(f"Version 1 rows: {table_v1.count_rows()}")

# Current table (latest version)
table_latest = db.open_table("documents")
print(f"Latest rows: {table_latest.count_rows()}")

# Checkout a version
table.checkout(version=1)
results = table.search(query_vector).limit(10).to_pandas()

# Return to latest
table.checkout_latest()
```

### Merge-on-Read Updates

LanceDB uses merge-on-read for updates and deletes. Instead of rewriting data in place, modifications are stored as delta files that are merged at read time.

```python
# Update rows
table.update(
    where="category = 'cat-0'",
    values={"category": "updated-category"},
)

# Delete rows
table.delete("year < 2022")

# Compact (merge deltas into base files for read performance)
table.compact_files()

# Clean up old versions
table.cleanup_old_versions(older_than=pd.Timedelta(days=7))
```

---

## Search

### Vector Search

```python
query_vector = np.random.rand(1536).astype(np.float32).tolist()

# Basic vector search
results = (
    table.search(query_vector)
    .limit(10)
    .to_pandas()
)

print(results[["id", "title", "_distance"]])
```

### Filtered Search

```python
# Vector search with SQL filter
results = (
    table.search(query_vector)
    .where("category = 'technology' AND year >= 2023")
    .limit(10)
    .to_pandas()
)
```

### Full-Text Search

LanceDB uses tantivy (Rust) for full-text search indexing.

```python
# Create full-text index
table.create_fts_index("title")

# Full-text search
results = (
    table.search("vector database optimization", query_type="fts")
    .limit(10)
    .to_pandas()
)
```

### Hybrid Search (Vector + Full-Text)

```python
# Hybrid search combines vector similarity with text relevance
results = (
    table.search(query_vector, query_type="hybrid")
    .where("year >= 2023")
    .limit(10)
    .to_pandas()
)
```

### Reranking for Hybrid Search

```python
from lancedb.rerankers import LinearCombinationReranker, CrossEncoderReranker

# Linear combination of vector + FTS scores
reranker = LinearCombinationReranker(weight=0.7)  # 70% vector, 30% FTS
results = (
    table.search(query_vector, query_type="hybrid")
    .rerank(reranker=reranker)
    .limit(10)
    .to_pandas()
)

# Cross-encoder reranking (higher quality, slower)
reranker = CrossEncoderReranker(model_name="cross-encoder/ms-marco-MiniLM-L-6-v2")
results = (
    table.search("vector database", query_type="hybrid")
    .rerank(reranker=reranker)
    .limit(10)
    .to_pandas()
)
```

---

## Cloud Mode (S3/GCS-Backed)

### S3 Backend

```python
import lancedb

# Connect to S3-backed database
db = lancedb.connect(
    "s3://my-bucket/lancedb/",
    storage_options={
        "aws_access_key_id": "your-key",
        "aws_secret_access_key": "your-secret",
        "region": "us-east-1",
    },
)

# Usage is identical to local mode
table = db.create_table("documents", data=data)
results = table.search(query_vector).limit(10).to_pandas()
```

### GCS Backend

```python
db = lancedb.connect(
    "gs://my-bucket/lancedb/",
    storage_options={
        "service_account": "/path/to/service-account.json",
    },
)
```

### Azure Blob Storage

```python
db = lancedb.connect(
    "az://my-container/lancedb/",
    storage_options={
        "account_name": "your-account",
        "account_key": "your-key",
    },
)
```

---

## LanceDB Cloud

LanceDB Cloud is the managed service with serverless compute.

```python
import lancedb

# Connect to LanceDB Cloud
db = lancedb.connect(
    "db://my-project",
    api_key="your-lancedb-cloud-api-key",
    region="us-east-1",
)

# Usage is identical
table = db.create_table("documents", data=data)
results = table.search(query_vector).limit(10).to_pandas()
```

---

## Embedding Function Registration

LanceDB can automatically embed data using registered embedding functions.

```python
from lancedb.embeddings import get_registry
from lancedb.pydantic import LanceModel, Vector

# Get an embedding function from the registry
openai_embed = get_registry().get("openai").create(name="text-embedding-3-small")

# Define a model with automatic embedding
class Document(LanceModel):
    title: str
    content: str = openai_embed.SourceField()
    vector: Vector(openai_embed.ndims()) = openai_embed.VectorField()
    category: str
    year: int

# Create table with embedding model
table = db.create_table("auto_embedded", schema=Document, mode="overwrite")

# Add data -- vectors are computed automatically
table.add([
    {
        "title": "Vector Database Guide",
        "content": "LanceDB is a serverless vector database...",
        "category": "technology",
        "year": 2025,
    },
])

# Search with text -- automatically embedded
results = table.search("how does vector search work").limit(10).to_pandas()
```

### Available Embedding Functions

| Provider | Function Name | Models |
|----------|-------------|--------|
| OpenAI | openai | text-embedding-3-small, text-embedding-3-large |
| Cohere | cohere | embed-english-v3.0, embed-multilingual-v3.0 |
| Sentence Transformers | sentence-transformers | all-MiniLM-L6-v2, etc. |
| Hugging Face | huggingface | Any HF model |
| Ollama | ollama | Local LLM embeddings |
| Instructor | instructor | instructor-large, instructor-xl |

---

## Indexing

### IVF_PQ Index

```python
# Create IVF_PQ index for approximate search
table.create_index(
    metric="cosine",
    num_partitions=256,       # IVF partitions (sqrt(N) is a good start)
    num_sub_vectors=96,       # PQ sub-quantizers
    index_cache_size=256,     # cache size in MB
)

# Search uses index automatically
results = table.search(query_vector).limit(10).nprobes(20).to_pandas()
```

### IVF_HNSW_SQ Index (LanceDB 0.6+)

```python
# Scalar-quantized HNSW within IVF partitions
table.create_index(
    metric="cosine",
    index_type="IVF_HNSW_SQ",
    num_partitions=256,
    # HNSW parameters per partition
    max_iterations=50,
)
```

### Full-Text Index

```python
# Create full-text search index
table.create_fts_index("content", use_tantivy=True)

# Create index on multiple fields
table.create_fts_index(["title", "content"], use_tantivy=True)
```

### Scalar Index

```python
# Create scalar index for faster filtering
table.create_scalar_index("category")
table.create_scalar_index("year")
```

---

## Data Ingestion Patterns

### Batch Insert

```python
import numpy as np

# Large batch insert
BATCH_SIZE = 10_000
total = 1_000_000

for i in range(0, total, BATCH_SIZE):
    end = min(i + BATCH_SIZE, total)
    batch = [
        {
            "id": j,
            "title": f"Document {j}",
            "category": f"cat-{j % 10}",
            "year": 2020 + (j % 6),
            "vector": np.random.rand(1536).astype(np.float32).tolist(),
        }
        for j in range(i, end)
    ]
    table.add(batch)

    if (i + BATCH_SIZE) % 100_000 == 0:
        print(f"Inserted {i + BATCH_SIZE:,} rows")

# Compact after bulk insert
table.compact_files()
print(f"Total rows: {table.count_rows()}")
```

### Arrow-Native Ingestion (Fastest)

```python
import pyarrow as pa
import numpy as np

# Direct Arrow ingestion (zero-copy where possible)
BATCH_SIZE = 50_000

for i in range(0, 1_000_000, BATCH_SIZE):
    end = min(i + BATCH_SIZE, 1_000_000)
    n = end - i

    batch = pa.table({
        "id": pa.array(range(i, end), type=pa.int64()),
        "title": pa.array([f"Doc {j}" for j in range(i, end)]),
        "category": pa.array([f"cat-{j % 10}" for j in range(i, end)]),
        "vector": pa.array(
            np.random.rand(n, 1536).astype(np.float32).tolist(),
            type=pa.list_(pa.float32(), 1536),
        ),
    })

    table.add(batch)
```

### Incremental Updates

```python
# LanceDB supports merge_insert for upsert-like behavior
table.merge_insert("id") \
    .when_matched_update_all() \
    .when_not_matched_insert_all() \
    .execute(new_data)
```

---

## Table Management

```python
# List tables
tables = db.table_names()
print(f"Tables: {tables}")

# Open existing table
table = db.open_table("documents")

# Get schema
print(table.schema)

# Count rows
print(f"Rows: {table.count_rows()}")

# Drop table
db.drop_table("old_table")

# Compact files (optimize read performance)
table.compact_files()

# Clean up old versions
table.cleanup_old_versions(older_than=pd.Timedelta(days=7))
```

---

## Common Pitfalls

1. **Not compacting after large inserts**: each `add()` creates new Lance fragments. Many small fragments degrade read performance. Call `compact_files()` after bulk inserts.

2. **Expecting server-based deployment in embedded mode**: LanceDB embedded is a library, not a server. For multi-process access, use cloud mode (S3/GCS) or LanceDB Cloud.

3. **Not creating an index for large tables**: without an index, LanceDB falls back to brute-force scan. For > 50K vectors, always create an IVF_PQ or IVF_HNSW_SQ index.

4. **Ignoring versioning storage growth**: every write creates a new version. Without `cleanup_old_versions()`, storage grows unboundedly.

5. **Using Python lists instead of Arrow for large inserts**: Python dict/list ingestion involves serialization overhead. Use PyArrow tables for 3-5x faster ingestion.

6. **Not setting `nprobes` for indexed search**: the default nprobes may be too low for high recall. Increase to 10-20% of `num_partitions` for production queries.

7. **Assuming LanceDB Cloud and embedded have identical performance**: cloud mode adds S3/GCS latency for reads. Embedded mode on local NVMe is significantly faster for latency-sensitive workloads.

8. **Forgetting to create scalar indexes for filtered queries**: without scalar indexes, filter conditions trigger full column scans. Create indexes on columns used in `where()` clauses.

---

## References

- LanceDB documentation: https://lancedb.github.io/lancedb/
- Lance format specification: https://github.com/lancedb/lance
- LanceDB Python API: https://lancedb.github.io/lancedb/python/
- LanceDB Cloud: https://cloud.lancedb.com/
- LanceDB embedding functions: https://lancedb.github.io/lancedb/embeddings/
- Arrow integration: https://arrow.apache.org/docs/python/
