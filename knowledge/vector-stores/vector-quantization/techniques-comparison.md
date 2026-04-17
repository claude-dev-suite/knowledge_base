# Vector Quantization -- Techniques Comparison

## Overview

This guide provides a detailed comparison of all major vector quantization techniques: scalar quantization (int8, int4), binary quantization, product quantization (PQ), optimized PQ (OPQ), residual quantization (RQ), and Matryoshka dimension truncation. For each technique, we cover the mechanism, storage math, quality impact, and when to use it. The goal is to help you select the right quantization technique for your scale, recall requirements, and budget.

---

## Scalar Quantization (SQ)

### Mechanism

Scalar quantization maps each float32 dimension independently to a reduced integer range. The mapping is linear: find the min and max of each dimension across the dataset, then scale to the target integer range.

```
For int8 (range 0-255):
  quantized[i] = round((value[i] - min[i]) / (max[i] - min[i]) * 255)

For dequantization:
  value[i] = quantized[i] / 255 * (max[i] - min[i]) + min[i]
```

### int8 Scalar Quantization

```python
import numpy as np


def scalar_quantize_int8(vectors: np.ndarray) -> tuple:
    """Quantize float32 vectors to int8."""
    # Compute per-dimension min/max from the dataset
    v_min = vectors.min(axis=0)
    v_max = vectors.max(axis=0)

    # Avoid division by zero for constant dimensions
    v_range = v_max - v_min
    v_range[v_range == 0] = 1.0

    # Scale to [0, 255] and cast to uint8
    scaled = (vectors - v_min) / v_range * 255.0
    quantized = np.clip(scaled, 0, 255).astype(np.uint8)

    return quantized, v_min, v_range


def scalar_dequantize_int8(quantized: np.ndarray, v_min: np.ndarray,
                            v_range: np.ndarray) -> np.ndarray:
    """Dequantize int8 vectors back to float32."""
    return (quantized.astype(np.float32) / 255.0) * v_range + v_min


# Storage math
d = 1536
n = 1_000_000

float32_bytes = n * d * 4              # 5.73 GB
int8_bytes = n * d * 1 + d * 4 * 2    # 1.44 GB + 12 KB (min/max)
compression = float32_bytes / int8_bytes
print(f"Compression: {compression:.1f}x")  # ~4x
```

### int4 Scalar Quantization

Each dimension stored in 4 bits (range 0-15). Two dimensions packed into one byte.

```python
def scalar_quantize_int4(vectors: np.ndarray) -> tuple:
    """Quantize float32 vectors to int4 (packed into bytes)."""
    v_min = vectors.min(axis=0)
    v_max = vectors.max(axis=0)
    v_range = v_max - v_min
    v_range[v_range == 0] = 1.0

    # Scale to [0, 15]
    scaled = (vectors - v_min) / v_range * 15.0
    quantized = np.clip(scaled, 0, 15).astype(np.uint8)

    # Pack two values per byte
    n, d = quantized.shape
    assert d % 2 == 0, "Dimensions must be even for int4 packing"
    packed = (quantized[:, 0::2] << 4) | quantized[:, 1::2]

    return packed, v_min, v_range


# Storage math
d = 1536
n = 1_000_000

float32_bytes = n * d * 4          # 5.73 GB
int4_bytes = n * d // 2 + d * 4 * 2  # 0.72 GB + 12 KB
compression = float32_bytes / int4_bytes
print(f"Compression: {compression:.1f}x")  # ~8x
```

### Scalar Quantization Quality

| Method | Compression | Recall@10 (no rescore) | Recall@10 (rescore top-100) |
|---|---|---|---|
| float32 | 1x | 0.993 | - |
| int8 | 4x | 0.985 | 0.992 |
| int4 | 8x | 0.960 | 0.985 |

**When to use scalar quantization**:
- First choice for production (simple, effective)
- Good for any embedding model (no training required)
- Can be applied after the fact without rebuilding the index
- Stacks well with HNSW and IVF

---

## Binary Quantization (BQ)

### Mechanism

Binary quantization converts each float32 dimension to a single bit: positive values become 1, negative values become 0. Similarity is computed using Hamming distance (number of differing bits), which is extremely fast using CPU popcount instructions.

