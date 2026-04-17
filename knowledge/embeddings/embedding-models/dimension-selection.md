# Embedding Dimension Selection Guide

## Overview / TL;DR

Embedding dimensions directly affect three things: retrieval quality, storage cost, and search latency. Higher dimensions capture more semantic nuance but consume more memory and slow down similarity computation. With Matryoshka Representation Learning (MRL), many modern models allow truncating embeddings to lower dimensions with graceful quality degradation rather than catastrophic loss. This guide provides the math for storage planning, empirical quality-vs-dimension curves, and concrete recommendations for when 256 dims suffice versus when you need 3072.

---

## How Dimensions Affect Quality

Each dimension in an embedding vector encodes one component of the text's semantic representation. More dimensions mean finer-grained distinctions between concepts.

**Analogy**: Think of a 2D map versus a 3D globe. The 2D map (low dimensions) captures general location but loses altitude, depth, and curvature. The 3D globe (high dimensions) captures everything. For most practical purposes, the 2D map is sufficient -- but for mountainous terrain (complex semantic distinctions), you need the extra dimension.

### Empirical Quality vs Dimension Curves

Based on published benchmarks and reproduction experiments, here are typical MTEB retrieval scores at different dimension levels for Matryoshka-capable models:

**text-embedding-3-large** (native 3072 dims):

| Dimensions | MTEB Retrieval Avg | Relative to Full | Storage per 1M Vectors |
|-----------|-------------------|------------------|----------------------|
| 3072 | 66.1 | 100% | 11.7 GB |
| 1536 | 65.5 | 99.1% | 5.9 GB |
| 1024 | 65.0 | 98.3% | 3.9 GB |
| 512 | 63.8 | 96.5% | 2.0 GB |
| 256 | 61.5 | 93.0% | 1.0 GB |

**nomic-embed-text-v1.5** (native 768 dims):

| Dimensions | MTEB Retrieval Avg | Relative to Full | Storage per 1M Vectors |
|-----------|-------------------|------------------|----------------------|
| 768 | 62.8 | 100% | 2.9 GB |
| 512 | 62.1 | 98.9% | 2.0 GB |
| 256 | 60.5 | 96.3% | 1.0 GB |
| 128 | 57.2 | 91.1% | 0.5 GB |
| 64 | 52.1 | 83.0% | 0.25 GB |

**Key observation**: There is a "quality cliff" below which quality drops sharply. For most models, this cliff is around 128-256 dimensions. Above the cliff, quality degrades gracefully (1-2% per halving). Below it, quality collapses (5-10% per halving).

---

## Storage Math

### Formula

```
Storage (bytes) = N_vectors * dimensions * bytes_per_value
```

Where `bytes_per_value` depends on the numeric type:

| Type | Bytes per Value | Use Case |
|------|----------------|----------|
| float32 | 4 | Default, full precision |
| float16 | 2 | Reduced precision, minimal quality loss |
| int8 (scalar quantization) | 1 | 4x compression, ~1-2% quality loss |
| binary (1 bit) | 0.125 | 32x compression, ~5-10% quality loss |

### Storage Tables

**float32 storage (default)**:

| Dimensions | 100K Vectors | 1M Vectors | 10M Vectors | 100M Vectors |
|-----------|-------------|-----------|------------|-------------|
| 256 | 98 MB | 0.95 GB | 9.5 GB | 95 GB |
| 512 | 195 MB | 1.9 GB | 19 GB | 190 GB |
| 768 | 293 MB | 2.9 GB | 29 GB | 290 GB |
| 1024 | 390 MB | 3.8 GB | 38 GB | 380 GB |
| 1536 | 586 MB | 5.7 GB | 57 GB | 570 GB |
| 3072 | 1.17 GB | 11.4 GB | 114 GB | 1.14 TB |
| 4096 | 1.56 GB | 15.3 GB | 153 GB | 1.53 TB |

**Note**: These figures are for raw vector storage only. Vector database indexes (HNSW, IVF) typically add 1.5-3x overhead. A 3.8 GB vector set at 1024 dims may require 8-12 GB total with HNSW indexing.

