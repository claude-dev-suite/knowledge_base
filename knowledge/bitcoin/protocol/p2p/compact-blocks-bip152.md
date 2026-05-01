# Compact Blocks (BIP152) - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/p2p`.
> Canonical source: BIP152
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/p2p/SKILL.md

## Concept

Compact blocks reduce per-hop block propagation latency from
hundreds of milliseconds (full block transfer) to tens of milliseconds.
Receivers reconstruct the block from a header + 6-byte short txids +
small list of "prefilled" txs, mostly using mempool tx data already
present locally. The skill names the messages; this article walks
the algorithm, the reconstruction failure cases, and the high-bandwidth
vs low-bandwidth signaling modes.

## Walkthrough / mechanics

**Messages added by BIP152:**

| Message | Direction | Purpose |
|---------|-----------|---------|
| `sendcmpct` | both | announce compact-block willingness, version, mode |
| `cmpctblock` | sender->receiver | compact-block payload |
| `getblocktxn` | receiver->sender | request specific txs by index |
| `blocktxn` | sender->receiver | reply with full txs at requested indices |

**`sendcmpct` payload:**

```
[1 byte]  announce_high_bw   (1 = mode 1, 0 = mode 0)
[8 bytes] version_le         (1 = legacy txid, 2 = wtxid post BIP339)
```

Mode 0 (low-bandwidth, default): peer must `inv` first, receiver
asks for cmpctblock via `getdata`.

Mode 1 (high-bandwidth): peer pushes `cmpctblock` immediately on new
block. Used for trusted relays and reduces latency by one round trip.

Bitcoin Core selects up to 3 high-bandwidth peers automatically based
on first-to-deliver-block heuristics.

**`cmpctblock` payload:**

```
[ 80 bytes ] block header
[ 8 bytes  ] nonce (random, used to randomize short id keys)
[ varint   ] short_ids count
[ 6 bytes * ] short_ids array (one per non-prefilled tx)
[ varint   ] prefilled_txn count
[ ...      ] prefilled_txn array: { varint diff-index, full tx }
```

Short-id derivation:

```
header_hash    = hash256(block.header || nonce)
sip_key0       = first 8 bytes of header_hash
sip_key1       = next 8 bytes of header_hash
short_id(tx)   = SipHash-2-4(sip_key0, sip_key1, txid_or_wtxid)[0..6]
```

The receiver SipHashes its mempool txids with the same keys, builds
a lookup `short_id -> txid`, and matches against the cmpctblock list.

**Prefilled txs** are txs the sender knows the receiver doesn't have
in mempool, sent in full. The coinbase is always prefilled (index 0).

## Worked example

**Sender side (after mining a block):**

```python
def make_cmpctblock(block, peer_known_txids):
    nonce = os.urandom(8)
    header = block.header.serialize()
    h = hash256(header + nonce)
    k0, k1 = h[:8], h[8:16]

    short_ids = []
    prefilled = [(0, block.txs[0])]  # coinbase always prefilled
    for i, tx in enumerate(block.txs[1:], start=1):
        if tx.txid in peer_known_txids:
            short_ids.append(siphash24(k0, k1, tx.txid)[:6])
        else:
            prefilled.append((i, tx))

    return {
        "header": header,
        "nonce": nonce,
        "short_ids": short_ids,
        "prefilled": prefilled,
    }
```

**Receiver side:**

```python
def reconstruct(cmpct, mempool, peer_block):
    h = hash256(cmpct["header"] + cmpct["nonce"])
    k0, k1 = h[:8], h[8:16]

    # build mempool short-id index
    sid_to_tx = {}
    for tx in mempool:
        sid = siphash24(k0, k1, tx.txid)[:6]
        sid_to_tx[sid] = tx

    # try to fill block.txs by index
    block_txs = [None] * (len(cmpct["short_ids"]) + len(cmpct["prefilled"]))
    for i, tx in cmpct["prefilled"]:
        block_txs[i] = tx

    missing_indices = []
    short_id_iter = iter(cmpct["short_ids"])
    for i in range(len(block_txs)):
        if block_txs[i] is None:
            sid = next(short_id_iter)
            if sid in sid_to_tx:
                block_txs[i] = sid_to_tx[sid]
            else:
                missing_indices.append(i)

    if missing_indices:
        ask_getblocktxn(peer_block, missing_indices)
        # peer responds with blocktxn -> fill remaining
    return block_txs
```

**Failure modes:**

1. **SipHash collision in mempool.** Two different txids produce the
   same 6-byte short id. Probability: `2^-48` per pair, but with
   thousands of txs in mempool this can occur. Receiver detects when
   reconstructed merkle root doesn't match header; falls back to
   full block download via `getdata BLOCK`.

2. **Cache miss.** Tx wasn't in receiver's mempool (orphan, fee-floor
   filtered, recent broadcast). Receiver requests via `getblocktxn`.

3. **Reorder.** Tx was in mempool but with different witness data
   (RBF replacement). Match by txid succeeds but block-tx witness
   differs. Use BIP339 wtxid relay to detect.

## Common bugs / pitfalls

1. **Using legacy txid post-BIP339.** Mode 2 of `sendcmpct` uses
   wtxid; nodes that announce v2 then send v1-style short ids
   confuse receivers.
2. **Coinbase not prefilled.** Coinbase is never in any peer's
   mempool, so it MUST be in the prefilled list. Forgetting causes
   immediate `getblocktxn` round trip for index 0.
3. **Nonce reuse across blocks.** Reusing the same nonce lets an
   attacker craft txs with chosen short-id collisions for the next
   block. Nonce must be fresh per cmpctblock.
4. **High-bandwidth mode with too many peers.** Mode 1 broadcasts
   `cmpctblock` to all configured peers immediately. Set a sane
   limit (Core uses 3) to avoid bandwidth amplification.
5. **`getblocktxn` indices out-of-range.** Indices are tx positions
   in the block, 0-based. Sending an index >= block length is
   ill-formed; peer disconnects.
6. **Mempool-snapshot race.** Between receiving `cmpctblock` and
   reconstruction, the receiver's mempool may evict a tx that was
   in cmpctblock's short_ids. Use a snapshot or accept the
   `getblocktxn` round trip.

## References

- BIP152: https://github.com/bitcoin/bips/blob/master/bip-0152.mediawiki
- BIP339 (wtxid relay): https://github.com/bitcoin/bips/blob/master/bip-0339.mediawiki
- Bitcoin Core impl: `src/net_processing.cpp`, `src/blockencodings.cpp`
- SipHash spec: https://www.aumasson.jp/siphash/siphash.pdf