```python
import numpy as np


def binary_quantize(vectors: np.ndarray) -> np.ndarray:
    """Quantize float32 vectors to binary (one bit per dimension)."""
    # Center the vectors (important for quality)
    mean = vectors.mean(axis=0)
    centered = vectors - mean

    # Positive -> 1, Negative -> 0
    bits = (centered > 0).astype(np.uint8)

    # Pack 8 bits per byte
    n, d = bits.shape
    assert d % 8 == 0
    packed = np.packbits(bits, axis=1)

    return packed, mean


def binary_hamming_distance(a: np.ndarray, b: np.ndarray) -> int:
    """Compute Hamming distance between two packed binary vectors."""
    xor = np.bitwise_xor(a, b)
    return sum(bin(byte).count('1') for byte in xor)


# Storage math
d = 1536
n = 1_000_000

float32_bytes = n * d * 4          # 5.73 GB
binary_bytes = n * d // 8 + d * 4  # 0.18 GB + 6 KB (mean)
compression = float32_bytes / binary_bytes
print(f"Compression: {compression:.1f}x")  # ~32x
```

### Speed Advantage

```python
# Hamming distance comparison is ~10-30x faster than float32 cosine
import time

d = 1536
n_queries = 10000

# Float32 cosine distance
a = np.random.randn(d).astype(np.float32)
B = np.random.randn(n_queries, d).astype(np.float32)

start = time.perf_counter()
for _ in range(100):
    dists = 1.0 - B @ a / (np.linalg.norm(B, axis=1) * np.linalg.norm(a))
float_time = time.perf_counter() - start

# Binary Hamming distance
a_bin = np.packbits((a > 0).astype(np.uint8))
B_bin = np.packbits((B > 0).astype(np.uint8), axis=1)

start = time.perf_counter()
for _ in range(100):
    xor = np.bitwise_xor(B_bin, a_bin)
    hamming = np.unpackbits(xor, axis=1).sum(axis=1)
binary_time = time.perf_counter() - start

print(f"Float32: {float_time:.3f}s")
print(f"Binary:  {binary_time:.3f}s")
print(f"Speedup: {float_time/binary_time:.1f}x")
```

### Binary Quantization Quality

| Embedding Model | Recall@10 (BQ only) | Recall@10 (BQ + rescore 100) |
|---|---|---|
| OpenAI text-embedding-3-small (1536d) | 0.87 | 0.96 |
| OpenAI text-embedding-3-large (3072d) | 0.90 | 0.97 |
| Cohere embed-english-v3.0 (1024d) | 0.84 | 0.94 |
| BGE-small-en-v1.5 (384d) | 0.78 | 0.90 |

**Key insight**: binary quantization quality depends heavily on dimensionality. Higher dimensions = more bits = better binary recall. Models with 1024+ dimensions work well. Models with <512 dimensions lose too much information.

**When to use binary quantization**:
- Maximum compression needed (32x)
- High-dimensional embeddings (1024+)
- Combined with rescoring for quality recovery
- First-pass retrieval before a cross-encoder reranker

---

## Product Quantization (PQ)

### Mechanism

Product quantization divides the vector into m subspaces, then independently quantizes each subspace using k-means clustering (typically k=256 with 8-bit codes). Each subspace is represented by a single byte -- the index of the nearest centroid (codebook entry).

```
Original vector (1536 float32 = 6144 bytes):
[d0, d1, d2, ..., d31 | d32, d33, ..., d63 | ... | d1504, ..., d1535]
 <--- subspace 0 --->   <--- subspace 1 --->       <--- subspace 47 ->

PQ code (48 bytes):
[code0, code1, code2, ..., code47]
 byte    byte   byte        byte

Each code is an index (0-255) into a learned codebook of 256 centroids
for that subspace.
```

```python
import faiss
import numpy as np


def train_pq(vectors: np.ndarray, m: int = 48, n_bits: int = 8) -> faiss.IndexPQ:
    """Train a Product Quantizer on the given vectors."""
    d = vectors.shape[1]
    assert d % m == 0, f"Dimensions {d} must be divisible by m={m}"

    index = faiss.IndexPQ(d, m, n_bits)
    index.train(vectors)
    index.add(vectors)
    return index


# Storage math
d = 1536
m = 48  # subquantizers
n = 1_000_000

float32_bytes = n * d * 4          # 5.73 GB
pq_bytes = n * m * 1               # 0.046 GB (48 bytes per vector)
codebook_bytes = m * 256 * (d // m) * 4  # 0.6 MB (codebooks, negligible)
compression = float32_bytes / pq_bytes
print(f"Compression: {compression:.1f}x")  # ~128x

# With m=96
pq_bytes_96 = n * 96 * 1          # 0.092 GB
compression_96 = float32_bytes / pq_bytes_96
print(f"Compression (m=96): {compression_96:.1f}x")  # ~64x
```