### Storage Cost Calculation

```python
def calculate_storage(
    n_vectors: int,
    dimensions: int,
    bytes_per_value: int = 4,
    index_overhead: float = 2.0,
) -> dict:
    """Calculate storage requirements for a vector index.

    Args:
        n_vectors: Number of vectors to store.
        dimensions: Embedding dimensions.
        bytes_per_value: 4 for float32, 2 for float16, 1 for int8.
        index_overhead: Multiplier for index structures (HNSW ~2x, IVF ~1.5x).

    Returns:
        Dictionary with raw and total storage in bytes and human-readable formats.
    """
    raw_bytes = n_vectors * dimensions * bytes_per_value
    total_bytes = raw_bytes * index_overhead

    def human_readable(b: int) -> str:
        for unit in ["B", "KB", "MB", "GB", "TB"]:
            if b < 1024:
                return f"{b:.2f} {unit}"
            b /= 1024
        return f"{b:.2f} PB"

    return {
        "raw_bytes": raw_bytes,
        "total_bytes": int(total_bytes),
        "raw_human": human_readable(raw_bytes),
        "total_human": human_readable(total_bytes),
        "monthly_cost_s3": total_bytes / (1024**3) * 0.023,  # S3 standard pricing
        "monthly_cost_ebs": total_bytes / (1024**3) * 0.08,  # EBS gp3 pricing
    }


# Example: 10M vectors, 1024 dims, float32, HNSW index
result = calculate_storage(10_000_000, 1024, bytes_per_value=4, index_overhead=2.0)
print(f"Raw storage: {result['raw_human']}")
print(f"Total with index: {result['total_human']}")
print(f"Monthly S3 cost: ${result['monthly_cost_s3']:.2f}")
print(f"Monthly EBS cost: ${result['monthly_cost_ebs']:.2f}")

# Output:
# Raw storage: 38.15 GB
# Total with index: 76.29 GB
# Monthly S3 cost: $1.75
# Monthly EBS cost: $6.10
```

---

## Matryoshka Truncation

### How It Works

Matryoshka Representation Learning (MRL) trains embedding models so that the first N dimensions of the full embedding form a valid, useful embedding on their own. The name comes from Russian nesting dolls -- smaller representations are nested inside larger ones.

When a model supports MRL, you can truncate the embedding to any prefix length (e.g., take the first 256 out of 3072 dimensions) and still get meaningful similarity scores. Without MRL, truncating embeddings produces garbage.

### Which Models Support Matryoshka

| Model | MRL Support | Min Useful Dims | Native Dims |
|-------|-----------|----------------|-------------|
| text-embedding-3-small | Yes (API `dimensions` param) | 256 | 1536 |
| text-embedding-3-large | Yes (API `dimensions` param) | 256 | 3072 |
| Voyage-3 / Voyage-3-large | Yes (`output_dimension` param) | 256 | 1024/2048 |
| Jina v3 | Yes (`dimensions` param) | 64 | 1024 |
| nomic-embed-text-v1.5 | Yes (manual truncation) | 64 | 768 |
| mxbai-embed-large-v1 | Yes (manual truncation) | 256 | 1024 |
| BGE-M3 | No | N/A | 1024 |
| E5-Mistral-7B | No | N/A | 4096 |

### Truncation Code

For models with API support (OpenAI, Voyage, Jina), use the built-in parameter:

```python
from openai import OpenAI

client = OpenAI()

# OpenAI handles truncation + normalization internally
response = client.embeddings.create(
    model="text-embedding-3-large",
    input="query text",
    dimensions=256,  # Truncated from 3072
)
embedding = response.data[0].embedding  # Already normalized
```

For models without API support (local models), truncate manually and re-normalize:

