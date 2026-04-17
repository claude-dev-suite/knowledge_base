# ANN Algorithms Deep Guide

## Overview

Approximate Nearest Neighbor (ANN) search is the foundation of every vector database. Exact nearest neighbor search requires comparing a query vector against every vector in the dataset, which is O(N * d) where N is the number of vectors and d is the dimensionality. At 1 million vectors with 1536 dimensions, exact search takes ~50ms on modern hardware. At 100 million vectors, it takes ~5 seconds per query. ANN algorithms trade a small amount of accuracy (recall) for orders-of-magnitude speedup, typically achieving >95% recall at 100-1000x the throughput of brute force. This guide covers the ANN landscape, key metrics, algorithm taxonomy, and guidance on when to use which approach.

For deep dives into specific algorithms, see the companion articles on HNSW and IVF+PQ.

---

## Why ANN Is Needed at Scale

### Brute Force Scaling

| Vectors | Dimensions | Brute Force Latency (single core) | Brute Force Latency (8 cores) |
|---|---|---|---|
| 10K | 1536 | 0.5 ms | 0.1 ms |
| 100K | 1536 | 5 ms | 0.7 ms |
| 1M | 1536 | 50 ms | 7 ms |
| 10M | 1536 | 500 ms | 65 ms |
| 100M | 1536 | 5,000 ms | 650 ms |
| 1B | 1536 | 50,000 ms | 6,500 ms |

At 10M+ vectors, brute force is impractical for interactive applications. ANN algorithms reduce latency to single-digit milliseconds regardless of dataset size.

### The ANN Tradeoff

Every ANN algorithm makes the same fundamental tradeoff:

```
Exact Search:   100% recall,  O(N) latency,  O(N*d) memory
ANN Search:     <100% recall, O(log N) latency, O(N*d + index) memory

The goal: minimize recall loss while maximizing throughput.
```

---

## Key Metrics

### Recall@K

The fraction of true K nearest neighbors that the ANN algorithm returns.

```
Recall@10 = |ANN_top10 intersection Exact_top10| / 10
```

| Recall@10 | Quality | Typical Use Case |
|---|---|---|
| 0.99+ | Excellent | Production RAG, recommendations |
| 0.95-0.99 | Good | Most applications |
| 0.90-0.95 | Acceptable | High-throughput, latency-sensitive |
| < 0.90 | Poor | Only for pre-filtering before reranking |

### Queries Per Second (QPS)

Throughput at a given recall level. The standard benchmark metric plots recall vs QPS.

```python
import time
import numpy as np


def benchmark_qps(index, queries: np.ndarray, k: int = 10, duration: float = 30.0):
    """Measure QPS for an ANN index."""
    num_queries = 0
    start = time.perf_counter()

    while time.perf_counter() - start < duration:
        query = queries[num_queries % len(queries)]
        _ = index.search(query.reshape(1, -1), k)
        num_queries += 1

    elapsed = time.perf_counter() - start
    return num_queries / elapsed


def benchmark_recall(
    index, queries: np.ndarray, ground_truth: np.ndarray, k: int = 10
):
    """Measure recall@K for an ANN index."""
    total_recall = 0.0
    for i, query in enumerate(queries):
        results = index.search(query.reshape(1, -1), k)
        predicted = set(results[1][0])  # indices
        actual = set(ground_truth[i][:k])
        total_recall += len(predicted & actual) / k

    return total_recall / len(queries)
```

### Memory Usage

Total bytes consumed by the index structure, not including the raw vectors.

```
Index Memory = algorithm-specific overhead per vector * N

HNSW:   ~(M * 2 * 4 + 48) bytes/vector  (graph links + metadata)
IVF:    ~(nlist * d * 4 + N * 4) bytes   (centroids + inverted list pointers)
PQ:     ~(m * code_size) bytes/vector     (compressed codes)
```

### Build Time

Time to construct the index from raw vectors. Important for:
- Initial index creation
- Rebuilding after data changes
- CI/CD pipeline time for index updates

| Algorithm | Build Complexity | 1M vectors (1536d) | 10M vectors (1536d) |
|---|---|---|---|
| Flat (brute force) | O(1) | 0 sec | 0 sec |
| IVFFlat | O(N * nlist_iters) | 30 sec | 5 min |
| IVFPQ | O(N * nlist_iters + N * pq_train) | 2 min | 20 min |
| HNSW | O(N * log(N)) | 5 min | 1.5 hours |

