# Vector Quantization -- Implementation Guide

## Overview

This guide provides practical implementation code for vector quantization across major vector databases and libraries: Faiss IndexIVFPQ with rescoring, Qdrant quantization configuration (scalar, binary, product), Weaviate PQ/BQ/SQ configuration, pgvector halfvec for 2x savings, and guidance on selecting the right technique for your workload. All code uses current 2024/2025 APIs.

---

## Faiss: IndexIVFPQ with Rescoring

### Basic IVFPQ

```python
import faiss
import numpy as np
import time

d = 1536              # Dimensions
nlist = 4096          # Number of IVF partitions
m = 48                # PQ subquantizers (must divide d)
n_bits = 8            # Bits per code (256 centroids per subspace)

# Generate sample data (replace with real embeddings)
n = 1_000_000
vectors = np.random.rand(n, d).astype(np.float32)
queries = np.random.rand(100, d).astype(np.float32)

# Build IVFPQ index
quantizer = faiss.IndexFlatL2(d)
index = faiss.IndexIVFPQ(quantizer, d, nlist, m, n_bits)

# Train on representative subset
print("Training...")
train_size = min(200_000, n)
start = time.perf_counter()
index.train(vectors[:train_size])
print(f"Training time: {time.perf_counter() - start:.1f}s")

# Add all vectors
print("Adding vectors...")
start = time.perf_counter()
index.add(vectors)
print(f"Add time: {time.perf_counter() - start:.1f}s")

# Search
index.nprobe = 32
distances, indices = index.search(queries, k=10)

# Measure recall (compare with brute force)
flat_index = faiss.IndexFlatL2(d)
flat_index.add(vectors)
_, gt_indices = flat_index.search(queries, k=10)

recall = np.mean([
    len(set(indices[i]) & set(gt_indices[i])) / 10
    for i in range(len(queries))
])
print(f"Recall@10: {recall:.4f}")
```

### IVFPQ with IndexRefineFlat (Automatic Rescoring)

```python
import faiss
import numpy as np

d = 1536
nlist = 4096
m = 48

# Build base IVFPQ index
quantizer = faiss.IndexFlatL2(d)
base_index = faiss.IndexIVFPQ(quantizer, d, nlist, m, 8)
base_index.train(vectors[:200_000])
base_index.add(vectors)

# Wrap with refinement index
# This stores full-precision vectors alongside PQ codes
# and automatically rescores PQ candidates with exact distances
refine_index = faiss.IndexRefineFlat(base_index)
refine_index.k_factor = 10  # Retrieve 10x candidates, then rescore

# Search (automatically does PQ search + exact rescore)
base_index.nprobe = 32
distances, indices = refine_index.search(queries, k=10)

print(f"Memory: {refine_index.sa_code_size()} bytes per vector in base")
```

### OPQ + IVFPQ (Optimized Product Quantization)

```python
import faiss
import numpy as np

d = 1536
m = 48
nlist = 4096

# OPQ pre-transform: learns a rotation matrix to minimize PQ error
opq_matrix = faiss.OPQMatrix(d, m)

# Build the composite index: OPQ rotation -> IVFPQ
quantizer = faiss.IndexFlatL2(d)
sub_index = faiss.IndexIVFPQ(quantizer, d, nlist, m, 8)
index = faiss.IndexPreTransform(opq_matrix, sub_index)

# Train (learns rotation matrix + IVF centroids + PQ codebooks)
print("Training OPQ+IVFPQ...")
index.train(vectors[:200_000])

# Add vectors
index.add(vectors)

# Search
sub_index.nprobe = 32
distances, indices = index.search(queries, k=10)

# OPQ typically improves recall by 1-3% over standard PQ
```

### Saving and Loading Faiss Indexes

```python
import faiss

# Save index to disk
faiss.write_index(index, "ivfpq_index.faiss")

# Load index from disk
loaded_index = faiss.read_index("ivfpq_index.faiss")

# For IVFPQ, set nprobe after loading
loaded_index.nprobe = 32
distances, indices = loaded_index.search(queries, k=10)
```

### Custom Two-Stage Retrieval

