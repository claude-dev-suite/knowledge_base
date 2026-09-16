# coinstatsindex Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/core/indexes`.
> Canonical source: https://github.com/bitcoin/bitcoin/blob/master/src/index/coinstatsindex.cpp
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/core/indexes/SKILL.md

## Concept

`gettxoutsetinfo` answers questions about the entire UTXO set: count, total value, hash. By default it does this by walking every entry in the chainstate LevelDB live on each call. On a modern node that is well over a hundred million records (~166.4 million as of 16 September 2026, per blockchain.com's UTXO-count chart) and takes minutes, locking other operations. `coinstatsindex` precomputes and rolls these stats forward as new blocks arrive, turning the call into an O(1) lookup. It is also the only way to get the `muhash` UTXO commitment, an order-independent hash that is much cheaper to maintain incrementally than a Merkle-tree-style commitment.

## Walkthrough / mechanics

The index lives in `<datadir>/indexes/coinstatsindex/` and is roughly 1 GB on a synced node. Bitcoin Core 30.0 (October 2025) rewrote the index to fix an overflow bug that was already observable on the default Signet (#30469); the rewritten index syncs from scratch on the first start after the upgrade and moved from the old `<datadir>/indexes/coinstats/` to the current path. The old directory is deliberately not deleted, so that a downgrade to 29.x or lower still finds its index. Build cost on first enable is several hours of CPU. After that, each new block delta updates the running totals; the index is always within one block of the tip.

`gettxoutsetinfo` accepts a `hash_type` parameter:

- `hash_serialized_3` (default): legacy-algorithm hash. Always requires a full UTXO walk — `coinstatsindex` is consulted only for `muhash` and `none` — and cannot be queried for a specific block. Order-dependent.
- `none`: fast, returns count and value without a hash.
- `muhash`: order-independent multiplicative-set UTXO hash. Computable incrementally with `coinstatsindex` enabled, otherwise extremely slow. It is a Bitcoin Core implementation feature with no assigned BIP (checked against the BIPs index, September 2026).

`hash_serialized_2` was removed from `gettxoutsetinfo` in Bitcoin Core 26.0 (December 2023) because the value it computed contained a bug and did not take all data into account; `hash_serialized_3` supersedes it (#28685). Passing `hash_serialized_2` to a 26.0 or newer node is an invalid `hash_type` error.

It also accepts `hash_or_height` to query a *historical* UTXO state, which only works when the index is built. Without the index you can only query the tip.

`total_unspendable_amount` is cumulative over the whole chain, but every field under `block_info` is a delta for the queried block alone, including the `unspendables` breakdown (`src/rpc/blockchain.cpp` at v31.1 subtracts the previous height's running totals). A recent block normally reports zeros there; the 50 BTC genesis subsidy shows up only in the `genesis_block` delta at height 0.

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
# Tip state, all hash types, fast. Sample values below are illustrative.
# The height is the tip as of 16 September 2026; txouts is blockchain.com's
# UTXO-count series for that date, which tracks Core's own count closely but
# is not the same metric. total_amount is scheduled issuance through that
# height (20,085,265.625 BTC) less total_unspendable_amount.
$ bitcoin-cli gettxoutsetinfo muhash
{
  "height": 967284,
  "bestblock": "00000000...3ba",
  "txouts": 166380577,
  "bogosize": 12478543275,
  "muhash": "0a3f...e9b",
  "total_amount": 20084821.285,
  "total_unspendable_amount": 444.34,
  "block_info": {
    "prevout_spent": 12345.67,
    "coinbase": 3.125,
    "new_outputs_ex_coinbase": 12345.67,
    "unspendable": 0.0,
    "unspendables": {
      "genesis_block": 0.0,
      "bip30": 0.0,
      "scripts": 0.0,
      "unclaimed_rewards": 0.0
    }
  }
}

# Historical UTXO state at a fixed height
$ bitcoin-cli gettxoutsetinfo none 800000
{ "height": 800000, "bestblock": "00000000...e7c", "txouts": 152901234, ... }

# Historical at a specific block hash
$ bitcoin-cli gettxoutsetinfo none "00000000...e7c"
```

Without the index, the same call against tip with `hash_serialized_3`:

```bash
$ bitcoin-cli gettxoutsetinfo hash_serialized_3
# blocks for 1-3 minutes, holds wallet lock
{ "height": 967284, "txouts": 166380577, ... }
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
- Believing `coinstatsindex` is required to call `gettxoutsetinfo`. Every `hash_type` works at tip without the index, just slowly. The index is required only for historical queries and for fast `muhash`.
- Using `hash_serialized_2` at all. It was removed in 26.0 (December 2023); scripts that still pass it fail with an invalid `hash_type` error against any node from that release on.
- Using `hash_serialized_3` for cross-version reproducibility, or expecting `coinstatsindex` to speed it up. It is order-dependent, and the index is bypassed for it entirely, so it always walks the whole UTXO set; `muhash` is order-independent, stable, and the hash type the index actually accelerates.
- Running multiple `gettxoutsetinfo` calls in parallel without the index. Each grabs `cs_main`; they serialize and starve other RPCs.
- Disabling the index by removing the conf line. Like `txindex`, the disk files remain. Stop bitcoind and `rm -rf indexes/coinstatsindex/` to reclaim space.
- Assuming the 30.0 upgrade reclaimed the old index's disk. It did not: `indexes/coinstats/` survives alongside the new `indexes/coinstatsindex/`, and bitcoind logs a warning naming it on every start. Delete it by hand once a downgrade to 29.x or lower is off the table.
- Confusing `coinstatsindex` with the consensus-level UTXO commitment. There is no consensus commitment; `muhash` is a soft, opt-in tooling feature for snapshots and assumeutxo verification.

## References

- `src/index/coinstatsindex.cpp` in bitcoin/bitcoin.
- `src/kernel/coinstats.cpp` in bitcoin/bitcoin (muhash and `hash_serialized_3` computation). There is no assigned BIP for muhash.
- `gettxoutsetinfo` RPC reference.
- assumeutxo design notes in `doc/assumeutxo.md`.
