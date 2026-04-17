# HNSW Algorithm Deep Dive

## Overview

Hierarchical Navigable Small World (HNSW) is the dominant approximate nearest neighbor algorithm used in production vector databases. Published by Malkov and Yashunin in 2016, HNSW builds a multi-layer graph where each layer is a navigable small world graph. The top layers provide long-range navigation (coarse search), and the bottom layer provides fine-grained precision. Search starts at the top layer and greedily descends, using each layer's result as the entry point for the next. This guide covers the algorithm internals, graph construction, search procedure, parameter effects, memory analysis, and practical tuning.

---

## Core Concepts

### Navigable Small World Property

A navigable small world graph has two key properties:
1. **Short-range connections**: nodes connect to nearby neighbors (high local clustering).
2. **Long-range connections**: a few connections span large distances (enabling O(log N) navigation).

This mirrors the "six degrees of separation" phenomenon: you can reach any person through roughly 6 social connections because the network has both local clusters and long-range bridges.

### Hierarchical Structure

HNSW adds hierarchy to the navigable small world concept:

```
Layer 3 (sparse):     A ------------- D
                      |               |
Layer 2:              A ---- B ------ D
                      |      |        |
Layer 1:              A -- B -- C --- D -- E
                      |    |    |    |    |
Layer 0 (dense):   A-B-C-D-E-F-G-H-I-J-K-L-M-...  (ALL vectors)
```

Key properties:
- **Layer 0 contains ALL vectors** and has the most connections (densest).
- **Higher layers contain exponentially fewer vectors** (sparser).
- **The top layer** contains very few vectors and provides coarse-grained navigation.
- **Each vector is assigned a maximum layer** using an exponential distribution.

---

## Graph Construction

### Layer Assignment

When a new vector is inserted, its maximum layer is determined by an exponential distribution:

```
layer = floor(-ln(random()) * mL)

where mL = 1 / ln(M)  (M is the max connections parameter)
```

For M=16, mL ~ 0.36:
- ~63% of vectors are on layer 0 only
- ~23% of vectors reach layer 1
- ~9% of vectors reach layer 2
- ~3% of vectors reach layer 3
- ~1% of vectors reach layer 4+

```python
import math
import random


def assign_layer(M: int) -> int:
    """Assign a layer to a new vector using exponential distribution."""
    mL = 1.0 / math.log(M)
    return int(-math.log(random.random()) * mL)


# Simulate layer distribution for 1M vectors
layer_counts = {}
for _ in range(1_000_000):
    layer = assign_layer(16)
    layer_counts[layer] = layer_counts.get(layer, 0) + 1

for layer in sorted(layer_counts.keys()):
    pct = layer_counts[layer] / 1_000_000 * 100
    print(f"Layer {layer}: {layer_counts[layer]:>7,} vectors ({pct:.1f}%)")

# Typical output:
# Layer 0: 639,211 vectors (63.9%)
# Layer 1: 230,456 vectors (23.0%)
# Layer 2:  83,127 vectors (8.3%)
# Layer 3:  30,052 vectors (3.0%)
# Layer 4:  10,891 vectors (1.1%)
# Layer 5:   3,943 vectors (0.4%)
# Layer 6:   1,427 vectors (0.1%)
# Layer 7:     530 vectors (0.1%)
# Layer 8:     188 vectors (0.0%)
# ...
```

### Insertion Algorithm

```python
def insert(graph, new_vector, M, ef_construction):
    """Insert a new vector into the HNSW graph."""
    # Step 1: Assign layer
    new_layer = assign_layer(M)

    # Step 2: Find entry point by searching from top layer down to new_layer+1
    entry_point = graph.top_entry_point
    current_layer = graph.max_layer

    # Greedy search on upper layers (only track 1 nearest neighbor)
    while current_layer > new_layer:
        entry_point = greedy_search(graph, new_vector, entry_point,
                                     current_layer, ef=1)
        current_layer -= 1

    # Step 3: Search and connect on layers new_layer down to 0
    for layer in range(min(new_layer, graph.max_layer), -1, -1):
        # Search for ef_construction nearest neighbors at this layer
        candidates = beam_search(graph, new_vector, entry_point,
                                  layer, ef=ef_construction)

        # Select M neighbors to connect (heuristic selection)
        neighbors = select_neighbors(new_vector, candidates, M, layer)

        # Create bidirectional connections
        for neighbor in neighbors:
            graph.add_edge(new_vector, neighbor, layer)
            graph.add_edge(neighbor, new_vector, layer)

            # If neighbor now has too many connections, prune
            if len(graph.get_neighbors(neighbor, layer)) > M:
                pruned = select_neighbors(neighbor,
                                          graph.get_neighbors(neighbor, layer),
                                          M, layer)
                graph.set_neighbors(neighbor, pruned, layer)

        entry_point = candidates[0]  # Nearest found at this layer

    # Update global entry point if new vector is on a higher layer
    if new_layer > graph.max_layer:
        graph.top_entry_point = new_vector
        graph.max_layer = new_layer
```