```python
import numpy as np
from sentence_transformers import SentenceTransformer


def truncate_embeddings(
    embeddings: np.ndarray,
    target_dims: int,
) -> np.ndarray:
    """Truncate Matryoshka embeddings and re-normalize.

    This ONLY works for models trained with MRL. For non-MRL models,
    truncation destroys the embedding quality.

    Args:
        embeddings: Array of shape (n, full_dims).
        target_dims: Number of dimensions to keep.

    Returns:
        Truncated and L2-normalized embeddings of shape (n, target_dims).
    """
    truncated = embeddings[:, :target_dims]
    norms = np.linalg.norm(truncated, axis=1, keepdims=True)
    # Avoid division by zero
    norms = np.maximum(norms, 1e-12)
    return truncated / norms


# Example with nomic-embed
model = SentenceTransformer("nomic-ai/nomic-embed-text-v1.5", trust_remote_code=True)

texts = [
    "search_query: How does autoscaling work?",
    "search_document: Kubernetes HPA adjusts pods based on CPU.",
]

# Get full embeddings
full_embs = model.encode(texts, normalize_embeddings=True)
print(f"Full dims: {full_embs.shape[1]}")  # 768

# Truncate to 256
truncated_embs = truncate_embeddings(full_embs, 256)
print(f"Truncated dims: {truncated_embs.shape[1]}")  # 256

# Cosine similarity still works
similarity = np.dot(truncated_embs[0], truncated_embs[1])
print(f"Similarity (256 dims): {similarity:.4f}")
```

---

## Dimension Selection by Use Case

### When 256 Dimensions Suffice

- **Pre-filtering / candidate generation**: Using embeddings as a first-pass filter before a re-ranker. The re-ranker corrects errors, so the embedding only needs to be "good enough" to include relevant documents in the top-100 candidates.
- **High-volume, low-latency scenarios**: Processing millions of queries per day where every millisecond of search latency matters.
- **Storage-constrained environments**: Edge deployments, mobile devices, or very large corpora (100M+ vectors) where storage is the bottleneck.
- **Classification / clustering**: Tasks where broad semantic similarity suffices.

### When 512-1024 Dimensions Are Optimal

- **Standard RAG retrieval**: The sweet spot for most production systems. Quality is within 2-4% of full dimensions, storage is manageable, and latency is low.
- **Semantic search applications**: User-facing search where quality matters but sub-100ms latency is required.
- **Medium-scale corpora**: 1M-10M vectors where storage is a consideration but not the primary constraint.

### When 1536-3072+ Dimensions Are Needed

- **High-precision retrieval without re-ranking**: When you cannot afford a re-ranker and the embedding must do all the heavy lifting.
- **Fine-grained semantic distinctions**: Domain-specific text where subtle differences in meaning matter (medical diagnosis descriptions, legal clause variations).
- **Small corpora**: Under 100K vectors where storage is irrelevant and you want maximum quality.
- **Evaluation and benchmarking**: When comparing models or fine-tuning approaches, use full dimensions to isolate model quality from truncation effects.

---

## Latency Impact

Vector similarity search latency scales linearly with dimensions for brute-force search and sub-linearly for index-based search (HNSW, IVF).

### Brute-Force Search (Exact)

```
Latency proportional to: N_vectors * dimensions * bytes_per_value
```

| Dimensions | 1M Vectors (ms) | 10M Vectors (ms) | 100M Vectors (ms) |
|-----------|-----------------|------------------|-------------------|
| 256 | 2 | 20 | 200 |
| 512 | 4 | 40 | 400 |
| 1024 | 8 | 80 | 800 |
| 3072 | 24 | 240 | 2,400 |

### HNSW Index Search (Approximate)

HNSW latency depends on `ef_search` and dimensions but is much less sensitive to corpus size:

| Dimensions | 1M Vectors (ms) | 10M Vectors (ms) | 100M Vectors (ms) |
|-----------|-----------------|------------------|-------------------|
| 256 | 0.5 | 0.8 | 1.5 |
| 512 | 0.8 | 1.2 | 2.0 |
| 1024 | 1.2 | 1.8 | 3.0 |
| 3072 | 3.0 | 4.5 | 7.0 |

### Benchmark Code

