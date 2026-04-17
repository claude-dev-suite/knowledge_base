# IVF + PQ Algorithm Deep Dive

## Overview

IVF (Inverted File Index) combined with PQ (Product Quantization) is the go-to algorithm for vector search at scales where HNSW's memory requirements become prohibitive. While HNSW stores the full vector at each graph node (requiring ~6 KB per 1536-dim float32 vector), IVF+PQ compresses vectors to 48-96 bytes using codebook-based quantization, enabling 100M-1B vector deployments on commodity hardware. This guide covers the IVF coarse quantizer, residual computation, PQ subspace decomposition, codebook training, the nprobe parameter, Faiss implementation, comparison with HNSW, and DiskANN as a hybrid alternative.

---

## IVF: Inverted File Index

### Concept

IVF partitions the vector space into `nlist` regions using k-means clustering. Each region has a centroid. At search time, only the nearest `nprobe` regions are searched, skipping the rest.

```
Training Phase:
  1. Run k-means on training data to find nlist centroids
  2. Assign each vector to its nearest centroid
  3. Build inverted lists: centroid -> [vector IDs in that region]

Search Phase:
  1. Find nprobe nearest centroids to query
  2. Search only vectors in those nprobe regions
  3. Return top-k from the combined candidate set
```

### Visual Representation

```
Vector Space (2D for illustration)

    nlist = 4 clusters:
    
    Cluster A           Cluster B
    . . .               . . .
    . * .               . * .
    . . .               . . .
    
    Cluster C           Cluster D
    . . .               . . .
    . * .               . * .
    . . .               . . .
    
    * = centroid
    . = vectors assigned to that cluster
    
    Query: Q
    nprobe=2: search clusters A and B (nearest 2 centroids to Q)
    Skip: clusters C and D (not searched)
```

### IVF in Faiss

```python
import faiss
import numpy as np

d = 1536
nlist = 4096    # Number of partitions

# Create coarse quantizer (centroid index)
quantizer = faiss.IndexFlatL2(d)

# Create IVF index with flat storage (no compression)
index = faiss.IndexIVFFlat(quantizer, d, nlist)

# Training: learns the nlist centroids via k-means
# Need at least 30 * nlist training vectors
train_data = vectors[:200_000]
index.train(train_data)

# Add vectors (assigns each to nearest centroid)
index.add(vectors)

# Search: set nprobe to control accuracy/speed tradeoff
index.nprobe = 32
distances, indices = index.search(queries, k=10)
```

### nlist Selection

| Dataset Size | Recommended nlist | Rationale |
|---|---|---|
| 10K - 100K | 64 - 256 | Small clusters for adequate per-cluster size |
| 100K - 1M | 256 - 1024 | sqrt(N) heuristic |
| 1M - 10M | 1024 - 4096 | Balance search speed and recall |
| 10M - 100M | 4096 - 16384 | More partitions for larger data |
| 100M - 1B | 16384 - 65536 | Very fine partitioning |

Rule of thumb: `nlist ~ sqrt(N)` to `4 * sqrt(N)`.

### nprobe Parameter

nprobe controls how many of the nlist partitions are searched per query.

```python
# Recall vs nprobe at 1M vectors, nlist=1024
index.nprobe = 1     # recall ~0.30, fastest
index.nprobe = 4     # recall ~0.55
index.nprobe = 8     # recall ~0.70
index.nprobe = 16    # recall ~0.82
index.nprobe = 32    # recall ~0.90
index.nprobe = 64    # recall ~0.95
index.nprobe = 128   # recall ~0.98
index.nprobe = 256   # recall ~0.99
index.nprobe = 1024  # recall ~1.00 (exhaustive, defeats purpose)
```

**Guidance**: set nprobe to 1-10% of nlist for a good recall/speed balance. At nlist=4096, nprobe=32-64 gives 0.93-0.96 recall.

---

## PQ: Product Quantization

### Concept

Product quantization compresses vectors by decomposing them into m subspaces, then replacing each subspace with a codebook index (typically 8 bits = 256 centroids).

```
Original vector: [d0, d1, ..., d1535]  (1536 float32 = 6144 bytes)
                   |          |          |
                   v          v          v
Subspace 0:     [d0...d31]  ->  code0 (1 byte, index into 256 centroids)
Subspace 1:     [d32...d63] ->  code1 (1 byte)
...
Subspace 47:    [d1504...d1535] -> code47 (1 byte)

PQ code: [code0, code1, ..., code47]  (48 bytes)

Compression: 6144 / 48 = 128x
```

### Codebook Training