```python
import faiss
import numpy as np
from pathlib import Path


class TwoStageRetriever:
    """Production-ready two-stage retriever: IVFPQ + exact rescoring."""

    def __init__(self, dim: int, nlist: int = 4096, pq_m: int = 48):
        self.dim = dim
        self.nlist = nlist
        self.pq_m = pq_m
        self.index = None
        self.vectors_mmap = None
        self.n_vectors = 0

    def build(self, vectors: np.ndarray, vectors_path: str):
        """Build index and save full-precision vectors for rescoring."""
        self.n_vectors = vectors.shape[0]

        # Save full-precision vectors as memory-mapped file
        mmap = np.memmap(vectors_path, dtype=np.float32, mode='w+',
                         shape=vectors.shape)
        mmap[:] = vectors[:]
        mmap.flush()
        del mmap

        # Load as read-only mmap
        self.vectors_mmap = np.memmap(
            vectors_path, dtype=np.float32, mode='r',
            shape=vectors.shape
        )

        # Build IVFPQ index
        quantizer = faiss.IndexFlatL2(self.dim)
        self.index = faiss.IndexIVFPQ(
            quantizer, self.dim, self.nlist, self.pq_m, 8
        )

        train_size = min(200_000, self.n_vectors)
        self.index.train(vectors[:train_size])
        self.index.add(vectors)

    def search(self, query: np.ndarray, k: int = 10,
               nprobe: int = 32, rescore_factor: int = 10) -> tuple:
        """Search with IVFPQ + full-precision rescoring."""
        self.index.nprobe = nprobe
        rescore_k = min(k * rescore_factor, self.n_vectors)

        # Stage 1: Fast approximate search
        query_2d = query.reshape(1, -1).astype(np.float32)
        _, candidate_ids = self.index.search(query_2d, rescore_k)
        candidate_ids = candidate_ids[0]
        valid_mask = candidate_ids >= 0
        candidate_ids = candidate_ids[valid_mask]

        if len(candidate_ids) == 0:
            return np.array([]), np.array([])

        # Stage 2: Load full-precision vectors and compute exact distances
        candidate_vectors = self.vectors_mmap[candidate_ids]
        exact_distances = np.linalg.norm(
            candidate_vectors - query.reshape(1, -1), axis=1
        ) ** 2  # Squared L2 for consistency with Faiss

        # Stage 3: Re-rank and return top-k
        top_k_local = np.argsort(exact_distances)[:k]
        return candidate_ids[top_k_local], exact_distances[top_k_local]

    def batch_search(self, queries: np.ndarray, k: int = 10,
                     nprobe: int = 32, rescore_factor: int = 10) -> tuple:
        """Batch search for multiple queries."""
        all_ids = []
        all_dists = []
        for query in queries:
            ids, dists = self.search(query, k, nprobe, rescore_factor)
            all_ids.append(ids)
            all_dists.append(dists)
        return all_ids, all_dists

    def save(self, index_path: str):
        """Save the IVFPQ index."""
        faiss.write_index(self.index, index_path)

    def load(self, index_path: str, vectors_path: str, n_vectors: int, dim: int):
        """Load saved index and vectors."""
        self.index = faiss.read_index(index_path)
        self.dim = dim
        self.n_vectors = n_vectors
        self.vectors_mmap = np.memmap(
            vectors_path, dtype=np.float32, mode='r',
            shape=(n_vectors, dim)
        )


# Usage
retriever = TwoStageRetriever(dim=1536)
retriever.build(vectors, "vectors.mmap")
ids, dists = retriever.search(queries[0], k=10, nprobe=32, rescore_factor=10)
```

---

## Qdrant Quantization Configuration

### Scalar Quantization (int8)

