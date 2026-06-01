# Designing for the Memory Hierarchy (Mechanical Sympathy)

## Overview

When a system is bound by the machine rather than by I/O or business logic, the
memory hierarchy — not algorithmic big-O constants — often decides performance.
Designing software that matches how the hardware actually works is "mechanical
sympathy."

## The latency cliff

Approximate access latencies:

```
L1 cache     ~1 ns
L2 cache     ~4 ns
L3 cache    ~15 ns
DRAM       ~100 ns      <- a cache miss to DRAM is ~100x an L1 hit
NVMe SSD  ~50-100 us
network    ms
```

A DRAM miss costs roughly a hundred L1 hits. On hot paths, **data layout and
access pattern frequently matter more than the algorithm's constant factors.**

## Levers

- **Cache lines (64 B):** keep hot fields together; avoid pointer-chasing
  (linked lists and trees thrash the cache). Prefer contiguous arrays.
- **False sharing:** two cores writing different variables that share a cache
  line ping-pong the line between caches. Pad/align per-core data to a line.
- **Data-oriented design (SoA vs AoS):** struct-of-arrays streams well and
  enables SIMD; array-of-structs wastes bandwidth when you touch one field.
- **Predictable access:** sequential, predictable patterns let hardware
  prefetchers and branch predictors win; random access defeats them.
- **SIMD/vectorization** (AVX/NEON/SVE): data-parallel hot loops, needing
  aligned, contiguous, branch-light data.
- **NUMA:** place memory near the core that uses it; cross-socket access is much
  slower. Partition data per node rather than sharing one global structure.
- **Concurrency cost:** shared writes generate cache-coherence (MESI) traffic.
  Lock-free/RCU avoids contention but is hard and ABA-prone; sharded or per-core
  state often beats clever lock-free code.

## When this applies

Microsecond latency budgets, high-throughput data processing, database and
storage-engine internals, codecs, trading, and simulation. It is **not** the
right lens for ordinary CRUD/web services, where I/O and architecture-level
concerns dominate. Always identify the bottleneck resource first, then apply the
matching technique, and measure rather than micro-optimize on intuition.
