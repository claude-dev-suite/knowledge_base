# txid vs wtxid Explained - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/transactions`.
> Canonical source: https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/transactions/SKILL.md

## Concept

Pre-SegWit, every transaction had a single identifier: `txid = SHA256d(serialize(tx))`. Witness data was stored inside the scriptSig and was therefore covered by the txid - making any unsigned-but-malleable signature visibly change the txid (transaction malleability). BIP141 split serialization into "stripped" (no witness) and "full" (with witness), creating two ids: `txid` for the stripped form and `wtxid` for the full form. Tooling that confuses them produces silent bugs.

## Walkthrough / mechanics

### Stripped serialization (txid input)

```
nVersion (4)
vin_count (varint)
vin[]:
  outpoint (36)
  scriptSig (varint+bytes)         <-- legacy may have data here
  nSequence (4)
vout_count (varint)
vout[]:
  value (8)
  scriptPubKey (varint+bytes)
nLockTime (4)
```

`txid = SHA256d(serialize_stripped(tx))`. This is what the merkle tree in the block header commits to and what `vin[i].outpoint.txid` references when spending.

### Full serialization (wtxid input)

```
nVersion (4)
0x00 0x01            <-- marker + flag, present only for segwit txs
vin_count (varint)
vin[] same as above (scriptSig is empty for native segwit inputs)
vout_count (varint)
vout[] same as above
witness[]:           <-- one stack per input
  for each input:
    item_count (varint)
    items[]:
      length (varint) || bytes
nLockTime (4)
```

`wtxid = SHA256d(serialize_full(tx))`.

For coinbase, BIP141 defines `wtxid = 0x00..00` (32 zero bytes).

For a non-segwit tx (no marker/flag), `wtxid == txid`.

### Witness merkle commitment

The block's coinbase transaction has an output:

```
OP_RETURN <0x24> 0xaa21a9ed <wtxid_merkle_root>
```

(36 bytes payload: 4-byte magic + 32-byte root)

The witness merkle root is computed over `wtxid` of every transaction in order, with the coinbase entry replaced by 32 zero bytes. The coinbase's witness stack must contain exactly one 32-byte item used as a "witness reserved value" (typically zero) to construct the commitment hash:

```
commitment = SHA256d(witness_root_hash || witness_reserved_value)
```

Validators that don't understand SegWit ignore the OP_RETURN; SegWit validators verify it matches.

### Why two ids?

Reason 1: txid stability. A signature can be rewritten (different valid encoding) without changing the meaning of the tx; pre-SegWit this changed the txid. With SegWit, witnesses are excluded from txid - signatures cannot mutate the chain reference. This unlocks Lightning's pre-signed update transactions.

Reason 2: clean upgrade path. Witness-only data (annex, taproot keypath signature) does not affect old peers who skim only the stripped serialization.

### Where each is used

| Use | Identifier |
|-----|------------|
| `vin[i].outpoint.txid` | `txid` |
| `getrawtransaction <id>` lookup key | both work |
| `mempool` indexing | `txid` (also `wtxid` since 0.21 for `getrawmempool true` mode) |
| Block merkle root (header) | `txid` |
| Block witness root (coinbase OP_RETURN commitment) | `wtxid` |
| `wtxidrelay` peer announcement (BIP339) | `wtxid` |
| Lightning channel "obscured commitment number" XOR | derived per-channel - uses txid of funding |

## Worked example

Coinbase tx of block 481824 (first segwit block):

```
stripped serialization (=> txid)
nVersion = 01000000
vin = 1 input, prevout = 00..00:0xffffffff, scriptSig = "<height + extranonce>"
vout = ...
nLockTime = 00000000

full serialization (=> wtxid)
nVersion = 01000000
marker/flag = 0001
vin same
vout same
witness[0] = 1 item of 32 zero bytes  <-- witness reserved value
nLockTime = 00000000
```

By BIP141 rule, this coinbase's wtxid is **defined as zero**, regardless of the bytes above. The witness root hashes wtxids of all *other* txs, replacing position 0 with zero, then `SHA256d(witness_root || witness_reserved_value)` is checked against the coinbase OP_RETURN.

## Common bugs / anti-patterns

- Computing wtxid for a non-segwit tx by **emitting** marker/flag + empty witness; correct behavior is to skip the marker entirely so wtxid == txid.
- Indexing the mempool only by txid: a peer can replace the witness with a different valid encoding (still consensus-valid by malleability of ECDSA `s` ordering pre-BIP146, less common post-LowS), changing wtxid while keeping txid - causing duplicate inv announcements.
- Using `SHA256d(serialize_full(tx))` as the lookup key in places that should use txid (e.g. when constructing prevout references in a child).
- Forgetting that the wtxid of the coinbase is defined as 0, not the real hash, when constructing or verifying the witness commitment.
- Building a block template without the witness commitment OP_RETURN when any segwit tx is included - block is rejected.
- Mixing endianness: in JSON RPC, txid/wtxid are big-endian hex; in raw tx bytes (outpoint), they are little-endian. Many tools mis-roundtrip.

## References

- BIP141 (witness commitment): https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki
- BIP144 (peer-to-peer encoding): https://github.com/bitcoin/bips/blob/master/bip-0144.mediawiki
- BIP339 (wtxid relay): https://github.com/bitcoin/bips/blob/master/bip-0339.mediawiki
- Bitcoin Core `src/primitives/transaction.cpp` - `GetHash()` vs `GetWitnessHash()`
