# Kernel Bypass and the High-Performance Network Data Path

## Overview

The classic socket + kernel TCP/IP stack is convenient but spends most of its
budget on system calls, context switches, and copying data between user and
kernel space. At high packet rates these dominate. Kernel-bypass and related
techniques move the data path closer to (or onto) the NIC to reach line rate.

## The latency/throughput ladder

| Tier | Mechanism | Trade-off |
|------|-----------|-----------|
| Sockets + epoll | Classic kernel stack | Simplest; syscalls + copies cap throughput |
| io_uring | Async submission/completion rings shared with the kernel | Fewer syscalls/copies; keeps the kernel stack |
| eBPF / XDP | Run verified code at the driver hook, before the stack | Drop/redirect/rewrite at line rate; logic constrained by the verifier |
| Kernel bypass (DPDK) | Poll-mode userspace driver owns the NIC | 10–100M pps at µs latency; burns CPU cores, no kernel stack (you reimplement what you need) |

Pick the **lowest tier that meets the requirement** — bypass is powerful but
costs cores and operational complexity.

## The levers that matter

- **Zero-copy:** avoid user/kernel copies via `sendfile`/`splice`, io_uring
  fixed buffers, or DPDK mbufs. Copies are frequently the real bottleneck.
- **Per-core, shared-nothing:** thread-per-core with receive-side scaling (RSS)
  and CPU affinity (the Seastar model) removes cross-core locking and cache-line
  bouncing.
- **NIC offloads:** checksum, TSO/LRO, RSS, and increasingly crypto/TCP offload;
  SmartNICs/DPUs push the entire data plane off the host CPU.
- **Congestion control choice:** CUBIC for throughput, BBR for latency and
  bufferbloat, DCTCP inside the datacenter — it shapes tail latency.

## Guidance

A normal API server should use sockets/epoll or io_uring and not over-engineer.
Software routers, firewalls, and load balancers at line rate want XDP/eBPF or
DPDK. Microsecond-tail systems (trading, telco data planes) justify DPDK plus
per-core design and SmartNIC offload.
