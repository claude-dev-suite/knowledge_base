# Vector Quantization Deep Guide

## Overview

Vector quantization is the process of reducing the memory footprint of vector embeddings by compressing their representation. A single 1536-dimension float32 embedding consumes 6,144 bytes. At 100 million vectors, that is 573 GB of raw vector data alone, plus index overhead. Quantization techniques reduce this by 2x to 32x while preserving retrieval quality through a combination of compressed representations and rescoring patterns. This guide covers the quantization taxonomy, why quantization is essential at scale, full-precision baselines, and the rescoring pattern that makes aggressive quantization practical.

For detailed comparisons of specific techniques and implementation code, see the companion articles on technique comparisons and implementation patterns.

---

## Why Quantize

### The Memory Problem

| Vectors | Dimensions | float32 Size | With HNSW Index (m=16) | Total |
|---|---|---|---|---|
| 100K | 1536 | 0.6 GB | 0.1 GB | 0.7 GB |
| 1M | 1536 | 5.7 GB | 1.3 GB | 7.0 GB |
| 10M | 1536 | 57 GB | 13 GB | 70 GB |
| 100M | 1536 | 573 GB | 128 GB | 701 GB |
| 1B | 1536 | 5,730 GB | 1,280 GB | 7,010 GB |

At 10M vectors, you need a machine with 128+ GB RAM for comfortable operation. At 100M+, you need either extremely expensive hardware, distributed systems, or quantization.

### The Cost Problem

Cloud memory costs roughly $5-10 per GB per month for reserved instances.

| Vectors | float32 RAM Cost/mo | Scalar (int8) Cost/mo | Binary (bit) Cost/mo | PQ (48 subvectors) Cost/mo |
|---|---|---|---|---|
| 1M | $35-70 | $10-20 | $1-2 | $5-10 |
| 10M | $350-700 | $90-175 | $10-20 | $50-100 |
| 100M | $3,500-7,000 | $900-1,750 | $100-200 | $500-1,000 |
| 1B | $35,000-70,000 | $9,000-17,500 | $1,000-2,000 | $5,000-10,000 |

At billion-scale, quantization is not optional -- it is the difference between a viable and an impossible system.

### The Speed Problem

Memory bandwidth is the bottleneck for vector search. Less data per vector means:
- More vectors fit in CPU cache
- Fewer cache misses during graph traversal
- Higher queries per second

```
Approximate speed improvement from quantization:

float32 baseline:     1x
int8 (scalar):        2-4x  (4x less data, some compute overhead)
binary:               10-30x (32x less data + Hamming is cheap)
PQ:                   5-15x  (compressed distance computation)
```

---

## Full-Precision Baselines

### float32 (32 bits per dimension)

The standard precision for embedding models. All models output float32.

```python
import numpy as np

# Standard embedding: 1536 dimensions, float32
embedding = np.random.randn(1536).astype(np.float32)

print(f"Bytes per vector: {embedding.nbytes}")           # 6144
print(f"Bytes per dimension: {embedding.itemsize}")       # 4
print(f"Memory for 1M vectors: {1e6 * 6144 / 1e9:.1f} GB")  # 6.1 GB
```

### float16 (16 bits per dimension)

Half-precision. 2x compression with negligible recall loss for most embedding models.

```python
embedding_f16 = embedding.astype(np.float16)

print(f"Bytes per vector: {embedding_f16.nbytes}")        # 3072
print(f"Compression: {embedding.nbytes / embedding_f16.nbytes:.0f}x")  # 2x
```

**Quality impact**: <0.5% recall loss for text embeddings. Most embedding models do not use the full float32 precision -- the lower bits are effectively noise.

### bfloat16 (16 bits per dimension)

Brain floating point. Same exponent range as float32 but reduced mantissa. Used by Vespa and some ML frameworks.

```python
# NumPy does not natively support bfloat16
# Available via: torch.bfloat16, tensorflow bfloat16, numpy ml_dtypes
import torch

embedding_bf16 = torch.tensor(embedding).bfloat16()
print(f"Bytes per vector: {embedding_bf16.nbytes}")  # 3072
```

