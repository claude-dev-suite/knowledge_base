# Block Template Construction - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/mining/pow`.
> Canonical source: BIP 22 (getblocktemplate), BIP 23, Bitcoin Core `miner.cpp`
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/mining/pow/SKILL.md

## Concept

A block template is a fully-formed candidate block missing only a valid `nonce`
(and possibly an `extranonce` rotation). Whoever constructs the template
chooses which transactions get mined. Under Stratum V1 the pool builds the
template; under Stratum V2 + a Template Provider, the miner can build it
locally from their own bitcoind mempool.

A template contains:

- An 80-byte header skeleton (version, prev_hash, merkle_root placeholder,
  timestamp, nbits, nonce=0).
- An ordered list of transactions, with the coinbase as tx[0].
- Subsidy + total fees that the coinbase pays out.

## Walkthrough / mechanics

### Step 1 - Pick parent block

`prev_block_hash` is the tip of the best chain. Any new chain tip invalidates
the template (you must rebuild). Bitcoin Core sends ZMQ `hashblock`
notifications so miners can react within milliseconds.

### Step 2 - Select transactions from mempool

Bitcoin Core's `BlockAssembler` greedily picks transactions ordered by
ancestor fee rate (fee per virtual byte including unconfirmed parents) until
the block is full. The constraints:

- `weight <= 4_000_000` (post-SegWit).
- `sigops_cost <= 80_000`.
- All ancestors must be included if a child is.

### Step 3 - Build coinbase

The coinbase is a special transaction with one input whose `prevout` is null
and whose `scriptSig` contains arbitrary bytes (subject to BIP 34's height
prefix). After BIP 34 the scriptSig MUST start with the block height as a
minimal-push CScriptNum:

```
scriptSig = <push_height> <extranonce_bytes> <pool_tag>
```

The pool reserves a region of the scriptSig for the miner's `extranonce1`
(per-session) and `extranonce2` (per-share rotation). When the miner bumps
extranonce2, the coinbase TXID changes, and the merkle root must be
recomputed.

### Step 4 - Compute merkle root

Given an ordered list of TXIDs `[T0, T1, ..., Tn]`, build a binary tree by
pairwise SHA256d:

```
level0 = [T0, T1, T2, T3]            # if odd, duplicate the last
level1 = [H(T0||T1), H(T2||T3)]
level2 = [H(H(T0||T1) || H(T2||T3))] # this is the merkle_root
```

For Stratum V1 efficiency, the pool sends only the coinbase parts plus the
merkle branches needed to recompute the root once the miner finalises the
coinbase. The miner does not need the full tx list, just the branch.

### Step 5 - Validate witness commitment

For SegWit blocks, the coinbase tx must contain an OP_RETURN output with the
witness commitment:

```
0x6a 0x24 0xaa21a9ed <SHA256d(witness_root_hash || witness_reserved)>
```

`witness_reserved` is 32 zero bytes by convention. The witness commitment
binds all witness data to the block.

## Worked example

A typical `getblocktemplate` response (truncated):

```json
{
  "version": 0x20000000,
  "previousblockhash": "00000000000000000003a1f2...",
  "transactions": [
    { "data": "020000000001...", "txid": "abc...", "fee": 12500, "weight": 565 }
  ],
  "coinbaseaux": { "flags": "" },
  "coinbasevalue": 312501250000,
  "target": "0000000000000000000311e3b0...",
  "mintime": 1714000000,
  "curtime": 1714003600,
  "bits": "17034219",
  "height": 893500,
  "default_witness_commitment": "6a24aa21a9ed..."
}
```

`coinbasevalue = subsidy + fees = 3.125 BTC + 0.000125 BTC = 312_501_250_000 sats`.

A pool builds the coinbase, splits it at the extranonce position, hashes all
non-coinbase TXIDs into merkle branches, and sends `coinbase_part_1`,
`coinbase_part_2`, and `merkle_branches` over `mining.notify`.

## Common pitfalls

- **Stale templates** - if a new block arrives during template build, you
  must restart. Failing to subscribe to ZMQ `hashblock` causes wasted work.
- **Sigop limit** - high-sigop scripts (multisig) eat the 80k budget fast.
- **BIP 34 height prefix** - missing the height push makes the coinbase
  invalid; the block is rejected.
- **Witness commitment placement** - any output ordering is fine, but the
  commitment must be present and computed over the actual witness data.
- **Extranonce overlap** - if pool's extranonce1 plus miner's extranonce2
  size doesn't match what was advertised in `mining.subscribe`, the
  resulting coinbase is malformed and the block (if found) is rejected.

## References

- BIP 22: getblocktemplate <https://github.com/bitcoin/bips/blob/master/bip-0022.mediawiki>
- BIP 141: Witness commitment <https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki>
- Bitcoin Core `src/node/miner.cpp` - BlockAssembler implementation
- Stratum V1 spec <https://reference.cash/mining/stratum-protocol>