```python
import faiss
import numpy as np

d = 1536
m = 48       # Number of subquantizers
n_bits = 8   # Bits per code (256 centroids per subspace)

# Product Quantizer
pq = faiss.ProductQuantizer(d, m, n_bits)

# Train codebooks on representative data
# For each of the m subspaces, learn 256 centroids using k-means
train_data = vectors[:100_000].astype(np.float32)
pq.train(train_data)

# Encode vectors to PQ codes
codes = pq.compute_codes(vectors)  # shape: (N, 48), dtype: uint8

# Decode (approximate reconstruction)
reconstructed = pq.decode(codes)    # shape: (N, 1536), dtype: float32

# Reconstruction error
error = np.mean(np.linalg.norm(vectors - reconstructed, axis=1))
print(f"Mean reconstruction error: {error:.4f}")
```

### PQ Distance Computation

The key insight: distances can be computed from PQ codes using precomputed lookup tables, without decompressing vectors.

```python
def asymmetric_distance_computation(query, pq, codes):
    """Compute distances between a float32 query and PQ-encoded database.

    This is "asymmetric" because the query is full-precision but the
    database vectors are quantized. This gives better accuracy than
    symmetric (both quantized) computation.
    """
    m = pq.M
    k = pq.ksub  # 256
    dsub = pq.dsub  # d / m

    # Step 1: Precompute distance table
    # For each subspace j, compute distance from query subvector to all 256 centroids
    table = np.zeros((m, k), dtype=np.float32)
    for j in range(m):
        query_sub = query[j * dsub : (j + 1) * dsub]
        centroids = faiss.vector_to_array(pq.centroids).reshape(m, k, dsub)
        for c in range(k):
            diff = query_sub - centroids[j, c]
            table[j, c] = np.dot(diff, diff)

    # Step 2: Look up distances for each database vector
    # Total distance = sum of subspace distances
    n = codes.shape[0]
    distances = np.zeros(n, dtype=np.float32)
    for i in range(n):
        for j in range(m):
            distances[i] += table[j, codes[i, j]]

    return distances


# In practice, Faiss implements this efficiently using SIMD
# The table precomputation is O(m * k * dsub)
# The distance lookup is O(N * m) -- much faster than O(N * d) brute force
```

### Subspace Decomposition

```
Full vector (1536 dimensions, float32):
|<---- d0-d31 ---->|<---- d32-d63 ---->|...|<---- d1504-d1535 ---->|
     Subspace 0         Subspace 1               Subspace 47

Each subspace: 32 dimensions
Codebook: 256 centroids per subspace, each 32-dim
Code: 1 byte per subspace (index 0-255)

Total code: 48 bytes
Total codebook: 48 * 256 * 32 * 4 = 1.5 MB (tiny, fits in L1 cache)
```

### Effect of m on Quality and Speed

| m | Subvec dims (1536d) | Bytes/vector | Recall@10 (IVF+PQ) | Distance compute speed |
|---|---|---|---|---|
| 8 | 192 | 8 | 0.75 | Fastest |
| 16 | 96 | 16 | 0.85 | Fast |
| 32 | 48 | 32 | 0.90 | Moderate |
| 48 | 32 | 48 | 0.92 | Moderate |
| 64 | 24 | 64 | 0.94 | Slower |
| 96 | 16 | 96 | 0.96 | Slow |
| 192 | 8 | 192 | 0.97 | Slowest |

Each subvector should have at least 4-8 dimensions for adequate quantization quality.

---

## IVF + PQ Combined

### Architecture

```
IVF+PQ Index Structure:

Coarse Quantizer (nlist centroids, full precision)
  |
  +-- Centroid 0 -> Inverted List 0: [PQ(v1), PQ(v5), PQ(v12), ...]
  |
  +-- Centroid 1 -> Inverted List 1: [PQ(v2), PQ(v7), PQ(v19), ...]
  |
  +-- Centroid 2 -> Inverted List 2: [PQ(v3), PQ(v8), PQ(v23), ...]
  |
  ...
  |
  +-- Centroid nlist-1 -> Inverted List nlist-1: [PQ(v4), ...]

Search:
  1. Find nprobe nearest centroids to query
  2. For each vector in those nprobe inverted lists:
     - Compute residual distance using PQ lookup table
  3. Return top-k
```

### Residual Computation

A critical optimization: PQ encodes the **residual** (vector - centroid), not the original vector. This reduces quantization error because residuals have smaller variance.