```python
from qdrant_client import QdrantClient
from qdrant_client.models import (
    VectorParams, Distance, PointStruct,
    ScalarQuantization, ScalarQuantizationConfig,
    ScalarType, QuantizationSearchParams, SearchParams
)

client = QdrantClient(url="http://localhost:6333")

# Create collection with scalar quantization
client.create_collection(
    collection_name="documents_sq",
    vectors_config=VectorParams(
        size=1536,
        distance=Distance.COSINE
    ),
    quantization_config=ScalarQuantization(
        scalar=ScalarQuantizationConfig(
            type=ScalarType.INT8,
            quantile=0.99,     # Clip outliers (top/bottom 1%)
            always_ram=True    # Keep quantized vectors in RAM
        )
    )
)

# Insert vectors
points = [
    PointStruct(
        id=i,
        vector=vectors[i].tolist(),
        payload={"content": f"Document {i}", "category": "test"}
    )
    for i in range(1000)
]
client.upsert(collection_name="documents_sq", points=points)

# Search with rescoring enabled
results = client.search(
    collection_name="documents_sq",
    query_vector=query_vector.tolist(),
    limit=10,
    search_params=SearchParams(
        quantization=QuantizationSearchParams(
            rescore=True,      # Rescore with full-precision vectors
            oversampling=3.0   # Retrieve 3x candidates for rescoring
        )
    )
)

for result in results:
    print(f"ID: {result.id}, Score: {result.score:.4f}")
```

### Binary Quantization (1-bit)

```python
from qdrant_client.models import BinaryQuantization, BinaryQuantizationConfig

# Create collection with binary quantization
client.create_collection(
    collection_name="documents_bq",
    vectors_config=VectorParams(
        size=1536,
        distance=Distance.COSINE
    ),
    quantization_config=BinaryQuantization(
        binary=BinaryQuantizationConfig(
            always_ram=True
        )
    )
)

# Search with higher oversampling (BQ needs more rescoring)
results = client.search(
    collection_name="documents_bq",
    query_vector=query_vector.tolist(),
    limit=10,
    search_params=SearchParams(
        quantization=QuantizationSearchParams(
            rescore=True,
            oversampling=5.0  # Higher oversampling for BQ
        )
    )
)
```

### Product Quantization

```python
from qdrant_client.models import ProductQuantization, ProductQuantizationConfig

# Create collection with product quantization
client.create_collection(
    collection_name="documents_pq",
    vectors_config=VectorParams(
        size=1536,
        distance=Distance.COSINE
    ),
    quantization_config=ProductQuantization(
        product=ProductQuantizationConfig(
            compression=ProductQuantizationConfig.CompressionRatio.X32,
            always_ram=True
        )
    )
)

# Compression ratios available:
# X4  - 4x compression
# X8  - 8x compression
# X16 - 16x compression
# X32 - 32x compression
# X64 - 64x compression

# Search with rescoring
results = client.search(
    collection_name="documents_pq",
    query_vector=query_vector.tolist(),
    limit=10,
    search_params=SearchParams(
        quantization=QuantizationSearchParams(
            rescore=True,
            oversampling=4.0
        )
    )
)
```

### Changing Quantization on Existing Collection

```python
from qdrant_client.models import ScalarQuantization, ScalarQuantizationConfig, ScalarType

# Update quantization on an existing collection
client.update_collection(
    collection_name="existing_collection",
    quantization_config=ScalarQuantization(
        scalar=ScalarQuantizationConfig(
            type=ScalarType.INT8,
            quantile=0.99,
            always_ram=True
        )
    )
)

# This triggers re-quantization of all vectors in the background
# The collection remains queryable during the process
```

---

## Weaviate Quantization Configuration

### Product Quantization (PQ)

```python
import weaviate
from weaviate.classes.config import Configure, Property, DataType

client = weaviate.connect_to_local()

# Create collection with PQ compression
collection = client.collections.create(
    name="Documents_PQ",
    vectorizer_config=Configure.Vectorizer.none(),
    vector_index_config=Configure.VectorIndex.hnsw(
        distance_metric=Configure.VectorDistances.COSINE,
        ef=100,
        max_connections=24,
        ef_construction=200,
        quantizer=Configure.VectorIndex.Quantizer.pq(
            segments=48,           # Number of PQ subquantizers
            centroids=256,         # Centroids per subspace (256 = 8 bits)
            training_limit=100000, # Training set size
            encoder_type="kmeans", # "kmeans" or "tile"
            encoder_distribution="log-normal"
        )
    ),
    properties=[
        Property(name="content", data_type=DataType.TEXT),
        Property(name="category", data_type=DataType.TEXT),
    ]
)

# Insert data (PQ codebooks are trained automatically after training_limit vectors)
with collection.batch.dynamic() as batch:
    for i in range(len(vectors)):
        batch.add_object(
            properties={"content": f"Document {i}", "category": "test"},
            vector=vectors[i].tolist()
        )

# Search (Weaviate handles rescoring internally)
results = collection.query.near_vector(
    near_vector=query_vector.tolist(),
    limit=10,
    return_metadata=weaviate.classes.query.MetadataQuery(distance=True)
)

for obj in results.objects:
    print(f"Distance: {obj.metadata.distance:.4f} - {obj.properties['content']}")
```