### PQ Distance Computation

PQ computes distances without decompressing vectors using a lookup table:

```python
def pq_distance_table(query: np.ndarray, codebooks: np.ndarray) -> np.ndarray:
    """Precompute distance from query to all codebook entries.

    Returns a table of shape (m, 256) where table[j][c] is the
    distance from query subvector j to centroid c of codebook j.
    """
    m = codebooks.shape[0]
    k = codebooks.shape[1]  # 256
    subvec_size = query.shape[0] // m

    table = np.zeros((m, k), dtype=np.float32)
    for j in range(m):
        query_sub = query[j * subvec_size : (j + 1) * subvec_size]
        for c in range(k):
            diff = query_sub - codebooks[j][c]
            table[j][c] = np.dot(diff, diff)  # L2 distance
    return table


def pq_compute_distance(table: np.ndarray, codes: np.ndarray) -> float:
    """Compute approximate distance using PQ lookup table.

    table: (m, 256) precomputed distances
    codes: (m,) PQ codes for one vector
    """
    return sum(table[j][codes[j]] for j in range(len(codes)))
```

### PQ Parameter Selection

| Parameter | Values | Effect |
|---|---|---|
| m (subquantizers) | 8, 16, 32, 48, 64, 96 | More = better quality, larger codes |
| n_bits | 8 (standard), 4, 12, 16 | More bits = more centroids = better quality |
| Subvector size | d / m | Smaller subvectors = more compression but lower quality |

**Guidelines for m selection**:

| Dimensions | Min m | Recommended m | Max m | Bytes per vector |
|---|---|---|---|---|
| 384 | 8 | 48 | 48 | 48 |
| 768 | 16 | 48-96 | 96 | 48-96 |
| 1536 | 32 | 48-96 | 192 | 48-192 |
| 3072 | 48 | 96-192 | 384 | 96-384 |

Rule of thumb: each subvector should have at least 4-8 dimensions. Below 4 dimensions per subspace, quality degrades rapidly.

### PQ Quality

| m | Subvector dims (1536d) | Bytes/vector | Recall@10 (no rescore) | Recall@10 (rescore 100) |
|---|---|---|---|---|
| 16 | 96 | 16 | 0.85 | 0.95 |
| 32 | 48 | 32 | 0.90 | 0.97 |
| 48 | 32 | 48 | 0.92 | 0.975 |
| 96 | 16 | 96 | 0.95 | 0.99 |
| 192 | 8 | 192 | 0.97 | 0.995 |

---

## Optimized Product Quantization (OPQ)

### Mechanism

OPQ applies a learned rotation matrix to the vectors before PQ encoding. The rotation aligns the data distribution with the PQ subspace boundaries, reducing quantization error.

```python
import faiss
import numpy as np

d = 1536
m = 48

# Standard PQ
index_pq = faiss.IndexPQ(d, m, 8)
index_pq.train(vectors)

# Optimized PQ (with learned rotation)
index_opq = faiss.IndexPQ(d, m, 8)
opq = faiss.OPQMatrix(d, m)
index_opq = faiss.IndexPreTransform(opq, index_pq)
index_opq.train(vectors)
index_opq.add(vectors)
```

### OPQ vs PQ Quality

| Method | Bytes/vector | Recall@10 | Improvement over PQ |
|---|---|---|---|
| PQ (m=48) | 48 | 0.920 | baseline |
| OPQ (m=48) | 48 | 0.935 | +1.5% |
| PQ (m=96) | 96 | 0.950 | - |
| OPQ (m=96) | 96 | 0.960 | +1.0% |

OPQ provides a modest but consistent improvement (1-3% recall) at zero additional storage cost. The only cost is longer training time (learning the rotation matrix).

---

## Residual Quantization (RQ)

### Mechanism