```python
# Without residual (naive):
# PQ encodes the full vector v
# Error = ||v - PQ_decode(PQ_encode(v))||

# With residual:
# centroid = nearest_centroid(v)
# residual = v - centroid
# PQ encodes the residual
# Error = ||residual - PQ_decode(PQ_encode(residual))|| << naive error
```

```python
import faiss
import numpy as np

d = 1536
nlist = 4096
m = 48       # PQ subquantizers
n_bits = 8   # 256 centroids per subspace

# Create IVF+PQ index
quantizer = faiss.IndexFlatL2(d)
index = faiss.IndexIVFPQ(quantizer, d, nlist, m, n_bits)

# Train: learns both IVF centroids and PQ codebooks
# PQ codebooks are trained on residuals (vector - nearest centroid)
train_data = vectors[:200_000]
index.train(train_data)

# Add vectors
index.add(vectors)

# Search
index.nprobe = 32
distances, indices = index.search(queries, k=10)
```

### IVFPQ with Rescoring

```python
import faiss
import numpy as np

d = 1536
nlist = 4096
m = 48

# Build IVFPQ index
quantizer = faiss.IndexFlatL2(d)
index_ivfpq = faiss.IndexIVFPQ(quantizer, d, nlist, m, 8)
index_ivfpq.train(vectors[:200_000])
index_ivfpq.add(vectors)

# Method 1: IndexRefineFlat (automatic rescoring)
index_refine = faiss.IndexRefineFlat(index_ivfpq)
index_refine.k_factor = 10  # Over-retrieve 10x, then rescore

index_ivfpq.nprobe = 32
distances, indices = index_refine.search(queries, k=10)

# Method 2: Manual two-stage
# Stage 1: Get candidates from IVFPQ
index_ivfpq.nprobe = 32
_, candidates = index_ivfpq.search(queries, k=100)  # Over-retrieve

# Stage 2: Rescore candidates with exact distance
# Load full-precision vectors for candidates
for i, query in enumerate(queries):
    cand_ids = candidates[i][candidates[i] >= 0]
    cand_vectors = vectors[cand_ids]  # Full-precision vectors
    exact_dists = np.linalg.norm(cand_vectors - query, axis=1)
    top_k_local = np.argsort(exact_dists)[:10]
    final_ids = cand_ids[top_k_local]
```

---

## HNSW as Coarse Quantizer

Instead of using a flat index (IndexFlatL2) for the IVF coarse quantizer, you can use HNSW for faster centroid lookup:

```python
import faiss

d = 1536
nlist = 16384  # Large nlist for big datasets
m = 48

# HNSW coarse quantizer (fast centroid lookup)
quantizer = faiss.IndexHNSWFlat(d, 32)
quantizer.hnsw.efSearch = 64

# IVF+PQ with HNSW coarse quantizer
index = faiss.IndexIVFPQ(quantizer, d, nlist, m, 8)
index.quantizer_trains_alone = 2  # Special training mode for HNSW quantizer

# Train
index.train(vectors[:200_000])
index.add(vectors)

# Search -- centroid lookup is O(log(nlist)) instead of O(nlist)
index.nprobe = 64
distances, indices = index.search(queries, k=10)
```

This is important for large nlist values (16K+), where flat centroid search becomes a bottleneck.

---

## Comparison: HNSW vs IVF+PQ

### Memory Usage at Scale

| Vectors | Dimensions | HNSW (m=16, float32) | IVFPQ (m=48) | Savings |
|---|---|---|---|---|
| 1M | 1536 | 6.2 GB | 0.15 GB | 41x |
| 10M | 1536 | 62 GB | 1.5 GB | 41x |
| 100M | 1536 | 620 GB | 15 GB | 41x |
| 1B | 1536 | 6,200 GB | 150 GB | 41x |

### Quality Comparison

At 10M vectors, 1536d:

| Algorithm | Recall@10 | QPS (1 thread) | Memory |
|---|---|---|---|
| HNSW (ef=200) | 0.994 | 60 | 62 GB |
| IVFFlat (nprobe=64) | 0.965 | 40 | 62 GB |
| IVFPQ (nprobe=64) | 0.920 | 150 | 1.5 GB |
| IVFPQ + rescore 100 | 0.975 | 80 | 1.5 GB + disk |
| IVFPQ + rescore 200 | 0.985 | 50 | 1.5 GB + disk |
| OPQ + IVFPQ + rescore 100 | 0.980 | 75 | 1.5 GB + disk |

### When to Use Which