### Binary Quantization (BQ)

```python
collection = client.collections.create(
    name="Documents_BQ",
    vectorizer_config=Configure.Vectorizer.none(),
    vector_index_config=Configure.VectorIndex.hnsw(
        distance_metric=Configure.VectorDistances.COSINE,
        quantizer=Configure.VectorIndex.Quantizer.bq(
            rescore_limit=200  # Number of candidates to rescore
        )
    ),
    properties=[
        Property(name="content", data_type=DataType.TEXT),
    ]
)
```

### Scalar Quantization (SQ)

```python
collection = client.collections.create(
    name="Documents_SQ",
    vectorizer_config=Configure.Vectorizer.none(),
    vector_index_config=Configure.VectorIndex.hnsw(
        distance_metric=Configure.VectorDistances.COSINE,
        quantizer=Configure.VectorIndex.Quantizer.sq(
            training_limit=100000,
            rescore_limit=200
        )
    ),
    properties=[
        Property(name="content", data_type=DataType.TEXT),
    ]
)
```

---

## pgvector: halfvec for 2x Savings

### halfvec (float16) Column Type

```sql
-- Create table with halfvec (float16) column
CREATE TABLE documents (
    id BIGSERIAL PRIMARY KEY,
    content TEXT NOT NULL,
    embedding halfvec(1536),       -- float16, 2 bytes per dimension
    metadata JSONB DEFAULT '{}'
);

-- Create HNSW index on halfvec
CREATE INDEX idx_docs_hnsw ON documents
USING hnsw (embedding halfvec_cosine_ops)
WITH (m = 16, ef_construction = 128);
```

### Python: Insert and Query halfvec

```python
import psycopg
from pgvector.psycopg import register_vector
import numpy as np

conn = psycopg.connect("postgresql://user:pass@localhost/vectordb")
register_vector(conn)

# Insert: Python float list is automatically cast to halfvec
embedding = np.random.rand(1536).astype(np.float32).tolist()
conn.execute(
    "INSERT INTO documents (content, embedding) VALUES (%s, %s::halfvec)",
    ("Hello world", str(embedding))
)
conn.commit()

# Query
query_vec = np.random.rand(1536).astype(np.float32).tolist()
results = conn.execute(
    """
    SELECT id, content, 1 - (embedding <=> %s::halfvec) AS similarity
    FROM documents
    ORDER BY embedding <=> %s::halfvec
    LIMIT 10
    """,
    (str(query_vec), str(query_vec))
).fetchall()

for row in results:
    print(f"ID: {row[0]}, Similarity: {row[2]:.4f} - {row[1][:50]}")
```

### Convert Existing float32 to halfvec

```sql
-- Add halfvec column
ALTER TABLE documents ADD COLUMN embedding_half halfvec(1536);

-- Convert existing float32 vectors to float16
UPDATE documents SET embedding_half = embedding::halfvec;

-- Create new index on halfvec column
CREATE INDEX idx_docs_half_hnsw ON documents
USING hnsw (embedding_half halfvec_cosine_ops)
WITH (m = 16, ef_construction = 128);

-- Drop old index and column (after verifying quality)
DROP INDEX idx_docs_hnsw;
ALTER TABLE documents DROP COLUMN embedding;
ALTER TABLE documents RENAME COLUMN embedding_half TO embedding;
```

### pgvector bit Type (Binary Vectors)