### Neighbor Selection Heuristic

The neighbor selection heuristic is critical for graph quality. The simple approach selects the M nearest candidates. The heuristic approach (used in practice) also considers diversity:

```python
def select_neighbors_heuristic(query, candidates, M, extend_candidates=False):
    """Select M neighbors using the heuristic from the HNSW paper.

    The heuristic prefers candidates that are close to the query AND
    not too close to each other, promoting graph connectivity.
    """
    if extend_candidates:
        # Extend candidate set with neighbors of candidates
        extended = set(candidates)
        for c in candidates:
            for neighbor in graph.get_neighbors(c):
                extended.add(neighbor)
        candidates = list(extended)

    # Sort candidates by distance to query
    candidates.sort(key=lambda c: distance(query, c))

    selected = []
    for candidate in candidates:
        if len(selected) >= M:
            break

        # Only add if this candidate is closer to query than to any
        # already-selected neighbor (promotes diversity)
        is_good = True
        for existing in selected:
            if distance(candidate, existing) < distance(candidate, query):
                is_good = False
                break

        if is_good:
            selected.append(candidate)

    return selected
```

---

## Search Algorithm

### Greedy Search (Single Layer)

```python
def greedy_search(graph, query, entry_point, layer, ef=1):
    """Greedy search on a single layer. Returns ef nearest neighbors."""
    visited = {entry_point}
    candidates = MinHeap()  # Sorted by distance (ascending)
    results = MaxHeap()     # Sorted by distance (descending), bounded to ef

    dist = distance(query, entry_point)
    candidates.push(dist, entry_point)
    results.push(dist, entry_point)

    while len(candidates) > 0:
        # Get closest unvisited candidate
        c_dist, closest = candidates.pop()

        # If closest candidate is farther than the farthest result, stop
        if c_dist > results.peek_distance():
            break

        # Explore neighbors of closest
        for neighbor in graph.get_neighbors(closest, layer):
            if neighbor in visited:
                continue
            visited.add(neighbor)

            n_dist = distance(query, neighbor)

            # Add to results if closer than current farthest result
            if n_dist < results.peek_distance() or len(results) < ef:
                candidates.push(n_dist, neighbor)
                results.push(n_dist, neighbor)

                # Trim results to ef
                if len(results) > ef:
                    results.pop_farthest()

    return results.get_sorted()  # Return ef nearest neighbors
```

### Multi-Layer Search (Full Algorithm)

```python
def search(graph, query, k, ef_search):
    """Full HNSW search across all layers."""
    # Start from the entry point at the top layer
    entry_point = graph.top_entry_point
    current_layer = graph.max_layer

    # Phase 1: Greedy descent (top layer down to layer 1)
    # Only track 1 nearest neighbor per layer
    while current_layer > 0:
        entry_point = greedy_search(graph, query, entry_point,
                                     current_layer, ef=1)[0]
        current_layer -= 1

    # Phase 2: Beam search on layer 0
    # Track ef_search candidates for quality
    results = greedy_search(graph, query, entry_point, layer=0, ef=ef_search)

    # Return top-k from the ef_search results
    return results[:k]
```

### Search Complexity

```
Search time = O(log(N)) * cost_per_layer

cost_per_layer ~ O(ef_search * M * d)
  where d = dimension (distance computation cost)

Total: O(log(N) * ef_search * M * d)
```

In practice, the log(N) factor comes from the number of layers (top-layer to layer-0 is ~log(N) / log(M) steps), and layer 0 dominates the search time.

---

## Parameter Effects

### M (Max Connections per Node)

M controls the maximum number of bidirectional connections each node has at each layer.

| M | Graph Properties | Tradeoff |
|---|---|---|
| 4-8 | Sparse graph, fast build, low memory | Lower recall, faster search |
| 12-16 | Balanced (default in most implementations) | Good recall/memory balance |
| 24-32 | Dense graph, slow build, high memory | Higher recall, slower search |
| 48-64 | Very dense, very slow build | Diminishing recall returns |

