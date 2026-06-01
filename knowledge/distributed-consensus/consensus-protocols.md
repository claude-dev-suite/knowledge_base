# Consensus Protocols and Replication

## Overview

Consensus lets a set of nodes agree on a single value or an ordered log of
operations despite failures. It is the backbone of replicated state machines,
leader election, and strongly-consistent stores. The first question is the
**fault model**; the second is the **consistency** you actually need.

## Fault models and protocols

| Protocol | Tolerates | Nodes for f failures | Notes |
|----------|-----------|----------------------|-------|
| Raft | crash faults | 2f + 1 | Understandable, leader-based; etcd/Consul |
| (Multi-)Paxos | crash faults | 2f + 1 | Foundational, subtle; Chubby/Spanner lineage |
| Viewstamped Replication | crash faults | 2f + 1 | Raft-like, predates it |
| PBFT / Tendermint | Byzantine (malicious) | 3f + 1 | For untrusted nodes; higher message cost |

Use crash-fault consensus **inside** a trust boundary. Use BFT **only** when
nodes can be malicious (cross-organization, blockchain) — it needs more nodes
and more messages.

## Replication and quorums

- **Quorums:** with N replicas, choosing write and read quorums so that
  `W + R > N` guarantees a read sees the latest write.
- **Leader-based** (Raft): simple and strongly consistent, but the leader is a
  throughput bottleneck and failover has a window.
- **Leaderless** (Dynamo-style): highly available and partition-tolerant, but
  needs read-repair/anti-entropy and conflict resolution (e.g. CRDTs).
- **Sync vs async replication** trades durability/latency against the recovery
  point (RPO) on failover.

## Consistency models

Linearizable → sequential → causal → eventual. Stronger models require more
coordination and therefore cost latency and availability. Do not ask for
linearizable if causal consistency is sufficient.

## CAP / PACELC and clocks

Under a network **P**artition you must choose **C**onsistency or **A**vailability;
**E**lse (no partition) you still trade **L**atency against **C**onsistency. Real
systems are points on this spectrum (Spanner is CP and uses TrueTime's
bounded-uncertainty clocks for external consistency; Dynamo is AP). Avoid relying
on wall-clock ordering — use logical or hybrid logical clocks, or bounded-error
physical clocks where you need real-time ordering.