**float16 vs bfloat16**: float16 has higher precision in [0, 1] range (more mantissa bits), while bfloat16 handles larger magnitudes (same exponent as float32). For normalized embeddings, float16 is slightly better. For unnormalized, bfloat16 is safer.

---

## The Quantization Spectrum

```
Precision / Quality
    ^
    |  float32        (4 bytes/dim)    100% quality
    |  float16        (2 bytes/dim)    99.5% quality
    |  int8 scalar    (1 byte/dim)     98-99% quality
    |  int4 scalar    (0.5 byte/dim)   95-98% quality
    |  PQ             (~0.1 byte/dim)  90-95% quality
    |  binary         (1 bit/dim)      85-92% quality
    +-----------------------------------------> Compression ratio
         1x   2x   4x   8x   16x   32x
```

### Quantization Categories

| Category | Technique | Compression | Mechanism |
|---|---|---|---|
| **Reduced precision** | float16, bfloat16 | 2x | Fewer bits per float |
| **Scalar quantization** | int8, int4 | 4x, 8x | Linear mapping to integer range |
| **Binary quantization** | 1-bit | 32x | Sign of each dimension |
| **Product quantization** | PQ, OPQ | 10-50x | Subspace codebook lookup |
| **Residual quantization** | RQ | 10-50x | Cascaded codebook refinement |
| **Dimension reduction** | Matryoshka | 2-6x | Truncate to fewer dimensions |

---

## The Rescoring Pattern

The most important pattern in quantized vector search: **search on compressed vectors, rescore on full-precision vectors**.

### How It Works

```
Query
  |
  v
Step 1: Approximate search on compressed index
        - Search HNSW/IVF with quantized vectors
        - Retrieve top-100 candidates (fast, low memory)
  |
  v
Step 2: Rescore top candidates with full-precision vectors
        - Load exact float32 vectors for top-100 from disk/memory
        - Compute exact distances
        - Re-rank and return top-10 (accurate)
  |
  v
Final: Top-10 results with near-exact recall
```

### Why Rescoring Works

1. **The quantized search only needs to get the right candidates into the top-100**, not rank them perfectly. Even with 5% error in distance computation, the true top-10 almost always appear in the top-100 approximate results.

2. **Loading 100 full-precision vectors from disk is fast** (~600 KB for 1536d float32, which is a single SSD read).

3. **The recall improvement is dramatic**: IVFPQ alone might give 0.92 recall@10. IVFPQ + rescoring with 100 candidates gives 0.97+ recall@10.

### Rescoring in Faiss

```python
import faiss
import numpy as np

d = 1536
n = 10_000_000

# Full-precision vectors (on disk or in memory)
vectors = np.random.rand(n, d).astype(np.float32)

# Step 1: Build quantized index for fast search
nlist = 4096
m = 48  # PQ subquantizers
n_bits = 8

quantizer = faiss.IndexFlatL2(d)
index_pq = faiss.IndexIVFPQ(quantizer, d, nlist, m, n_bits)
index_pq.train(vectors[:200_000])
index_pq.add(vectors)

# Step 2: Create a refine index that wraps the PQ index
# It uses the PQ index for candidate generation, then rescores with flat vectors
index_refine = faiss.IndexRefineFlat(index_pq)
index_refine.k_factor = 10  # Search 10x more candidates than k for rescoring

# Search: automatically does PQ search + flat rescore
index_pq.nprobe = 32
distances, indices = index_refine.search(queries, k=10)
```

### Rescoring Quality Impact

| Method | Recall@10 | QPS | Memory |
|---|---|---|---|
| IVFPQ (nprobe=32) | 0.920 | 200 | 520 MB |
| IVFPQ + rescore top-50 | 0.960 | 150 | 520 MB + disk |
| IVFPQ + rescore top-100 | 0.975 | 120 | 520 MB + disk |
| IVFPQ + rescore top-200 | 0.985 | 80 | 520 MB + disk |
| HNSW (float32, ef=200) | 0.994 | 60 | 57 GB |

The IVFPQ + rescore approach achieves 98.5% recall using 520 MB of RAM (for the PQ index) plus disk access for the top-200 vectors, compared to 57 GB for a full-precision HNSW index.

---

## Quantization in Vector Databases

### Database Support Matrix