**Memory impact**: each connection uses 8 bytes (4 bytes for node ID + 4 bytes for next pointer). Memory per vector:

```
graph_memory_per_vector = M * 2 * 8 bytes  (bidirectional, ~16 bytes per connection)

M=16: 256 bytes per vector
M=32: 512 bytes per vector
M=64: 1024 bytes per vector
```

**On graph structure**: higher M means each node has more neighbors, which:
- Increases the chance of finding a good path to the query
- Reduces the number of hops needed (shorter paths)
- Makes the graph more robust to difficult query distributions

### ef_construction (Build-Time Beam Width)

ef_construction controls how many candidates are tracked during insertion.

| ef_construction | Build Quality | Build Time |
|---|---|---|
| 50-100 | Adequate for prototyping | Fast |
| 128-200 | Good for production | Moderate |
| 256-400 | High quality | Slow |
| 500+ | Diminishing returns | Very slow |

**Key insight**: ef_construction affects the quality of the graph permanently. Once built, a low-quality graph cannot be improved by increasing ef_search. If you need high recall, set ef_construction high during build.

```python
import faiss
import numpy as np

d = 768
M = 32

# Build time comparison (1M vectors)
for ef_c in [64, 128, 256, 512]:
    index = faiss.IndexHNSWFlat(d, M)
    index.hnsw.efConstruction = ef_c

    vectors = np.random.rand(1_000_000, d).astype(np.float32)

    import time
    start = time.perf_counter()
    index.add(vectors)
    elapsed = time.perf_counter() - start
    print(f"ef_construction={ef_c}: {elapsed:.0f}s")

# Typical output:
# ef_construction=64:  120s
# ef_construction=128: 200s
# ef_construction=256: 350s
# ef_construction=512: 620s
```

### ef_search (Query-Time Beam Width)

ef_search controls the accuracy-speed tradeoff at query time. It can be changed without rebuilding the index.

| ef_search | Recall@10 (typical) | Relative Latency |
|---|---|---|
| 10 | 0.85-0.90 | 1.0x |
| 20 | 0.90-0.94 | 1.3x |
| 40 | 0.94-0.97 | 1.8x |
| 80 | 0.97-0.99 | 2.8x |
| 100 | 0.98-0.99 | 3.2x |
| 200 | 0.99-0.999 | 5.5x |
| 400 | 0.999+ | 10x |

**ef_search must be >= k**: you cannot return more results than candidates tracked. Most implementations enforce ef_search >= k.

```python
import faiss
import numpy as np

d = 1536
M = 32
n = 1_000_000

vectors = np.random.rand(n, d).astype(np.float32)
queries = np.random.rand(100, d).astype(np.float32)

# Build index
index = faiss.IndexHNSWFlat(d, M)
index.hnsw.efConstruction = 200
index.add(vectors)

# Ground truth
flat_index = faiss.IndexFlatL2(d)
flat_index.add(vectors)
_, gt_indices = flat_index.search(queries, 10)

# Measure recall at different ef_search values
for ef in [10, 20, 40, 80, 100, 200, 400]:
    index.hnsw.efSearch = ef
    _, pred_indices = index.search(queries, 10)

    recall = np.mean([
        len(set(gt_indices[i]) & set(pred_indices[i])) / 10
        for i in range(len(queries))
    ])
    print(f"ef_search={ef:>3}: recall@10={recall:.4f}")
```

---

## Memory Analysis

### Total Memory Formula

```
Total HNSW Memory = Vector Storage + Graph Storage + Metadata

Vector Storage = N * D * bytes_per_element
  float32: N * D * 4
  float16: N * D * 2
  int8:    N * D * 1

Graph Storage = N * M_max * 2 * 4  (for layer 0)
              + N_upper * M * 2 * 4  (for upper layers, ~37% of N)

  where M_max = M * 2 for layer 0 (hnswlib convention)

Metadata = N * ~48 bytes (IDs, layer info, lock bytes)
```

### Practical Memory Estimates

| Vectors | Dimensions | Type | M | Vector Memory | Graph Memory | Total |
|---|---|---|---|---|---|---|
| 100K | 1536 | float32 | 16 | 0.57 GB | 0.05 GB | 0.62 GB |
| 1M | 1536 | float32 | 16 | 5.7 GB | 0.49 GB | 6.2 GB |
| 1M | 1536 | float16 | 16 | 2.9 GB | 0.49 GB | 3.4 GB |
| 1M | 768 | float32 | 16 | 2.9 GB | 0.49 GB | 3.4 GB |
| 5M | 1536 | float32 | 16 | 28.6 GB | 2.4 GB | 31 GB |
| 10M | 1536 | float32 | 16 | 57.2 GB | 4.9 GB | 62 GB |
| 10M | 1536 | float32 | 32 | 57.2 GB | 9.8 GB | 67 GB |
| 100M | 1536 | float32 | 16 | 572 GB | 49 GB | 621 GB |
| 100M | 768 | float16 | 16 | 143 GB | 49 GB | 192 GB |

