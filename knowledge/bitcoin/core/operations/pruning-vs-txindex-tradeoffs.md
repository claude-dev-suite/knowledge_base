# Pruning vs txindex Tradeoffs - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/core/operations`.
> Canonical source: https://github.com/bitcoin/bitcoin/blob/master/doc/reduce-traffic.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/core/operations/SKILL.md

## Concept

`prune=` and `txindex=1` represent opposite ends of a spectrum about what historical data the node retains. Pruning discards block files past a configurable retention floor, drops the node from a full archival peer to a `NODE_NETWORK_LIMITED` peer (only the last ~288 blocks served), and is mutually exclusive with any of the optional indexes. `txindex=1` is the maximalist opposite: maintain a `txid -> (block, position)` map across the entire chain so any historical transaction can be retrieved by id alone. The two settings cannot coexist; switching between them requires a full reindex.

## Walkthrough / mechanics

Block storage layout: `<datadir>/blocks/blk*.dat` files (~128 MB each), with `rev*.dat` undo files alongside. The chainstate (UTXO set) lives in `<datadir>/chainstate/`. With `prune=N`, Core deletes old `blk*.dat`/`rev*.dat` once their content is older than the retention window in megabytes. The minimum allowed is `prune=550` (~3 days of recent history); the practical minimum is closer to `prune=2000` (~10 days) to survive a multi-day reorg or chain reorg-window operations.

`txindex=1` is a separate LevelDB at `indexes/txindex/` that maps every txid in every block to a position in the block files. Without it, `getrawtransaction <txid>` only succeeds for txs that are currently in mempool, were broadcast since the daemon started, or whose blockhash you also pass as a third argument (which avoids the index entirely by just reading the block).

The asymmetry to internalize: pruning removes the underlying block bytes. txindex adds a lookup table on top of those bytes. You cannot have both because txindex's map points at block bytes the prune flag would have erased.

A pruned node still validates everything during IBD. It just discards block files behind it. The UTXO set and chainstate are always retained.

## Worked example

A pruned node config (Lightning gateway with on-chain peg-in support):

```ini
# /etc/bitcoin/bitcoin.conf
prune=15000
maxuploadtarget=2000
blockfilterindex=1
peerblockfilters=1
```

A full archival node config (block explorer):

```ini
prune=0
txindex=1
blockfilterindex=1
coinstatsindex=1
dbcache=4096
```

Disk usage at height ~840000 (early 2026):

| Setting | blocks/ | chainstate/ | indexes/ | Total |
|---|---|---|---|---|
| `prune=550` | ~0.6 GB | ~10 GB | n/a | ~11 GB |
| `prune=15000` | ~15 GB | ~10 GB | ~6 GB (filters) | ~31 GB |
| Full no indexes | ~700 GB | ~10 GB | n/a | ~710 GB |
| Full + txindex | ~700 GB | ~10 GB | ~80 GB | ~790 GB |
| Full + all indexes | ~700 GB | ~10 GB | ~87 GB | ~797 GB |

Switch from pruned to full archival (a full chain redownload):

```bash
$ bitcoin-cli stop
# Edit conf: remove prune=, add txindex=1
# Cannot un-prune in place; you must reindex.
$ bitcoind -reindex
```

Switch from no-txindex to txindex on a full node (rebuild index only, days faster than `-reindex`):

```bash
$ bitcoin-cli stop
# Edit conf: add txindex=1
$ bitcoind -daemon
$ tail -f ~/.bitcoin/debug.log
... Building txindex (3-12 hours on SSD)
... txindex is enabled
```

Demonstrating the lookup difference:

```bash
# Without txindex, must supply blockhash
$ bitcoin-cli getrawtransaction 4a5e1e... true
error code: -5
error message:
No such mempool transaction. Use -txindex...

$ bitcoin-cli getrawtransaction 4a5e1e... true 0000000000...e7c
{ "txid": "...", "vin": [...], ... }

# With txindex, blockhash optional
$ bitcoin-cli getrawtransaction 4a5e1e... true
{ "txid": "...", "vin": [...], "blockhash": "...", "confirmations": 142000 }
```

## Common pitfalls

- Setting `prune=550` on a node that runs Electrs or Fulcrum: the indexer needs full historical block data and falls over within minutes. Pair pruning only with services that read recent blocks via `blockfilterindex` (LDK, btcwallet).
- Believing pruned nodes do not validate. They do; pruning only affects retention.
- Adding `txindex=1` to a pruned node: bitcoind exits at startup with a clear message. Remove `prune=` first or switch carefully.
- Removing `txindex=1` from `bitcoin.conf` and expecting disk to shrink. The index files remain. Stop bitcoind and `rm -rf indexes/txindex/` manually.
- Forgetting that `getrawtransaction <txid> verbose <blockhash>` works without txindex on a non-pruned node, and is faster than running an index for one-off forensic queries.
- Pruning interacting with `blockfilterindex`: filters are kept full, but the underlying blocks are gone. A peer asking for a block via `getdata` gets a "not available" reply, but compact-filter queries still work.

## References

- `doc/reduce-traffic.md` in bitcoin/bitcoin.
- `src/node/blockstorage.cpp` for prune file deletion.
- `src/index/txindex.cpp` for txindex maintenance.
- BIP 158 for compact block filter semantics on pruned nodes.
