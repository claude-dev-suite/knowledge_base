# ZMQ Integration Patterns with Electrs - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/core/zmq`.
> Canonical source: https://github.com/romanz/electrs/blob/master/doc/usage.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/core/zmq/SKILL.md

## Concept

Electrs (and its sibling Fulcrum) is a Bitcoin Core address-indexer. It maintains a per-script index that turns "show me all history for `bc1q...`" into milliseconds of work, something Core itself never does. The architectural question is how Electrs learns that the chain advanced. The cheap, reliable answer is to combine Bitcoin Core's `blockfilterindex` for bulk historical scans with the ZMQ `hashblock` topic for tip notifications. ZMQ makes Electrs reactive to new blocks within a few hundred milliseconds; without it, Electrs polls `getblockcount` every few seconds and adds avoidable latency to every Lightning channel close, every BTCPay invoice, every Sparrow refresh.

## Walkthrough / mechanics

Electrs's mainline index loop polls Bitcoin Core's RPC. With ZMQ enabled and configured to listen, its `wait_for_new_block` step blocks on `hashblock` instead of polling. The `electrs.toml` setting is `daemon_p2p_addr` for direct P2P plus `daemon_rpc_addr` for RPC; the ZMQ tap is added separately as the `--zmq-pub-hashblock` flag or equivalent toml key.

Electrs uses Core in three distinct paths:

1. **Bulk historical scan**: Electrs reads block files directly from the Core datadir (it must run on the same machine or with shared block storage). For this it needs `blockfilterindex=1` so it can use compact filters to identify blocks of interest, and unpruned `blocks/`.
2. **Tip detection**: ZMQ `hashblock` push lets Electrs react to new blocks instantly.
3. **Mempool**: Electrs polls `getrawmempool` every few seconds; some forks subscribe to `rawtx` for lower-latency mempool indexing.

For a Lightning node co-located with a Core+Electrs stack, the chain of pushes is:

```
new block -> Bitcoin Core -> ZMQ hashblock -> Electrs index updated
                          -> ZMQ rawblock  -> LND/CLN watchtower
```

Both subscribers can listen to the same publisher port without contention; ZMQ pub-sub is multi-subscriber by design.

## Worked example

`bitcoin.conf` for the node side:

```ini
# Required for Electrs's filter-driven scan
blockfilterindex=1
peerblockfilters=1

# ZMQ taps Electrs and any other consumer can subscribe
zmqpubhashblock=tcp://127.0.0.1:28334
zmqpubrawtx=tcp://127.0.0.1:28333

# Electrs talks to Core P2P for some operations
listen=1
bind=127.0.0.1:8333
```

`electrs.toml`:

```toml
network = "bitcoin"
daemon_dir = "/var/lib/bitcoind"
daemon_rpc_addr = "127.0.0.1:8332"
daemon_p2p_addr = "127.0.0.1:8333"
electrum_rpc_addr = "127.0.0.1:50001"
auth = "user:secret_password"

# ZMQ: subscribe to hashblock for tip push notifications
# Older electrs versions: pass via --zmq-pub-hashblock CLI flag
# Newer: use the toml key
[index]
batch_size = 50

[zmq]
hashblock_addr = "tcp://127.0.0.1:28334"
```

Run:

```bash
$ electrs --conf /etc/electrs/electrs.toml --log-filters "INFO,electrs::index=DEBUG"
INFO  Starting Electrs ...
INFO  Connected to bitcoind 26.0 at 127.0.0.1:8332
INFO  Connected to ZMQ hashblock at tcp://127.0.0.1:28334
INFO  Catching up: from height 800000 to tip 840127
... (initial sync, can be hours)
INFO  Indexed up to height 840127
... (idle, then on new block:)
DEBUG ZMQ hashblock 00000000...e3a
INFO  Indexed new tip 840128
```

Cross-check tip latency from a Sparrow / Electrum client: time between block confirmation on a public block explorer and the moment Electrum shows confirmation drops from ~5 seconds (polling) to ~200 ms (ZMQ).

A second integrator on the same node: a Lightning node uses `rawblock` to feed its watchtower:

```ini
# bitcoin.conf, additional tap
zmqpubrawblock=tcp://127.0.0.1:28332
```

LND `lnd.conf`:

```ini
[Bitcoind]
bitcoind.zmqpubrawblock=tcp://127.0.0.1:28332
bitcoind.zmqpubrawtx=tcp://127.0.0.1:28333
bitcoind.rpcuser=lnd
bitcoind.rpcpass=secret
bitcoind.rpchost=127.0.0.1:8332
```

Both Electrs and LND happily subscribe to the same publisher concurrently.

## Common pitfalls

- Mounting Electrs on a pruned node. Electrs needs full block bytes to index history, regardless of `blockfilterindex`. A pruned node fails partway through the initial scan.
- Using `tcp://0.0.0.0:28334` for ZMQ on a public node. There is no auth, no encryption. A bad actor can flood your subscribers or saturate bandwidth requesting blocks. Bind to `127.0.0.1` and tunnel via SSH or a private network.
- Forgetting that ZMQ does not replay missed events. If Electrs restarts and misses a block, it must catch up via RPC. Electrs handles this; a custom subscriber may not.
- Subscribing to `rawtx` from Electrs and expecting confirmation indication. `rawtx` fires for both mempool entry and block inclusion. Use `sequence` if you need to distinguish.
- Running multiple Electrs instances against one node and assuming ZMQ load-balances. ZMQ pub-sub is broadcast: every subscriber gets every message. There is no consumer-group semantics.
- Lightning node + Electrs both binding the same RPC user. Use distinct `rpcauth=` lines so credential rotation is independent.

## References

- Electrs README and `doc/usage.md`.
- Fulcrum docs (`docs/configuration.md`).
- Bitcoin Core `doc/zmq.md`.
- BIP 158 (compact block filters), BIP 157 (P2P serving).