**Key insight**: at high dimensions, vector storage dominates. At low dimensions or with quantization, graph storage becomes significant. Reducing M saves graph memory but hurts recall.

### Memory Estimation Code

```python
import math


def estimate_hnsw_memory(
    num_vectors: int,
    dimensions: int,
    M: int = 16,
    bytes_per_dim: int = 4,
    include_metadata: bool = True
) -> dict:
    """Estimate HNSW index memory usage."""
    # Vector storage
    vector_bytes = num_vectors * dimensions * bytes_per_dim

    # Graph storage (layer 0 has 2*M connections, upper layers have M)
    # Fraction on each upper layer: ~1/M of the layer below
    # Simplified: layer 0 = N vectors, upper layers ~ 0.37 * N total
    M_layer0 = M * 2  # hnswlib doubles M for layer 0
    graph_layer0 = num_vectors * M_layer0 * 2 * 4  # bidirectional, 4 bytes per link
    graph_upper = int(num_vectors * 0.37) * M * 2 * 4
    graph_bytes = graph_layer0 + graph_upper

    # Metadata
    metadata_bytes = num_vectors * 48 if include_metadata else 0

    total = vector_bytes + graph_bytes + metadata_bytes

    return {
        "vector_gb": vector_bytes / 1e9,
        "graph_gb": graph_bytes / 1e9,
        "metadata_gb": metadata_bytes / 1e9,
        "total_gb": total / 1e9,
        "bytes_per_vector": total / num_vectors
    }


# Example
result = estimate_hnsw_memory(10_000_000, 1536, M=16, bytes_per_dim=4)
for key, val in result.items():
    print(f"  {key}: {val:.2f}")
```

---

## Build Complexity

### Time Complexity

```
Build time = O(N * log(N) * ef_construction * M * d)

Breakdown:
  N:                number of vectors
  log(N):           average number of layers traversed per insert
  ef_construction:  beam width during search for neighbors
  M:                number of connections to create
  d:                distance computation cost (dimension-dependent)
```

### Parallelism

HNSW build is inherently sequential per-vector (each insertion modifies the graph). However, implementations achieve parallelism through:

1. **Concurrent insertions with locks**: hnswlib and Faiss use per-node locks, allowing multiple insertions on different parts of the graph simultaneously.
2. **Batch graph construction**: some implementations batch insertions at the same layer.

```python
# Faiss: control build parallelism via OpenMP
import faiss
import os

# Set number of threads for parallel build
os.environ["OMP_NUM_THREADS"] = "8"
faiss.omp_set_num_threads(8)

index = faiss.IndexHNSWFlat(1536, 32)
index.hnsw.efConstruction = 200
index.add(vectors)  # Uses 8 threads
```

### Build Time Benchmarks

| Vectors | Dimensions | M | ef_construction | 1 Thread | 8 Threads |
|---|---|---|---|---|---|
| 100K | 1536 | 16 | 128 | 20 sec | 5 sec |
| 1M | 1536 | 16 | 128 | 5 min | 1.5 min |
| 1M | 1536 | 32 | 200 | 12 min | 3 min |
| 5M | 1536 | 16 | 128 | 30 min | 10 min |
| 10M | 1536 | 16 | 200 | 1.5 hrs | 25 min |
| 10M | 1536 | 32 | 256 | 3.5 hrs | 55 min |

---

## Practical Tuning Guide

### Step 1: Choose M

```
Dataset size < 1M:    M = 16 (default)
Dataset size 1M-10M:  M = 24 (slightly better recall)
Dataset size > 10M:   M = 32 (if RAM allows)
Memory-constrained:   M = 12 (save ~25% graph memory)
```

### Step 2: Choose ef_construction

```
Rule of thumb: ef_construction = 2 * M for minimum quality
Production:    ef_construction = 128-256
One-time build (can afford hours): ef_construction = 400+
```

### Step 3: Tune ef_search (At Query Time)

