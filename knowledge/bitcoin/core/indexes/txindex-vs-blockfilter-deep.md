# txindex vs blockfilterindex - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/core/indexes`.
> Canonical source: https://github.com/bitcoin/bips/blob/master/bip-0158.mediawiki
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/core/indexes/SKILL.md

## Concept

`txindex` and `blockfilterindex` are both auxiliary indexes Bitcoin Core can maintain on top of its primary block storage and chainstate, but they answer different questions and should be chosen by question, not by reflex. `txindex` answers "where on chain is this *known* txid?". `blockfilterindex` answers "which blocks *might* contain a script matching this descriptor / address?". The first is a precise key-value lookup; the second is a probabilistic prefilter that you then verify by downloading the candidate blocks. They complement rather than substitute, and their cost profiles are very different.

## Walkthrough / mechanics

`txindex` is implemented in `src/index/txindex.cpp` as a LevelDB at `<datadir>/indexes/txindex/` mapping `txid -> (block hash, file pos, length)`. Build cost scales with the cumulative *transaction* count rather than with block bytes, since the index stores one record per txid: ~80 GB measured at height ~840,000 (April 2024), a figure that has not been re-measured since. Cumulative transactions grew from 991,029,144 on 20 April 2024 to 1,439,337,665 on 15 September 2026 (blockchain.com n-transactions-total chart), about 45% more records, so budget roughly half again as much on a node at the current tip (height 967,284, 16 September 2026). It is populated during IBD if enabled from the start, or rebuilt in 3-12 hours on a synced node. The lookup itself is O(1).

`blockfilterindex` (BIP 158) implements compact block filters: a Golomb-Rice-encoded set of script hashes per block (`basic` filter type covers all output scripts and previous output scripts spent by the block). Storage is `<datadir>/indexes/blockfilter/basic/`, ~6 GB total at that same April 2024 height; filters are one per block, so this one does scale with block data, which grew about 36% from ~566 GB on 20 April 2024 to ~769 GB on 15 September 2026 (blockchain.com blockchain-size chart). To check whether a block contains an address of interest, a client computes the SipHash of the script and tests the filter; false positives ~1 in 784,931. On a hit, the client downloads the full block and confirms.

`peerblockfilters=1` further makes Core serve these filters over the P2P network so SPV-style light clients (BIP 157 GETCFILTERS) can use them remotely. Without `peerblockfilters` the filters are local-only.

When you want both questions answered: enable both. They do not conflict. `coinstatsindex` is a separate concern (rolling UTXO stats), and so is `txospenderindex`, added in Bitcoin Core 31.0 (April 2026), which answers a further question — "which transaction spent this *outpoint*?" — by storing one LevelDB key per non-coinbase transaction input at `<datadir>/indexes/txospenderindex/` and is consulted by `gettxspendingprevout`. As of 31.1 (July 2026) that makes four optional indexes in total. Like `txindex`, it refuses to start on a pruned node.

## Worked example

Compare the two indexes on a synced node:

```ini
# /etc/bitcoin/bitcoin.conf
txindex=1
blockfilterindex=1
peerblockfilters=1
```

Use `txindex` for forensic lookups:

```bash
# Look up a specific txid without knowing its block
$ bitcoin-cli getrawtransaction 4a5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b true | jq '.blockhash, .confirmations'
"00000000000000000000c5e9...b3a"
142573
```

Use `blockfilterindex` to scan the chain for a descriptor without rebuilding history:

```bash
# Get the filter for a recent block
$ HASH=$(bitcoin-cli getbestblockhash)
$ bitcoin-cli getblockfilter $HASH basic | jq '.filter[:32]'
"01a9d4c8f2..."

# Scan a range of blocks for outputs paying a descriptor
$ bitcoin-cli scanblocks start \
    '["wpkh([d34db33f/84h/0h/0h]xpub6CV2.../<0;1>/*)"]' \
    960000 967000 basic
{
  "from_height": 960000,
  "to_height": 967000,
  "relevant_blocks": [
    "00000000...e1c",
    "00000000...a7b",
    ...
  ],
  "completed": true
}
```

`scanblocks` returns candidate blocks; you then `getblock` each one and post-filter with the descriptor. This is the workflow Electrs uses for fast wallet rescans. It is far cheaper than `scantxoutset` for many descriptors against the entire chain.

Cost summary on a typical x86_64 SSD node, measured at height ~840,000 (April 2024) and not re-measured since. To project onto the September 2026 tip, scale `txindex` by the ~45% growth in cumulative transactions and `blockfilterindex` by the ~36% growth in block data:

| Index | Disk | IBD impact | Build from scratch | Use |
|---|---|---|---|---|
| `txindex` | ~80 GB | +20-50% | 3-12 h | exact txid lookup |
| `blockfilterindex` | ~6 GB | +5-10% | 1-3 h | descriptor scan |
| Both | ~86 GB | additive | parallel | full forensic |

## Common pitfalls

- Enabling `txindex` for a service that only needs to scan recent blocks for descriptor matches. The expensive `txindex` rebuild does not help; `blockfilterindex` does.
- Enabling `blockfilterindex=1` without `peerblockfilters=1` and then wondering why a downstream Electrs instance is missing data. Some indexers fetch via the local filter index directly (datadir read), but BIP 157 P2P clients need the peer flag.
- Believing filter false positives mean money is missed. They mean extra block downloads, never missed matches.
- Forgetting that a pruned node can run `blockfilterindex` (filters survive pruning of the underlying blocks since filters are computed at the time the block is processed) but loses the ability to *fetch* the original block on a positive match. Fine for filter-only services; broken for `scanblocks` which still needs to download the block to verify.
- Adding `txindex=1` — or, since 31.0, `txospenderindex=1` — to a pruned node: bitcoind exits at startup with a clear error ("Prune mode is incompatible with -txindex." / "... -txospenderindex.").
- Expecting `coinstatsindex` to speed up `scantxoutset`. It does not; `scantxoutset` iterates the chainstate cursor itself and takes minutes regardless of which indexes are enabled. For repeated scans, use `blockfilterindex` and `scanblocks` instead.
- Reaching for `txindex` to find out whether an output was *spent*. `txindex` maps txid to location and says nothing about spends; that is what `txospenderindex` plus `gettxspendingprevout` are for (31.0+).

## References

- BIP 157 / BIP 158 (compact block filters).
- `src/index/txindex.cpp`, `src/index/blockfilterindex.cpp`.
- `scanblocks`, `getblockfilter`, `scantxoutset`, `gettxspendingprevout` RPCs.
- `src/index/txospenderindex.cpp`; `doc/files.md` for the current `indexes/` layout.