---

## Taxonomy of ANN Algorithms

### Family Tree

```
ANN Algorithms
|
+-- Tree-Based
|   +-- KD-Tree (low dimensions, d < 20)
|   +-- Ball Tree (moderate dimensions)
|   +-- Annoy (random projection trees, Spotify)
|
+-- Hash-Based
|   +-- LSH (Locality-Sensitive Hashing)
|   +-- Multi-Probe LSH
|   +-- Cross-Polytope LSH
|
+-- Graph-Based
|   +-- NSW (Navigable Small World)
|   +-- HNSW (Hierarchical NSW) *** dominant algorithm ***
|   +-- NSG (Navigable Spreading-out Graph)
|   +-- DiskANN (graph + SSD, Microsoft)
|   +-- CAGRA (GPU-accelerated graph, NVIDIA)
|
+-- Quantization-Based
|   +-- IVF (Inverted File, coarse quantizer)
|   +-- PQ (Product Quantization, compression)
|   +-- OPQ (Optimized PQ, rotation)
|   +-- IVF+PQ (combined)
|   +-- ScaNN (anisotropic quantization, Google)
|   +-- RaBitQ (random bit quantization)
|
+-- Hybrid
    +-- IVF+HNSW (Faiss: IVF with HNSW coarse quantizer)
    +-- DiskANN (graph + PQ + SSD)
    +-- SPANN (inverted index + graph postings)
```

### Algorithm Selection Guide

| Scenario | Best Algorithm | Why |
|---|---|---|
| < 100K vectors, any dimension | Flat (brute force) | Fast enough, 100% recall |
| 100K - 10M vectors, RAM available | HNSW | Best recall/latency tradeoff |
| 10M - 100M vectors, RAM constrained | IVF+PQ | 10-50x memory reduction |
| 100M - 1B vectors, SSD available | DiskANN | Graph on SSD, PQ in RAM |
| GPU available, low latency needed | CAGRA | GPU-accelerated graph search |
| Streaming / real-time updates | HNSW | Supports incremental inserts |
| Batch-only, rebuild acceptable | IVF+PQ | Faster build, lower memory |
| Very low memory (mobile, edge) | PQ only | Extreme compression |

---

## Algorithm Comparison at Scale

### 1M Vectors, 1536 Dimensions, float32

| Algorithm | Recall@10 | QPS (1 thread) | QPS (8 threads) | Index Memory | Build Time |
|---|---|---|---|---|---|
| Flat (brute force) | 1.000 | 20 | 150 | 0 MB | 0 sec |
| HNSW (m=16, ef=128) | 0.993 | 500 | 3,000 | 128 MB | 5 min |
| IVFFlat (nlist=1000, nprobe=32) | 0.970 | 300 | 2,000 | 6 MB | 30 sec |
| IVFPQ (nlist=1000, m=48) | 0.920 | 800 | 5,000 | 52 MB | 2 min |
| IVFPQ + rerank | 0.975 | 400 | 2,500 | 52 MB | 2 min |
| ScaNN | 0.980 | 1,200 | 7,000 | 60 MB | 3 min |

### 10M Vectors, 1536 Dimensions, float32

| Algorithm | Recall@10 | QPS (1 thread) | Index Memory | Raw Vector Memory |
|---|---|---|---|---|
| HNSW (m=16, ef=200) | 0.990 | 80 | 1.3 GB | 57.2 GB |
| HNSW (m=32, ef=200) | 0.995 | 50 | 2.5 GB | 57.2 GB |
| IVFFlat (nlist=3162) | 0.960 | 60 | 20 MB | 57.2 GB |
| IVFPQ (m=48, nlist=3162) | 0.910 | 200 | 520 MB | 57.2 GB |
| IVFPQ + rerank 100 | 0.965 | 120 | 520 MB | 57.2 GB |
| DiskANN | 0.985 | 150 | 500 MB + SSD | 57.2 GB |

### 100M Vectors, 768 Dimensions

| Algorithm | Recall@10 | QPS (1 thread) | Total Memory |
|---|---|---|---|
| HNSW (m=32) | 0.992 | 15 | 340 GB |
| IVFPQ (m=96, nlist=10000) | 0.900 | 100 | 12 GB |
| IVFPQ + rerank 200 | 0.960 | 50 | 12 GB + disk |
| DiskANN | 0.980 | 80 | 10 GB + SSD |

