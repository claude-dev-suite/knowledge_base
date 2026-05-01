# Fulcrum vs Electrs Comparison - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/infrastructure/fulcrum`.
> Canonical source: https://github.com/cculianu/Fulcrum
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/infrastructure/fulcrum/SKILL.md

## Concept

Fulcrum and electrs both implement the Electrum server protocol but make very
different engineering trade-offs. Fulcrum (C++/Qt, by Calin Culianu) prioritizes
query throughput and aggressive caching; electrs (Rust, by Roman Mandryk)
prioritizes a small footprint, simple code base, and low RAM. This article is
a head-to-head comparison covering wire protocol parity, sync time, memory
profile, query latency, and operational concerns - so you can pick the right
backend for a given deployment.

## Walkthrough / mechanics

Wire-protocol level both expose identical Electrum RPC methods over TCP/TLS:
`blockchain.scripthash.get_history`, `blockchain.scripthash.subscribe`,
`blockchain.transaction.get`, `mempool.get_fee_histogram`, etc. Sparrow,
Electrum desktop, BlueWallet, and BTCPay treat them as drop-in replacements.

Sync architecture:

- electrs reads block files directly from `~/.bitcoin/blocks/*` and
  cross-checks via RPC. Indexing is single-threaded per task with RocksDB
  storage. Mainnet sync ~24-48h on NVMe.
- Fulcrum talks to bitcoind purely over JSON-RPC, fetching blocks via
  `getblock` calls in parallel worker threads. RocksDB storage with a
  larger in-memory cache. Mainnet sync ~12-24h on NVMe.

Memory model:

- electrs holds minimal state in RAM; RocksDB block cache is small. Steady
  state ~1.5-2.5 GB.
- Fulcrum keeps a multi-GB index hot in RAM (configurable via `db_mem`,
  `fast-sync`). Steady state ~3-6 GB; sync peak 8+ GB.

Query latency (typical mainnet, well-funded scripthash, p50/p99):

| Operation | electrs | Fulcrum |
|-----------|---------|---------|
| scripthash.get_history (1k tx) | 80 / 350 ms | 30 / 90 ms |
| scripthash.get_balance | 20 / 80 ms | 8 / 25 ms |
| transaction.get (verbose) | 15 / 60 ms | 12 / 50 ms |
| estimate.fee | 5 / 15 ms | 4 / 12 ms |

## Worked example

Run an apples-to-apples benchmark from a wallet's perspective. Pick a busy
scripthash (e.g., a known exchange hot wallet) and time the response:

```bash
HASH=8b01df4e3df60e7fb5d0a5f6234a9c0...
echo "{\"jsonrpc\":\"2.0\",\"method\":\"blockchain.scripthash.get_history\",\"params\":[\"$HASH\"],\"id\":0}" \
  | time nc -q 2 127.0.0.1 50001 > /dev/null
```

Run the same against both servers and compare wall time. For latency-sensitive
applications (BTCPay store with many invoices, public-facing servers) the
4-10x speedup matters; for personal wallets connecting once per session the
difference is invisible.

Decision matrix:

| Scenario | Recommended |
|----------|-------------|
| Raspberry Pi 4 (4 GB), one user | electrs |
| Personal node, low traffic | electrs |
| Power user with several devices | either |
| Public Electrum server | Fulcrum |
| BTCPay store with 100+ invoices/day | Fulcrum |
| Resource-constrained (2 GB RAM VM) | electrs |
| Maximum query throughput | Fulcrum |
| Smallest disk footprint | electrs (~6-10 GB index vs 12-15 GB) |
| Tor-only node, infrequent queries | electrs |
| Audited supply chain (Rust ecosystem) | electrs |

## Common pitfalls

- Assuming Fulcrum is always faster - on a 2 GB VM Fulcrum will OOM-thrash
  while electrs stays responsive; "fastest" depends on RAM headroom.
- Migrating data - the RocksDB layouts are not compatible. Switching servers
  means a fresh re-index; do not try to copy the `db/` folder.
- Different mempool semantics - both report mempool entries but Fulcrum's
  `mempool.get_fee_histogram` is more granular; wallets depending on specific
  bucket counts may render slightly differently.
- TLS cert handling differs - electrs typically delegates to nginx/stunnel,
  Fulcrum has built-in TLS. Pick one path and document it.
- Reorg recovery - both handle shallow reorgs transparently; under deep reorgs
  (rare on mainnet) electrs has been observed to need a manual restart while
  Fulcrum auto-recovers via its block-fetch parallelism.

## References

- Fulcrum: https://github.com/cculianu/Fulcrum
- electrs: https://github.com/romanz/electrs
- Electrum protocol spec: https://electrumx.readthedocs.io/en/latest/protocol-methods.html
- Independent benchmark thread: https://github.com/cculianu/Fulcrum/discussions
