# coinstatsindex Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/core/indexes`.
> Canonical source: https://github.com/bitcoin/bitcoin/blob/master/src/index/coinstatsindex.cpp
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/core/indexes/SKILL.md

## Concept

`gettxoutsetinfo` answers questions about the entire UTXO set: count, total value, hash. By default it does this by walking every entry in the chainstate LevelDB live on each call. On a modern node that is hundreds of millions of records and takes minutes, locking other operations. `coinstatsindex` precomputes and rolls these stats forward as new blocks arrive, turning the call into an O(1) lookup. It is also the only way to get the `muhash` UTXO commitment, an order-independent hash that is much cheaper to maintain incrementally than a Merkle-tree-style commitment.

## Walkthrough / mechanics

The index lives in `<datadir>/indexes/coinstats/` and is roughly 1 GB on a synced node. Build cost on first enable is several hours of CPU. After that, each new block delta updates the running totals; the index is always within one block of the tip.

`gettxoutsetinfo` accepts a `hash_type` parameter:

- `none` (default): fast, returns count and value without a hash.
- `hash_serialized_2`: legacy hash, requires a full UTXO walk if `coinstatsindex` is not enabled. Order-dependent.
- `muhash`: BIP 0?? proposed UTXO commitment. Computable incrementally with `coinstatsindex` enabled, otherwise extremely slow.

It also accepts `hash_or_height` to query a *historical* UTXO state, which only works when the index is built. Without the index you can only query the tip.

The index can be combined with pruning: pruning removes block files but the chainstate, undo data within the prune window, and the index itself remain available. So a pruned node with `coinstatsindex=1` can answer rolling stats but cannot answer arbitrary historical heights past the prune horizon.

## Worked example

Enable on an existing node:

```bash
$ bitcoin-cli stop
# Append to bitcoin.conf:
#   coinstatsindex=1
$ bitcoind -daemon
$ tail -f ~/.bitcoin/debug.log | grep -i coinstats
2026-04-29T12:00:01Z Coinstatsindex starting
2026-04-29T15:30:42Z Coinstatsindex is enabled
```

Then run queries that would otherwise be expensive:

```bash
# Tip state, all hash types, fast
$ bitcoin-cli gettxoutsetinfo muhash
{
  "height": 840127,
  "bestblock": "00000000...3ba",
  "txouts": 158234567,
  "bogosize": 11875432091,
  "muhash": "0a3f...e9b",
  "total_amount": 19654321.87654321,
  "total_unspendable_amount": 215.43,
  "block_info": {
    "prevout_spent": 12345.67,
    "coinbase": 6.25,
    "new_outputs_ex_coinbase": 12345.67,
    "unspendable": 0.0,
    "unspendables": {
      "genesis_block": 50.0,
      "bip30": 0.0,
      "scripts": 215.43,
      "unclaimed_rewards": 178.91
    }
  }
}

# Historical UTXO state at a fixed height
$ bitcoin-cli gettxoutsetinfo none 800000
{ "height": 800000, "bestblock": "00000000...e7c", "txouts": 152901234, ... }

# Historical at a specific block hash
$ bitcoin-cli gettxoutsetinfo none "00000000...e7c"
```

Without the index, the same call against tip with `hash_serialized_2`:

```bash
$ bitcoin-cli gettxoutsetinfo hash_serialized_2
# blocks for 1-3 minutes, holds wallet lock
{ "height": 840127, "txouts": 158234567, ... }
```

The historical query fails entirely without the index:

```bash
$ bitcoin-cli gettxoutsetinfo none 800000
error code: -8
error message:
Querying specific block heights requires coinstatsindex
```

Periodic monitoring loop (Prometheus exporter style):

```bash
while true; do
  bitcoin-cli gettxoutsetinfo none | jq -r \
    '"bitcoin_utxo_count \(.txouts)\nbitcoin_utxo_value \(.total_amount)"'
  sleep 60
done > /var/lib/node_exporter/textfile/bitcoin.prom
```

With `coinstatsindex` this loop is essentially free; without it, it locks the node for minutes every minute.

## Common pitfalls

- Enabling `coinstatsindex` and immediately running queries before the build completes. The RPC returns "Index is not built" until the background sync reaches the tip.
- Believing `coinstatsindex` is required to call `gettxoutsetinfo`. The default (`hash_type=none` at tip) works without the index, just slowly. The index is required only for historical queries and for fast `muhash`.
- Using `hash_serialized_2` for cross-version reproducibility. Its computation can vary between Core releases that change LevelDB iteration order; `muhash` is order-independent and stable.
- Running multiple `gettxoutsetinfo` calls in parallel without the index. Each grabs `cs_main`; they serialize and starve other RPCs.
- Disabling the index by removing the conf line. Like `txindex`, the disk files remain. Stop bitcoind and `rm -rf indexes/coinstats/` to reclaim space.
- Confusing `coinstatsindex` with the consensus-level UTXO commitment. There is no consensus commitment; `muhash` is a soft, opt-in tooling feature for snapshots and assumeutxo verification.

## References

- `src/index/coinstatsindex.cpp` in bitcoin/bitcoin.
- BIP draft for muhash UTXO set commitment.
- `gettxoutsetinfo` RPC reference.
- assumeutxo design notes in `doc/assumeutxo.md`.