```python
def find_optimal_ef_search(index, queries, ground_truth, target_recall=0.95):
    """Binary search for the minimum ef_search that achieves target recall."""
    low, high = 10, 500

    while low < high:
        mid = (low + high) // 2
        index.hnsw.efSearch = mid
        _, pred = index.search(queries, 10)

        recall = compute_recall(pred, ground_truth)
        if recall >= target_recall:
            high = mid
        else:
            low = mid + 1

    return low
```

### Step 4: Verify with Benchmarks

```python
def benchmark_hnsw(index, queries, ground_truth, k=10):
    """Full benchmark: recall, latency, and QPS at various ef_search values."""
    results = []
    for ef in [10, 20, 40, 60, 80, 100, 150, 200, 300, 400]:
        index.hnsw.efSearch = ef

        # Measure latency
        import time
        start = time.perf_counter()
        n_queries = 0
        while time.perf_counter() - start < 5.0:  # 5 second measurement
            _, pred = index.search(queries[n_queries % len(queries):n_queries % len(queries) + 1], k)
            n_queries += 1
        elapsed = time.perf_counter() - start
        qps = n_queries / elapsed
        latency_ms = elapsed / n_queries * 1000

        # Measure recall
        _, all_pred = index.search(queries, k)
        recall = compute_recall(all_pred, ground_truth)

        results.append({
            "ef_search": ef,
            "recall": recall,
            "qps": qps,
            "latency_ms": latency_ms
        })
        print(f"ef={ef:>3}: recall={recall:.4f}, QPS={qps:.0f}, latency={latency_ms:.1f}ms")

    return results


def compute_recall(predicted, ground_truth):
    """Compute recall@K."""
    total = 0
    for i in range(len(predicted)):
        total += len(set(predicted[i]) & set(ground_truth[i])) / len(ground_truth[i])
    return total / len(predicted)
```

---

## HNSW Variants

### HNSW + Scalar Quantization (SQ)

Store vectors as int8, search using int8 distances, but keep the HNSW graph at full precision.

```python
# Faiss: HNSW with scalar quantizer
import faiss

d = 1536
M = 32

# HNSW with Flat (float32) -- baseline
index_flat = faiss.IndexHNSWFlat(d, M)

# HNSW with Scalar Quantizer (int8) -- 4x memory reduction for vectors
sq = faiss.ScalarQuantizer(d, faiss.ScalarQuantizer.QT_8bit)
index_sq = faiss.IndexHNSWSQ(d, sq, M)
```

### HNSW + Product Quantization (PQ)

```python
# Faiss: HNSW with PQ -- aggressive compression
# Note: Faiss does not have IndexHNSWPQ directly, but you can use:
index = faiss.IndexHNSWFlat(d, M)  # Graph in float32
# For PQ, use IVF+PQ with HNSW as the coarse quantizer:
quantizer = faiss.IndexHNSWFlat(d, 32)
index_ivfpq = faiss.IndexIVFPQ(quantizer, d, 4096, 48, 8)
```

---

## Common Pitfalls

1. **Building with low ef_construction then expecting high recall**: the graph quality is determined at build time. If ef_construction=50, no amount of ef_search increase will recover the lost quality. Rebuild with higher ef_construction.

2. **Setting ef_search lower than k**: most implementations silently set ef_search = max(ef_search, k), but results will have poor recall if ef_search is too close to k.

3. **Ignoring incremental insert cost**: each insertion is O(log N * ef_construction * M * d). At 100M vectors, individual inserts become expensive. Batch inserts when possible.

4. **Not accounting for graph memory**: at 100M vectors with M=32, graph overhead alone is ~10 GB, even before counting vector storage.

5. **Using HNSW for billion-scale without quantization**: at 1B vectors with 1536d float32, HNSW requires ~6 TB of RAM. At this scale, IVF+PQ or DiskANN is required.

6. **Benchmarking with random data**: ANN algorithm performance depends on data distribution. Random uniform vectors (worst case for all algorithms) give pessimistic results. Benchmark with your actual embeddings.

---

## References

- HNSW paper: Malkov, Y.A., Yashunin, D.A. "Efficient and robust approximate nearest neighbor search using Hierarchical Navigable Small World graphs." IEEE TPAMI, 2018. arXiv:1603.09320
- hnswlib (C++/Python): https://github.com/nmslib/hnswlib
- Faiss HNSW implementation: https://github.com/facebookresearch/faiss
- ann-benchmarks HNSW results: https://ann-benchmarks.com/
- Vespa HNSW: https://docs.vespa.ai/en/approximate-nn-hnsw.html