| Database | Scalar (int8) | Binary (1-bit) | PQ | float16 | Matryoshka |
|---|---|---|---|---|---|
| Qdrant | Yes | Yes | Yes | No | Via dimension param |
| Weaviate | Yes | Yes | Yes | No | Via dimension param |
| Milvus | Yes | No | Yes (IVF_PQ) | Yes | Via dimension param |
| pgvector | halfvec (f16) | bit type | No | Yes (halfvec) | Via dimension param |
| Pinecone | No (auto) | No (auto) | No (auto) | No (auto) | Via dimension param |
| Chroma | No | No | No | No | Via dimension param |
| OpenSearch | byte (int8) | No | Faiss PQ | No | Via dimension param |
| Vespa | int8 tensor | bfloat16 | No | bfloat16 | Via dimension param |

### Matryoshka Dimension Reduction

Not strictly quantization, but achieves similar compression by truncating embedding dimensions.

```python
from openai import OpenAI

client = OpenAI()

# Full 1536 dimensions
response = client.embeddings.create(
    model="text-embedding-3-small",
    input="Hello world",
    dimensions=1536  # default
)

# Reduced to 512 dimensions (3x compression, ~2% quality loss)
response = client.embeddings.create(
    model="text-embedding-3-small",
    input="Hello world",
    dimensions=512
)

# Reduced to 256 dimensions (6x compression, ~5% quality loss)
response = client.embeddings.create(
    model="text-embedding-3-small",
    input="Hello world",
    dimensions=256
)
```

**Matryoshka + Quantization**: these techniques stack. 512 dimensions with int8 scalar quantization gives 12x compression (1536d * 4 bytes -> 512d * 1 byte = 512 bytes vs 6144 bytes).

---

## Quality Impact Summary

### Recall@10 at 1M Vectors (1536d, cosine similarity)

| Quantization | Compression | Recall@10 (no rescore) | Recall@10 (rescore top-100) |
|---|---|---|---|
| float32 (baseline) | 1x | 0.993 (HNSW) | - |
| float16 | 2x | 0.991 | - |
| Scalar int8 | 4x | 0.985 | 0.992 |
| Scalar int4 | 8x | 0.960 | 0.985 |
| PQ (48 subvectors) | ~12x | 0.920 | 0.975 |
| Binary | 32x | 0.870 | 0.960 |

### Guidance by Use Case

| Use Case | Recommended Quantization | Why |
|---|---|---|
| RAG (quality critical) | float16 or scalar int8 | Minimal quality loss |
| Recommendation system | Scalar int8 + rescore | Good balance |
| Large-scale search (10M+) | PQ + rescore | Memory efficiency |
| First-pass retrieval before reranker | Binary + rescore | Maximum speed |
| Prototype / small dataset | None (float32) | Simplicity |

---

## When to Quantize

### Quantize When:

1. **Vector memory exceeds available RAM**: if your HNSW index does not fit in memory, query latency spikes due to disk I/O. Quantize to fit in memory.

2. **Cost of RAM is prohibitive**: at $7/GB/month, 100M vectors at float32 costs $4,900/month in RAM alone. Scalar quantization reduces this to $1,225.

3. **You need higher QPS**: quantized vectors require less memory bandwidth, directly increasing throughput.

4. **You are using a reranker anyway**: if your pipeline includes a cross-encoder reranker on top-K results, the retrieval stage only needs to get good candidates, not perfect rankings. Aggressive quantization + reranking is optimal.

### Do Not Quantize When:

1. **Dataset is small (< 500K vectors)**: float32 fits comfortably in memory. The engineering complexity of quantization is not justified.

2. **Recall is paramount and no reranker exists**: binary quantization with 0.87 recall and no reranker loses 13% of relevant documents. If every missed result matters, use float32 or float16.

3. **You are still iterating on the embedding model**: changing models invalidates trained PQ codebooks. Wait until the model is stable before investing in PQ training.

---

## Quantization Pipeline Architecture

### Typical Production Pipeline