---

## Brute Force: The Baseline

### Faiss IndexFlatL2

```python
import faiss
import numpy as np

d = 1536  # dimension
n = 1_000_000  # number of vectors

# Generate random data (replace with real embeddings)
vectors = np.random.rand(n, d).astype(np.float32)
queries = np.random.rand(100, d).astype(np.float32)

# Exact search (brute force)
index = faiss.IndexFlatL2(d)
index.add(vectors)

# Search
distances, indices = index.search(queries, k=10)

# For cosine similarity, normalize vectors first
faiss.normalize_L2(vectors)
faiss.normalize_L2(queries)
index_ip = faiss.IndexFlatIP(d)
index_ip.add(vectors)
distances, indices = index_ip.search(queries, k=10)
# distances are cosine similarities (higher = more similar)
```

### When to Use Brute Force

- **< 50K vectors**: search latency is under 5ms, which is fast enough for most applications.
- **As ground truth**: always benchmark ANN recall against brute force results.
- **As a reranking stage**: search compressed index, then rerank top-K with exact distances.
- **During development**: start with brute force, add ANN only when needed.

---

## Tree-Based Methods

### Annoy (Approximate Nearest Neighbors Oh Yeah)

Built by Spotify. Uses random projection trees. Good for static datasets that fit in memory.

```python
from annoy import AnnoyIndex
import numpy as np

d = 768
n_trees = 100  # More trees = better recall, more memory

# Build index
index = AnnoyIndex(d, 'angular')  # 'angular', 'euclidean', 'manhattan', 'dot'
for i in range(n_vectors):
    index.add_item(i, vectors[i])

index.build(n_trees)
index.save('index.ann')

# Load and search
index = AnnoyIndex(d, 'angular')
index.load('index.ann')

indices, distances = index.get_nns_by_vector(
    query_vector,
    n=10,
    search_k=1000,  # Higher = better recall, slower
    include_distances=True
)
```

**Properties**:
- Immutable after build (no incremental inserts)
- Memory-mapped (mmap), so multiple processes can share the index
- Lower recall than HNSW at the same latency
- Good for static datasets on resource-constrained systems

---

## Hash-Based Methods

### Locality-Sensitive Hashing (LSH)

LSH hashes similar vectors to the same bucket with high probability.

```python
import faiss
import numpy as np

d = 1536
n_bits = 128  # Number of hash bits

# LSH index
index = faiss.IndexLSH(d, n_bits)
index.add(vectors)

distances, indices = index.search(queries, k=10)
```

**Properties**:
- O(1) search time per hash table lookup
- Low recall without many hash tables
- High memory for many hash tables
- Largely superseded by HNSW and IVF in practice

---

## The Dominant Algorithm: HNSW

HNSW (Hierarchical Navigable Small World) is the most widely used ANN algorithm in production systems. It is used by Qdrant, Weaviate, ChromaDB, pgvector, OpenSearch (Lucene), Milvus, and many others.

See the companion article `hnsw-deep.md` for a detailed algorithmic deep dive.

### Why HNSW Dominates

1. **Best recall/latency tradeoff**: at any given latency target, HNSW typically achieves the highest recall.
2. **Supports incremental inserts**: new vectors can be added without rebuilding the index.
3. **No training step**: unlike IVF, HNSW does not require a clustering step on the data.
4. **Proven at scale**: used in production by all major vector databases.
5. **Simple to tune**: only 3 parameters (M, ef_construction, ef_search).

### Quick HNSW in Faiss

```python
import faiss
import numpy as np

d = 1536
M = 32           # Connections per node
ef_construction = 128  # Build-time beam width

# Create HNSW index
index = faiss.IndexHNSWFlat(d, M)
index.hnsw.efConstruction = ef_construction
index.hnsw.efSearch = 100  # Query-time beam width

# Add vectors
index.add(vectors)

# Search
distances, indices = index.search(queries, k=10)
```

---

## Quantization-Based Methods

### IVF + PQ: The Memory Champion

IVF (Inverted File) + PQ (Product Quantization) reduces memory by 10-50x while maintaining reasonable recall. It is the go-to algorithm for billion-scale datasets that do not fit in RAM.

