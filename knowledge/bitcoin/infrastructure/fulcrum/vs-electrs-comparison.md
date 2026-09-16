# Fulcrum vs Electrs Comparison - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/infrastructure/fulcrum`.
> Canonical source: https://github.com/cculianu/Fulcrum
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/infrastructure/fulcrum/SKILL.md

## Concept

Fulcrum and electrs both implement the Electrum server protocol but make very
different engineering trade-offs. Fulcrum (C++/Qt, by Calin Culianu) prioritizes
query throughput and aggressive caching; electrs (Rust, by Roman Zeyde)
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

- electrs no longer reads block files off disk - that was the pre-0.9.0
  architecture. 0.9.0 switched to the Bitcoin P2P protocol, and v0.12.0
  (13 September 2026) replaced that in turn with bitcoind's REST interface via
  the `bindex` library, which requires Bitcoin Core 31.0+ (released April 2026)
  running with `rest=1`, plus a full reindex. RocksDB storage. Upstream
  `doc/usage.md` at v0.12.0 reports a first sync of ~2h against ~800 GB of
  `blocks/*.dat` on a 6-core / 32 GB / NVMe box (July 2026), and ~18h on a
  2 GB-RAM ODROID-HC1 with an SSD.
- Fulcrum talks to bitcoind purely over JSON-RPC (optionally with a
  `zmqpubhashblock` subscription for block notifications), fetching blocks via
  `getblock` calls in parallel worker threads, and requires `txindex=1` on a
  non-pruned node. RocksDB storage with a larger in-memory cache. Upstream's
  README quotes 4 to 20+ h for a mainnet sync on SSD, days on an HDD (a figure
  carried unchanged in that README since v1.0, January 2020; it does not
  separate BCH from BTC mainnet, and upstream has never tested the HDD case).

Memory model:

- electrs holds minimal state in RAM; RocksDB block cache is small. Steady
  state ~1.5-2.5 GB.
- Fulcrum keeps a multi-GB index hot in RAM (configurable via `db_mem`; the
  `fast-sync` / UTXO-cache knob was removed in 2.0.0, September 2025). Steady
  state ~3-6 GB; sync peak 8+ GB.

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
| Smallest disk footprint | electrs (~56 GB measured index vs Fulcrum's ~133 GB recommended minimum) |
| Tor-only node, infrequent queries | electrs |
| Audited supply chain (Rust ecosystem) | electrs |

Both disk figures come from upstream docs and are far larger than the
10-GB-scale numbers this table used to quote. They are not like-for-like,
though: electrs `doc/usage.md` at v0.12.0 measures a real datadir - `du -h
db/bitcoin/` at 56G against 802G of `blocks/*.dat`, about 7%, and transiently
~14% just before the final compaction (July 2026) - whereas Fulcrum's ~133 GB
for BTC mainnet is a "Recommended hardware" minimum in the README, quoted "as
of Aug 2023" and not revised since, not a measurement of a synced index. Size
the index disk from those, on top of the node's own ~800 GB of blocks.

## Common pitfalls

- Assuming Fulcrum is always faster - on a 2 GB VM Fulcrum will OOM-thrash
  while electrs stays responsive; "fastest" depends on RAM headroom.
- Migrating data - the RocksDB layouts are not compatible. Switching servers
  means a fresh re-index; do not try to copy the `db/` folder. Fulcrum-to-
  Fulcrum is a different matter: since 2.0.0 (September 2025) the Fulcrum
  datadir is platform-neutral and can be copied between OSes and CPU
  architectures, though a 1.x datadir still needs a one-time `--db-upgrade`.
- Assuming either backend is a drop-in swap for the other on the same node -
  since v0.12.0 (September 2026) electrs refuses to index against Bitcoin Core
  older than 31.0 and needs `rest=1` in `bitcoin.conf`, while Fulcrum runs
  against Core v0.17.0+ over JSON-RPC and needs `txindex=1` (which electrs does
  not use). Check the node's version and config before switching.
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
- electrs v0.12.0 release notes (bindex, Core 31+, 13 Sep 2026): https://github.com/romanz/electrs/releases/tag/v0.12.0
- electrs index-size and sync figures: https://github.com/romanz/electrs/blob/v0.12.0/doc/usage.md
- electrs upgrade notes (disk -> P2P at 0.9.0, P2P -> REST at 0.12.0): https://github.com/romanz/electrs/blob/v0.12.0/doc/upgrading.md
- Fulcrum hardware recommendations: https://github.com/cculianu/Fulcrum#requirements
- Electrum protocol spec: https://electrumx.readthedocs.io/en/latest/protocol-methods.html
- Independent benchmark thread: https://github.com/cculianu/Fulcrum/discussions
