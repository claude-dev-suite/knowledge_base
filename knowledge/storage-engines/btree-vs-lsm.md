# B-tree vs LSM-tree Storage Engines

## Overview

Underneath every database is a storage engine that decides how data is laid out
on disk and how reads, writes, and durability are handled. The dominant choice
is between B-trees (update-in-place) and log-structured merge-trees
(append-and-merge). It is driven by the read/write mix and the storage medium.

## B-tree

Used by InnoDB, PostgreSQL, and most relational engines. Data lives in
fixed-size pages forming a balanced tree; updates modify pages **in place**.

- **Reads:** one predictable path down the tree; excellent for point lookups and
  range scans.
- **Writes:** random I/O and write amplification (a small change rewrites a whole
  page, plus WAL); fragmentation over time.
- Best for read-heavy workloads with rich queries and range scans.

## LSM-tree

Used by RocksDB, Cassandra, ScyllaDB, LevelDB. Writes go to an in-memory
memtable, are flushed sequentially to immutable sorted files (SSTables), and are
later merged by **compaction**.

- **Writes:** sequential and fast — ideal for high ingest and SSDs.
- **Reads:** may consult several levels (read amplification); mitigated by Bloom
  filters and block caches.
- **Space:** stale versions linger until compaction (space amplification);
  compaction itself consumes I/O and CPU.

## The three amplifications

Every engine trades **write**, **read**, and **space** amplification against each
other. State which one the workload can least afford:

- Write-heavy ingest / time-series → minimize write-amp → LSM.
- Read-heavy, range-scan, mixed queries → minimize read-amp → B-tree.

## Durability and concurrency (shared concerns)

- **WAL + fsync policy** is the durability/throughput knob; group commit
  amortizes fsync across transactions.
- **Buffer/page cache** hit ratio dominates real performance; checkpointing
  bounds crash-recovery time.
- **MVCC** gives lock-free snapshot reads but needs version GC/vacuum (Postgres
  bloat); the alternative is locking (2PL) or optimistic validation (OCC).

## Guidance

Write-heavy, SSD, time-series → LSM (RocksDB/Cassandra). Read-heavy, relational,
range scans → B-tree (Postgres/InnoDB). Embedded/edge KV → LMDB (B-tree, mmap) or
RocksDB (LSM) depending on the write ratio.