See the companion article `ivf-pq-deep.md` for a detailed algorithmic deep dive.

### Quick IVFPQ in Faiss

```python
import faiss
import numpy as np

d = 1536
nlist = 4096    # Number of IVF clusters
m = 48          # Number of PQ subquantizers (must divide d)
n_bits = 8      # Bits per subquantizer (256 centroids per subspace)

# Create index
quantizer = faiss.IndexFlatL2(d)  # Coarse quantizer
index = faiss.IndexIVFPQ(quantizer, d, nlist, m, n_bits)

# Train on a representative subset
train_data = vectors[:100_000]  # At least 30 * nlist vectors
index.train(train_data)

# Add vectors
index.add(vectors)

# Search
index.nprobe = 32  # Number of clusters to search
distances, indices = index.search(queries, k=10)
```

### ScaNN (Google)

```python
import scann
import numpy as np

searcher = scann.scann_ops_pybind.builder(
    vectors, num_neighbors=10, distance_measure="dot_product"
).tree(
    num_leaves=2000,
    num_leaves_to_search=100,
    training_sample_size=250000
).score_ah(
    dimensions_per_block=2,
    anisotropic_quantization_threshold=0.2
).reorder(
    reordering_num_neighbors=100
).build()

neighbors, distances = searcher.search(query_vector)
```

---

## GPU-Accelerated Methods

### CAGRA (NVIDIA RAPIDS)

```python
import cuml
from cuml.neighbors import CAGRA

# Build on GPU
index = CAGRA(
    n_neighbors=64,  # Graph degree
    intermediate_graph_degree=128,
    graph_build_algo="ivf_pq"
)
index.fit(vectors_gpu)  # cupy array

# Search on GPU
distances, indices = index.search(queries_gpu, k=10)
```

**Properties**:
- 10-100x faster build than CPU HNSW
- Sub-millisecond search latency
- Requires NVIDIA GPU with sufficient VRAM
- Available in RAPIDS cuML and Milvus

---

## Choosing an Algorithm: Decision Tree

```
Start
  |
  +-- Dataset size < 50K?
  |     YES --> Use brute force (IndexFlatL2/IP)
  |
  +-- Need incremental inserts?
  |     YES --> HNSW
  |     |
  |     +-- Fits in RAM?
  |           YES --> HNSW (pure)
  |           NO  --> HNSW + quantization (SQ/PQ)
  |
  +-- Batch-only, no updates?
  |     YES --> IVF-based
  |     |
  |     +-- Fits in RAM?
  |     |     YES --> IVFFlat
  |     |     NO  --> IVFPQ
  |     |
  |     +-- SSD available?
  |           YES --> DiskANN
  |
  +-- GPU available?
        YES --> CAGRA (build) + HNSW or IVF (serve)
```

---

## Common Pitfalls

1. **Benchmarking at the wrong scale**: ANN algorithm rankings change with scale. HNSW dominates at 1M vectors but IVF+PQ may be necessary at 100M. Always benchmark at your target scale.

2. **Ignoring build time**: HNSW build at 100M vectors takes hours. If you need frequent full rebuilds, consider IVF (minutes) or streaming-capable algorithms.

3. **Not computing ground truth**: without brute force ground truth, you cannot measure recall. Always compute exact K-NN on a representative query set.

4. **Using default parameters**: every ANN algorithm has tunable parameters. Default parameters optimize for a general case that may not match your data distribution or recall/latency requirements.

5. **Assuming recall is constant**: recall varies across queries. Some queries land near cluster boundaries (IVF) or in sparse graph regions (HNSW) and have lower recall. Measure p95/p99 recall, not just average.

6. **Not accounting for filtered search**: ANN algorithms are designed for unfiltered search. Adding filters (multi-tenant, time range) can dramatically reduce effective recall. Test with realistic filter selectivity.

---

## References

- ann-benchmarks: https://ann-benchmarks.com/
- Faiss documentation: https://github.com/facebookresearch/faiss/wiki
- HNSW paper: Malkov & Yashunin, 2018 (arXiv:1603.09320)
- Product Quantization paper: Jegou et al., 2011
- DiskANN paper: Subramanya et al., 2019 (NeurIPS)
- ScaNN paper: Guo et al., 2020 (ICML)
- CAGRA: https://docs.rapids.ai/api/cuvs/stable/
- Annoy: https://github.com/spotify/annoy