| Criterion | HNSW | IVF+PQ |
|---|---|---|
| Dataset fits in RAM | Best choice | Unnecessary compression |
| Dataset exceeds RAM | Impractical | Essential |
| Need incremental inserts | Good (O(log N) per insert) | Bad (requires retraining for new data) |
| Need frequent rebuilds | Slow (hours at 100M) | Fast (minutes at 100M) |
| Need maximum recall | Best (0.99+) | Good with rescoring (0.98+) |
| Memory budget is fixed | Limited by RAM | Scale by adjusting m and nlist |
| Query latency priority | Lower at small scale | Lower at large scale (smaller index) |

### Decision Tree

```
Can the full HNSW index fit in RAM?
  |
  +-- YES: Use HNSW
  |    |
  |    +-- Need to reduce memory further?
  |         +-- YES: HNSW + scalar quantization (int8)
  |         +-- NO: HNSW (float32)
  |
  +-- NO: Use IVF+PQ
       |
       +-- Need incremental updates?
       |    +-- YES: Consider DiskANN (graph + PQ + SSD)
       |    +-- NO: Standard IVFPQ
       |
       +-- Need maximum recall?
            +-- YES: IVFPQ + rescoring
            +-- NO: IVFPQ alone
```

---

## DiskANN: Bridging HNSW and IVF+PQ

### Overview

DiskANN (Microsoft, 2019) combines the benefits of both approaches: a graph index (like HNSW) stored on SSD with PQ codes in RAM for fast distance approximation.

```
DiskANN Architecture:

RAM:
  - PQ codes for all vectors (small, ~48 bytes each)
  - Graph metadata (node degrees, offsets)

SSD:
  - Full-precision vectors
  - Graph adjacency lists

Search:
  1. Start from entry point (in RAM)
  2. Navigate graph using PQ distance approximation (RAM)
  3. For top candidates, load full vectors from SSD for exact scoring
  4. Return top-k
```

### Key Properties

| Property | DiskANN | HNSW | IVFPQ |
|---|---|---|---|
| RAM usage | Low (PQ codes only) | High (full vectors + graph) | Low (PQ codes only) |
| Disk usage | High (full vectors + graph) | None (all in RAM) | Low (PQ codes only) |
| SSD required | Yes (NVMe recommended) | No | No |
| Build time | Moderate | Slow | Fast |
| Incremental inserts | Limited | Good | Bad |
| Recall at 100M | 0.98+ | 0.99+ (if fits in RAM) | 0.92 (without rescore) |
| Latency | 1-5 ms | 0.5-2 ms (in RAM) | 2-10 ms |

### DiskANN in Practice

```python
# DiskANN is available in:
# - Microsoft's open-source DiskANN library (C++)
# - Vamana index in Apache Lucene 9.7+
# - Azure AI Search
# - LanceDB (via Lance format)

# Python wrapper (diskannpy)
import diskannpy

# Build index
diskannpy.build_disk_index(
    data="vectors.bin",              # float32 binary file
    distance_metric="l2",
    index_directory="disk_index/",
    graph_degree=64,                 # Similar to HNSW M
    search_list_size=100,            # Similar to ef_construction
    num_pq_chunks=48,                # PQ subquantizers
    pq_dims=1536,
    max_points=10_000_000
)

# Search
index = diskannpy.DiskANNIndex(
    distance_metric="l2",
    index_directory="disk_index/",
    num_threads=8,
    cache_size=4_000_000_000  # 4 GB cache for SSD reads
)

ids, distances = index.batch_search(
    queries,
    k_neighbors=10,
    complexity=100  # Similar to ef_search
)
```

### DiskANN Performance

At 100M vectors, 768d, NVMe SSD:

| Configuration | Recall@10 | p50 Latency | QPS (8 threads) | RAM Usage |
|---|---|---|---|---|
| DiskANN (default) | 0.980 | 2.5 ms | 2,000 | 8 GB |
| DiskANN (high quality) | 0.995 | 5.0 ms | 1,000 | 8 GB |
| HNSW (if it fits) | 0.995 | 1.5 ms | 1,500 | 65 GB |
| IVFPQ (nprobe=64) | 0.920 | 3.0 ms | 3,000 | 5 GB |
| IVFPQ + rescore 200 | 0.985 | 8.0 ms | 800 | 5 GB + disk |

DiskANN achieves HNSW-like recall with IVF+PQ-like memory usage. The tradeoff is SSD dependency and higher latency than pure in-RAM HNSW.

---

## Advanced IVFPQ Configurations

### Multi-Probe LSH-like IVF