```sql
-- Binary vector column (1536 bits = 192 bytes)
CREATE TABLE documents_binary (
    id BIGSERIAL PRIMARY KEY,
    content TEXT NOT NULL,
    binary_embedding bit(1536)
);

-- Hamming distance index
CREATE INDEX idx_binary_hnsw ON documents_binary
USING hnsw (binary_embedding bit_hamming_ops);

-- Search by Hamming distance
SELECT id, content, binary_embedding <~> B'10110...' AS hamming_distance
FROM documents_binary
ORDER BY binary_embedding <~> B'10110...'
LIMIT 10;
```

### Python: Binary Quantization for pgvector

```python
import psycopg
import numpy as np


def float_to_binary_pgvector(embedding: list[float]) -> str:
    """Convert float32 embedding to binary string for pgvector bit type."""
    binary = ''.join('1' if v > 0 else '0' for v in embedding)
    return binary


# Insert binary vector
embedding = np.random.randn(1536).tolist()
binary_str = float_to_binary_pgvector(embedding)

conn.execute(
    "INSERT INTO documents_binary (content, binary_embedding) VALUES (%s, %s::bit)",
    ("Hello world", binary_str)
)

# Two-stage search: binary first-pass + float32 rescore
def two_stage_search(conn, query_embedding: list[float], k: int = 10,
                     candidates: int = 100):
    """Binary search for candidates, then rescore with exact similarity."""
    binary_query = float_to_binary_pgvector(query_embedding)

    # Stage 1: Fast binary search for candidates
    candidate_rows = conn.execute(
        """
        SELECT id, binary_embedding <~> %s::bit AS hamming_dist
        FROM documents_binary
        ORDER BY binary_embedding <~> %s::bit
        LIMIT %s
        """,
        (binary_query, binary_query, candidates)
    ).fetchall()

    candidate_ids = [row[0] for row in candidate_rows]

    if not candidate_ids:
        return []

    # Stage 2: Rescore with full-precision vectors from main table
    results = conn.execute(
        """
        SELECT d.id, d.content,
               1 - (d.embedding <=> %s::vector) AS similarity
        FROM documents d
        WHERE d.id = ANY(%s)
        ORDER BY d.embedding <=> %s::vector
        LIMIT %s
        """,
        (str(query_embedding), candidate_ids, str(query_embedding), k)
    ).fetchall()

    return results
```

---

## When to Use Which Technique

### Decision Matrix

```
Dataset < 500K vectors?
  YES -> No quantization needed (float32 or float16)
  NO ->
    |
    Already using Qdrant?
      YES -> ScalarQuantization (simplest, best quality/compression)
    |
    Already using Weaviate?
      YES -> SQ or PQ depending on scale
    |
    Already using pgvector?
      YES -> halfvec (2x savings, trivial migration)
    |
    Need maximum compression (>10x)?
      YES -> Faiss IVFPQ or Qdrant PQ
    |
    Need fastest possible search?
      YES -> Binary quantization + rescore
    |
    Default recommendation?
      -> Scalar int8 quantization (4x savings, <1% quality loss)
```

### Quick Reference

| Database | Best First Step | Best for Scale | Code Change Required |
|---|---|---|---|
| Qdrant | `ScalarQuantization(INT8)` | `ProductQuantization(X32)` | Collection config only |
| Weaviate | `Quantizer.sq()` | `Quantizer.pq(segments=48)` | Collection config only |
| pgvector | `halfvec` column type | `bit` + two-stage search | Schema change |
| Faiss | `IndexIVFPQ` | `OPQ + IndexIVFPQ + Refine` | Index build code |
| OpenSearch | `data_type: byte` | Faiss engine with PQ | Index mapping |

### Quantization Cheat Sheet by Scale

| Scale | Recommended | Expected Recall | Memory per 1536d Vector |
|---|---|---|---|
| < 500K | float32 (no quantization) | 0.99+ | 6,144 bytes |
| 500K - 5M | Scalar int8 | 0.985 | 1,536 bytes |
| 5M - 50M | Scalar int8 + rescore | 0.99+ | 1,536 bytes |
| 50M - 200M | PQ (m=48) + rescore | 0.98 | 48 bytes |
| 200M - 1B | PQ (m=48) + DiskANN/rescore | 0.97 | 48 bytes + SSD |
| 1B+ | Binary first-pass + PQ + rescore | 0.95+ | 192 bytes |