Residual quantization applies quantization in stages. Each stage quantizes the residual (error) from the previous stage, progressively refining the approximation.

```
Stage 1: Quantize vector -> code1, compute residual1 = vector - decode(code1)
Stage 2: Quantize residual1 -> code2, compute residual2 = residual1 - decode(code2)
Stage 3: Quantize residual2 -> code3, ...

Final code = [code1, code2, code3, ...]
```

```python
import faiss
import numpy as np

d = 1536
M = 48  # number of codebooks (same as PQ m)

# Residual Quantizer in Faiss
rq = faiss.ResidualQuantizer(d, M, 8)  # d, M codebooks, 8 bits each
rq.train(vectors[:100000])

# Encode
codes = rq.compute_codes(vectors)  # shape: (n, M)

# Decode (approximate reconstruction)
reconstructed = rq.decode(codes)

# Quality comparison
reconstruction_error = np.mean(np.linalg.norm(vectors - reconstructed, axis=1))
print(f"Mean reconstruction error: {reconstruction_error:.4f}")
```

### RQ vs PQ Quality

| Method | Bytes/vector | Reconstruction Error | Recall@10 |
|---|---|---|---|
| PQ (m=48) | 48 | 0.45 | 0.920 |
| OPQ (m=48) | 48 | 0.42 | 0.935 |
| RQ (M=48) | 48 | 0.38 | 0.945 |
| RQ (M=48, with beam search decode) | 48 | 0.35 | 0.955 |

RQ achieves better reconstruction quality than PQ at the same code size, but is slower to encode and decode.

---

## Matryoshka Dimension Truncation

### Mechanism

Matryoshka Representation Learning (MRL) trains embedding models so that the first k dimensions of the full embedding are a valid k-dimensional embedding. This means you can simply truncate dimensions for compression.

```python
from openai import OpenAI

client = OpenAI()


def get_embedding(text: str, dimensions: int = 1536) -> list[float]:
    """Get embedding with truncated dimensions (Matryoshka)."""
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=text,
        dimensions=dimensions
    )
    return response.data[0].embedding


# Storage comparison
full = get_embedding("Hello world", dimensions=1536)  # 6144 bytes
half = get_embedding("Hello world", dimensions=768)    # 3072 bytes (2x)
quarter = get_embedding("Hello world", dimensions=512)  # 2048 bytes (3x)
eighth = get_embedding("Hello world", dimensions=256)   # 1024 bytes (6x)
```

### Matryoshka Quality

| Dimensions | Compression vs 1536d | Recall@10 | MTEB Score (approx) |
|---|---|---|---|
| 1536 | 1x | 0.993 | 62.3 |
| 1024 | 1.5x | 0.990 | 62.0 |
| 768 | 2x | 0.987 | 61.5 |
| 512 | 3x | 0.980 | 60.5 |
| 256 | 6x | 0.960 | 58.0 |
| 128 | 12x | 0.920 | 54.0 |

### Stacking Matryoshka with Quantization

Matryoshka truncation and scalar/binary quantization are orthogonal techniques that stack:

| Configuration | Total Compression | Bytes per Vector | Recall@10 |
|---|---|---|---|
| 1536d float32 | 1x | 6,144 | 0.993 |
| 1536d int8 | 4x | 1,536 | 0.985 |
| 768d float32 | 2x | 3,072 | 0.987 |
| 768d int8 | 8x | 768 | 0.978 |
| 512d int8 | 12x | 512 | 0.970 |
| 512d binary | 96x | 64 | 0.910 |
| 256d int8 | 24x | 256 | 0.950 |

---

## Comprehensive Storage Comparison

### Per-Vector Storage (1536-dimensional embeddings)

| Technique | Bytes/Vector | Compression | Storage (1M vectors) | Storage (100M vectors) |
|---|---|---|---|---|
| float32 | 6,144 | 1x | 5.73 GB | 573 GB |
| float16 | 3,072 | 2x | 2.86 GB | 286 GB |
| int8 (scalar) | 1,536 | 4x | 1.43 GB | 143 GB |
| int4 (scalar) | 768 | 8x | 0.72 GB | 72 GB |
| PQ (m=48) | 48 | 128x | 0.045 GB | 4.5 GB |
| PQ (m=96) | 96 | 64x | 0.09 GB | 9 GB |
| Binary | 192 | 32x | 0.18 GB | 18 GB |
| Matryoshka 512d f32 | 2,048 | 3x | 1.91 GB | 191 GB |
| Matryoshka 512d int8 | 512 | 12x | 0.48 GB | 48 GB |