```python
import numpy as np
import time


def benchmark_similarity_search(
    n_vectors: int,
    dims: int,
    n_queries: int = 100,
    top_k: int = 10,
) -> dict:
    """Benchmark brute-force cosine similarity search.

    Args:
        n_vectors: Number of vectors in the corpus.
        dims: Embedding dimensions.
        n_queries: Number of queries to average over.
        top_k: Number of results to return.

    Returns:
        Dictionary with timing statistics.
    """
    # Generate random normalized vectors
    corpus = np.random.randn(n_vectors, dims).astype(np.float32)
    corpus /= np.linalg.norm(corpus, axis=1, keepdims=True)

    queries = np.random.randn(n_queries, dims).astype(np.float32)
    queries /= np.linalg.norm(queries, axis=1, keepdims=True)

    # Warm up
    _ = np.dot(queries[:1], corpus.T)

    # Benchmark
    start = time.perf_counter()
    for q in queries:
        scores = np.dot(corpus, q)
        top_indices = np.argpartition(scores, -top_k)[-top_k:]
    elapsed = time.perf_counter() - start

    return {
        "n_vectors": n_vectors,
        "dims": dims,
        "total_time_ms": elapsed * 1000,
        "avg_query_ms": elapsed * 1000 / n_queries,
        "queries_per_sec": n_queries / elapsed,
    }


# Run benchmarks
for dims in [256, 512, 1024, 3072]:
    result = benchmark_similarity_search(1_000_000, dims, n_queries=50)
    print(f"dims={dims}: {result['avg_query_ms']:.1f}ms/query, "
          f"{result['queries_per_sec']:.0f} QPS")
```

---

## Multi-Resolution Strategy

A powerful pattern for production systems is to store embeddings at multiple dimensions and use different resolutions for different stages:

```python
import numpy as np
from dataclasses import dataclass


@dataclass
class MultiResolutionIndex:
    """Store embeddings at multiple dimensions for tiered retrieval.

    Stage 1: Use 256-dim embeddings for fast candidate generation (top-1000).
    Stage 2: Use 1024-dim embeddings for precise re-scoring (top-1000 -> top-50).
    Stage 3: Pass top-50 to a cross-encoder re-ranker.
    """

    vectors_256: np.ndarray    # shape: (n, 256) -- for fast filtering
    vectors_1024: np.ndarray   # shape: (n, 1024) -- for precise scoring
    ids: list[str]

    @classmethod
    def from_full_embeddings(
        cls,
        full_embeddings: np.ndarray,
        ids: list[str],
    ) -> "MultiResolutionIndex":
        """Build multi-resolution index from full Matryoshka embeddings."""
        # Truncate + normalize at each resolution
        v256 = full_embeddings[:, :256]
        v256 /= np.linalg.norm(v256, axis=1, keepdims=True)

        v1024 = full_embeddings[:, :1024]
        v1024 /= np.linalg.norm(v1024, axis=1, keepdims=True)

        return cls(vectors_256=v256, vectors_1024=v1024, ids=ids)

    def search(
        self,
        query_full: np.ndarray,
        top_k: int = 10,
        candidate_pool: int = 1000,
    ) -> list[tuple[str, float]]:
        """Two-stage search: fast filter then precise re-score."""
        # Stage 1: Fast candidate generation with 256 dims
        q256 = query_full[:256]
        q256 = q256 / np.linalg.norm(q256)
        scores_256 = np.dot(self.vectors_256, q256)
        candidate_indices = np.argpartition(scores_256, -candidate_pool)[-candidate_pool:]

        # Stage 2: Precise re-scoring with 1024 dims
        q1024 = query_full[:1024]
        q1024 = q1024 / np.linalg.norm(q1024)
        candidate_vecs = self.vectors_1024[candidate_indices]
        scores_1024 = np.dot(candidate_vecs, q1024)

        # Sort by precise score and return top-k
        top_within_candidates = np.argsort(scores_1024)[-top_k:][::-1]
        results = []
        for idx in top_within_candidates:
            original_idx = candidate_indices[idx]
            results.append((self.ids[original_idx], float(scores_1024[idx])))

        return results
```

---

## Combining Dimension Reduction with Quantization