---

## Benchmarking Your Quantization

```python
import numpy as np
import time
from typing import Callable


def benchmark_quantization(
    search_fn: Callable,
    queries: np.ndarray,
    ground_truth: np.ndarray,
    k: int = 10,
    warmup: int = 100,
    duration: float = 30.0
) -> dict:
    """Benchmark a quantized search function against ground truth.

    Args:
        search_fn: Function(query) -> (ids, distances)
        queries: Query vectors
        ground_truth: Brute-force top-K indices for each query
        k: Number of results
        warmup: Number of warmup queries
        duration: Benchmark duration in seconds
    """
    # Warmup
    for i in range(warmup):
        _ = search_fn(queries[i % len(queries)])

    # Measure throughput
    num_queries = 0
    latencies = []
    start = time.perf_counter()

    while time.perf_counter() - start < duration:
        q_idx = num_queries % len(queries)
        q_start = time.perf_counter()
        ids, _ = search_fn(queries[q_idx])
        latencies.append((time.perf_counter() - q_start) * 1000)
        num_queries += 1

    elapsed = time.perf_counter() - start

    # Measure recall
    total_recall = 0.0
    for i in range(min(len(queries), 1000)):
        ids, _ = search_fn(queries[i])
        predicted = set(ids[:k])
        actual = set(ground_truth[i][:k])
        total_recall += len(predicted & actual) / k

    recall = total_recall / min(len(queries), 1000)

    return {
        "recall@10": recall,
        "qps": num_queries / elapsed,
        "p50_ms": np.percentile(latencies, 50),
        "p99_ms": np.percentile(latencies, 99),
        "total_queries": num_queries
    }


# Usage example
results = benchmark_quantization(
    search_fn=lambda q: retriever.search(q, k=10, nprobe=32),
    queries=queries,
    ground_truth=gt_indices,
    k=10
)
print(f"Recall: {results['recall@10']:.4f}")
print(f"QPS: {results['qps']:.0f}")
print(f"p50: {results['p50_ms']:.1f}ms, p99: {results['p99_ms']:.1f}ms")
```

---

## Common Pitfalls

1. **Not benchmarking recall before deploying quantization**: always compute recall against brute-force ground truth. A configuration that "feels fast" may have 0.80 recall, silently dropping 20% of relevant results.

2. **Forgetting rescoring for PQ and binary**: PQ alone gives ~0.92 recall. With rescoring top-100, it jumps to ~0.97. This is the most impactful optimization and costs almost nothing in latency.

3. **Using binary quantization with low-dimensional embeddings**: binary quantization on 384-dim vectors loses ~22% recall. It works well with 1024+ dimensions.

4. **Not training PQ on representative data**: PQ codebooks learned on 10K vectors will not generalize well to 10M vectors with different distribution. Use 100K-200K representative training vectors.

5. **Applying Weaviate PQ before the training_limit is reached**: Weaviate trains PQ codebooks after `training_limit` vectors are inserted. If you query before this threshold, you get unquantized (slow) search.

6. **Mixing quantized and unquantized vectors in Qdrant**: when you enable quantization on an existing collection, Qdrant re-quantizes all vectors in the background. Queries during this process may see inconsistent performance.

7. **Not considering pgvector halfvec as the simplest option**: if you use pgvector, switching from `vector` to `halfvec` is a one-line schema change that gives 2x storage savings with <0.5% recall loss. Start here before considering external quantization.

---

## References

- Faiss quantization: https://github.com/facebookresearch/faiss/wiki/Faiss-indexes
- Qdrant quantization: https://qdrant.tech/documentation/guides/quantization/
- Weaviate compression: https://weaviate.io/developers/weaviate/configuration/compression
- pgvector data types: https://github.com/pgvector/pgvector#data-types
- OpenSearch byte vectors: https://opensearch.org/docs/latest/search-plugins/knn/knn-vector-quantization/