```
Embedding Model
      |
      v
float32 vectors (source of truth, stored on disk/object storage)
      |
      +---> Quantized index (in memory, for fast search)
      |     - Scalar int8 or PQ for the HNSW/IVF index
      |     - Used for candidate generation
      |
      +---> Full-precision store (on disk/SSD)
            - Used for rescoring top-K candidates
            - Memory-mapped or loaded on demand

Query flow:
1. Embed query (float32)
2. Search quantized index -> top-100 candidates
3. Load full-precision vectors for top-100 from disk
4. Compute exact distances, re-rank
5. Return top-10
```

### Implementation Skeleton

```python
import faiss
import numpy as np
from pathlib import Path


class QuantizedSearcher:
    """Two-stage search: quantized index + full-precision rescore."""

    def __init__(self, dim: int, nlist: int = 4096, pq_m: int = 48):
        self.dim = dim
        self.nlist = nlist
        self.pq_m = pq_m
        self.index = None
        self.flat_store = None  # Memory-mapped full-precision vectors

    def build(self, vectors: np.ndarray, store_path: str):
        """Build quantized index and full-precision store."""
        n = vectors.shape[0]

        # Save full-precision vectors as memory-mapped file
        fp = np.memmap(store_path, dtype=np.float32, mode='w+',
                       shape=(n, self.dim))
        fp[:] = vectors[:]
        fp.flush()
        self.flat_store = np.memmap(store_path, dtype=np.float32,
                                    mode='r', shape=(n, self.dim))

        # Build quantized index
        quantizer = faiss.IndexFlatL2(self.dim)
        self.index = faiss.IndexIVFPQ(quantizer, self.dim,
                                       self.nlist, self.pq_m, 8)
        train_size = min(n, 200_000)
        self.index.train(vectors[:train_size])
        self.index.add(vectors)

    def search(self, query: np.ndarray, k: int = 10,
               nprobe: int = 32, rescore_factor: int = 10):
        """Search with quantized index + full-precision rescore."""
        self.index.nprobe = nprobe
        rescore_k = k * rescore_factor

        # Step 1: Approximate search
        _, candidate_ids = self.index.search(
            query.reshape(1, -1), rescore_k
        )
        candidate_ids = candidate_ids[0]
        candidate_ids = candidate_ids[candidate_ids >= 0]  # Remove -1 padding

        # Step 2: Rescore with full-precision vectors
        candidates = self.flat_store[candidate_ids]
        exact_distances = np.linalg.norm(
            candidates - query.reshape(1, -1), axis=1
        )

        # Step 3: Re-rank
        top_k_idx = np.argsort(exact_distances)[:k]
        return candidate_ids[top_k_idx], exact_distances[top_k_idx]
```

---

## Common Pitfalls

1. **Quantizing before the embedding model is finalized**: PQ codebooks are trained on the data distribution. If you change embedding models, you must retrain codebooks. Scalar and binary quantization are model-agnostic.

2. **Not rescoring after aggressive quantization**: binary quantization alone gives ~0.87 recall. With rescoring, it jumps to ~0.96. Always rescore when using PQ or binary quantization.

3. **Applying quantization to already-low-dimensional vectors**: quantizing 128-dim vectors with PQ (m=16 subspaces, 8 dims each) gives poor quality because each subspace has too few dimensions. PQ works best with 256+ dimensions.

4. **Ignoring calibration data for scalar quantization**: scalar quantization maps min/max of each dimension to int8 range. If your calibration data is not representative, outlier dimensions will be clipped, hurting recall.

5. **Assuming quantization is free**: quantized distance computation adds CPU overhead (codebook lookups for PQ, int8 arithmetic for scalar). The net speed improvement depends on the ratio of memory savings to compute overhead.

6. **Not measuring recall at your scale**: quantization recall depends on data distribution. Results at 100K vectors do not predict results at 10M vectors. Always benchmark at target scale.

---

## References

- Product Quantization for Nearest Neighbor Search: Jegou et al., 2011 (IEEE TPAMI)
- Optimized Product Quantization: Ge et al., 2013 (IEEE TPAMI)
- ScaNN: Efficient Vector Similarity Search: Guo et al., 2020 (ICML)
- Faiss: A Library for Efficient Similarity Search: https://github.com/facebookresearch/faiss
- Binary Quantization in Qdrant: https://qdrant.tech/articles/binary-quantization/
- Matryoshka Representation Learning: Kusupati et al., 2022 (NeurIPS)
- RaBitQ: Gao et al., 2024