```python
import faiss

# IVF with many lists and high nprobe for better recall
d = 1536
nlist = 16384
m = 96  # More PQ subquantizers for better quality

quantizer = faiss.IndexFlatL2(d)
index = faiss.IndexIVFPQ(quantizer, d, nlist, m, 8)

# Add PQ refinement for better distance approximation
index.polysemous_ht = 0  # Disable polysemous for standard PQ

index.train(vectors[:200_000])
index.add(vectors)

# High nprobe for high recall
index.nprobe = 128  # Search 128 of 16384 lists (~0.8%)
```

### IVFPQ with Pre-Transform (OPQ)

```python
import faiss

d = 1536
m = 48

# OPQ rotation + IVFPQ
opq = faiss.OPQMatrix(d, m)
quantizer = faiss.IndexFlatL2(d)
sub_index = faiss.IndexIVFPQ(quantizer, d, 4096, m, 8)

index = faiss.IndexPreTransform(opq, sub_index)

# Train everything together
index.train(vectors[:200_000])
index.add(vectors)

sub_index.nprobe = 32
distances, indices = index.search(queries, k=10)
```

### GPU-Accelerated IVFPQ

```python
import faiss

d = 1536
nlist = 4096
m = 48

# Build on GPU for fast training
res = faiss.StandardGpuResources()

# Create CPU index
cpu_index = faiss.IndexIVFPQ(faiss.IndexFlatL2(d), d, nlist, m, 8)

# Move to GPU for training
gpu_index = faiss.index_cpu_to_gpu(res, 0, cpu_index)
gpu_index.train(vectors[:200_000])

# Add on GPU (fast)
gpu_index.add(vectors)

# Move back to CPU for serving (if GPU memory is needed elsewhere)
cpu_index = faiss.index_gpu_to_cpu(gpu_index)

cpu_index.nprobe = 32
distances, indices = cpu_index.search(queries, k=10)
```

---

## Training Data Requirements

### Minimum Training Set Size

| nlist | Minimum Training Vectors | Recommended |
|---|---|---|
| 256 | 8,000 | 25,000 |
| 1024 | 30,000 | 100,000 |
| 4096 | 120,000 | 400,000 |
| 16384 | 500,000 | 1,600,000 |

For PQ codebook training, the minimum is ~256 * (d/m) = ~8,000 vectors for 1536d with m=48. In practice, 100K-200K training vectors give good codebooks.

### Training on a Subset

```python
# For very large datasets, train on a random subset
n_train = min(200_000, len(vectors))
train_indices = np.random.choice(len(vectors), n_train, replace=False)
train_data = vectors[train_indices]

index.train(train_data)
index.add(vectors)  # Add all vectors after training
```

---

## Common Pitfalls

1. **Training IVF+PQ on the full dataset**: training only needs a representative subset (100K-200K vectors). Training on 100M vectors wastes time with no quality improvement.

2. **Setting nprobe too low**: nprobe=1 gives ~30% recall. This is useless for most applications. Start with nprobe=32-64 and tune based on your recall target.

3. **Using too few PQ subquantizers**: m=8 with 1536-dim vectors means each subspace has 192 dimensions, which is too many for 256 centroids to represent well. Use at least m=32 for 1536-dim vectors.

4. **Forgetting the residual trick**: always use IVF+PQ together (faiss.IndexIVFPQ), not IVF + PQ separately. The residual encoding (PQ applied to vector - centroid) is critical for quality.

5. **Not rescoring for quality-sensitive applications**: IVFPQ alone gives ~0.92 recall. With rescoring top-100-200 candidates against full-precision vectors, recall jumps to 0.98+. This is nearly free if vectors are on SSD.

6. **Rebuilding the index for every data update**: IVFPQ cannot handle incremental inserts well (new vectors may not map well to existing centroids). If your data changes frequently, consider HNSW or DiskANN instead.

7. **Not trying OPQ**: OPQ (Optimized Product Quantization) adds a learned rotation before PQ encoding. It improves recall by 1-3% at zero additional storage cost and minimal additional training time.

---

## References

- Product Quantization for Nearest Neighbor Search: Jegou et al., IEEE TPAMI, 2011
- DiskANN: Subramanya et al., NeurIPS, 2019
- Faiss documentation: https://github.com/facebookresearch/faiss/wiki
- Faiss IndexIVFPQ: https://github.com/facebookresearch/faiss/wiki/Faiss-indexes
- OPQ: Ge et al., "Optimized Product Quantization," IEEE TPAMI, 2014
- DiskANN library: https://github.com/microsoft/DiskANN
- diskannpy: https://github.com/microsoft/DiskANN/tree/main/python
- ScaNN: https://github.com/google-research/google-research/tree/master/scann