For extreme compression, combine Matryoshka truncation with scalar or binary quantization:

| Combination | Dims | Quantization | Bytes/Vector | Relative Quality | Compression vs Full |
|------------|------|-------------|-------------|-----------------|-------------------|
| Full (baseline) | 1024 | float32 | 4,096 | 100% | 1x |
| Truncated | 512 | float32 | 2,048 | ~97% | 2x |
| Full + int8 | 1024 | int8 | 1,024 | ~98% | 4x |
| Truncated + int8 | 512 | int8 | 512 | ~95% | 8x |
| Truncated + int8 | 256 | int8 | 256 | ~92% | 16x |
| Truncated + binary | 1024 | binary | 128 | ~88% | 32x |
| Truncated + binary | 256 | binary | 32 | ~80% | 128x |

```python
import numpy as np


def quantize_int8(embeddings: np.ndarray) -> tuple[np.ndarray, np.ndarray, np.ndarray]:
    """Scalar quantize float32 embeddings to int8.

    Returns:
        Tuple of (quantized, scales, zero_points) for dequantization.
    """
    mins = embeddings.min(axis=0)
    maxs = embeddings.max(axis=0)
    scales = (maxs - mins) / 255.0
    scales = np.maximum(scales, 1e-12)  # Avoid division by zero
    zero_points = mins

    quantized = np.round((embeddings - zero_points) / scales).astype(np.int8)
    return quantized, scales, zero_points


def dequantize_int8(
    quantized: np.ndarray,
    scales: np.ndarray,
    zero_points: np.ndarray,
) -> np.ndarray:
    """Dequantize int8 back to float32."""
    return quantized.astype(np.float32) * scales + zero_points


def quantize_binary(embeddings: np.ndarray) -> np.ndarray:
    """Binary quantize: each value becomes 1 bit (positive = 1, negative = 0).

    Uses numpy packbits for compact storage.
    """
    binary = (embeddings > 0).astype(np.uint8)
    packed = np.packbits(binary, axis=1)
    return packed


def hamming_similarity(packed_a: np.ndarray, packed_b: np.ndarray) -> float:
    """Compute similarity using Hamming distance on binary vectors."""
    xor = np.bitwise_xor(packed_a, packed_b)
    hamming_dist = sum(bin(byte).count("1") for byte in xor)
    total_bits = len(packed_a) * 8
    return 1.0 - (hamming_dist / total_bits)


# Example: 256-dim + binary quantization = 32 bytes per vector
embeddings = np.random.randn(1000, 256).astype(np.float32)
packed = quantize_binary(embeddings)
print(f"Original size: {embeddings.nbytes / 1024:.1f} KB")
print(f"Binary packed size: {packed.nbytes / 1024:.1f} KB")
print(f"Compression ratio: {embeddings.nbytes / packed.nbytes:.0f}x")
```

---

## Decision Checklist

- [ ] Determined whether your model supports Matryoshka truncation
- [ ] Calculated storage requirements at target dimensions (vectors + index overhead)
- [ ] Measured retrieval quality at multiple dimension levels on your evaluation set
- [ ] Identified the "quality cliff" dimension for your model (below which quality drops sharply)
- [ ] Selected dimensions based on use case (256 for filtering, 512-1024 for search, full for precision)
- [ ] Considered combining truncation with quantization for large corpora
- [ ] Verified that truncated embeddings are re-normalized after truncation
- [ ] Tested latency at target dimensions against your SLA requirements

---

## References

- Kusupati et al., "Matryoshka Representation Learning" (2022) -- https://arxiv.org/abs/2205.13147
- OpenAI Embedding Guide (dimensions parameter) -- https://platform.openai.com/docs/guides/embeddings
- Nomic Embed: Matryoshka in practice -- https://blog.nomic.ai/posts/nomic-embed-matryoshka-v1.5
- Qdrant Binary Quantization -- https://qdrant.tech/documentation/guides/quantization/
- FAISS Scalar Quantization -- https://github.com/facebookresearch/faiss/wiki/Scalar-Quantizer