### Quality Comparison (Recall@10 at 1M vectors, HNSW)

| Technique | No Rescore | Rescore Top-50 | Rescore Top-100 | Rescore Top-200 |
|---|---|---|---|---|
| float32 | 0.993 | - | - | - |
| float16 | 0.991 | - | - | - |
| int8 | 0.985 | 0.990 | 0.992 | 0.993 |
| int4 | 0.960 | 0.978 | 0.985 | 0.990 |
| PQ (m=48) | 0.920 | 0.955 | 0.975 | 0.985 |
| PQ (m=96) | 0.950 | 0.980 | 0.990 | 0.993 |
| Binary | 0.870 | 0.935 | 0.960 | 0.975 |
| Matryoshka 512d | 0.980 | - | - | - |

---

## Decision Matrix

### Choose Based on Your Constraints

| Primary Constraint | Recommended Technique | Why |
|---|---|---|
| Simplicity | Scalar int8 | No training, drop-in replacement |
| Maximum compression | Binary + rescore | 32x compression, fast Hamming |
| Balanced compression + quality | PQ (m=48-96) + rescore | 64-128x compression, trainable |
| Model supports it | Matryoshka truncation | Zero overhead, quality-preserving |
| Quality-critical (RAG) | float16 or int8 | <1% recall loss |
| Cost-critical (billion-scale) | PQ or Binary + rescore | 10-32x memory reduction |
| Reranker in pipeline | Binary first-pass | Only need candidates, not ranking |
| Small dataset (< 500K) | None (float32) | Not worth the complexity |

### Choose Based on Your Scale

| Vectors | Budget | Recommended | Expected Recall@10 |
|---|---|---|---|
| < 500K | Any | float32 or float16 | 0.99+ |
| 500K - 5M | Standard | Scalar int8 | 0.985 |
| 5M - 50M | Standard | Scalar int8 + HNSW | 0.98 |
| 5M - 50M | Constrained | PQ (m=96) + rescore | 0.98 |
| 50M - 500M | Standard | PQ (m=48) + rescore | 0.97 |
| 50M - 500M | Constrained | Binary + rescore | 0.96 |
| 500M+ | Any | PQ + DiskANN/IVF | 0.95+ |

---

## Common Pitfalls

1. **Applying PQ without sufficient training data**: PQ codebooks need 30-50x the number of centroids in training data. With 256 centroids per subspace, you need at least 10K training vectors (100K+ recommended).

2. **Using binary quantization with low-dimensional embeddings**: binary quantization on 384-dim vectors loses ~22% recall. It works best with 1024+ dimensions.

3. **Not centering vectors before binary quantization**: binary quantization thresholds at zero. If vectors have a non-zero mean, centering before quantization improves quality by 3-5%.

4. **Assuming quantization quality is independent of the embedding model**: different models have different value distributions. An embedding model with high kurtosis (sharp peaks) quantizes differently than one with uniform values.

5. **Comparing compressed recall without rescoring**: binary quantization at 0.87 recall sounds terrible, but with rescoring it reaches 0.96+. Always evaluate the full pipeline.

6. **Ignoring Matryoshka when available**: if your embedding model supports Matryoshka dimensions (OpenAI text-embedding-3-*, Nomic, JinaAI), dimension truncation is free compression with no training step.

---

## References

- Product Quantization: Jegou et al., "Product Quantization for Nearest Neighbor Search," IEEE TPAMI, 2011
- Optimized Product Quantization: Ge et al., "Optimized Product Quantization," IEEE TPAMI, 2014
- Residual Quantization: Martinez et al., "Stacking Quantizers for Compositional Vector Compression," 2018
- Matryoshka Representation Learning: Kusupati et al., NeurIPS 2022
- Binary Quantization analysis: Qdrant blog, "Binary Quantization," 2024
- ScaNN: Guo et al., "Accelerating Large-Scale Inference with Anisotropic Vector Quantization," ICML 2020
- Faiss quantization: https://github.com/facebookresearch/faiss/wiki/Faiss-indexes
