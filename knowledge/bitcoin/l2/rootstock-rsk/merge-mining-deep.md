# RSK Merge-Mining Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/rootstock-rsk`.
> Canonical source: https://dev.rootstock.io/rsk/architecture/mining/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/rootstock-rsk/SKILL.md

## Concept

Merge-mining (a.k.a. AuxPoW) lets a Bitcoin miner secure a second chain
without dedicating any extra hashrate. Each Bitcoin block can carry a
commitment to one or more auxiliary chain block headers in its coinbase
script; if the Bitcoin block satisfies the auxiliary chain's lower
difficulty, the miner publishes the AuxPoW proof on the auxiliary chain
and earns its reward. RSK uses this design to be 1:1 secured by Bitcoin
hashrate.

## Walkthrough / mechanics

### AuxPoW header structure

Each RSK block header carries:

- Standard EVM-style fields (parent, state root, miner, timestamp, ...).
- `bitcoinMergedMiningHeader`: the parent **Bitcoin** block header.
- `bitcoinMergedMiningCoinbaseTransaction`: serialised Bitcoin coinbase
  with embedded RSK tag.
- `bitcoinMergedMiningMerkleProof`: Merkle path from the coinbase to the
  Bitcoin block's `merkle_root`.

The RSK block hash `H` to be committed is `0x52534B424C4F434B3A` (`RSKBLOCK:`)
followed by `H || target || difficulty_helpers`. The miner places this
40-byte tag inside the Bitcoin coinbase scriptSig.

### Difficulty separation

RSK has its **own** difficulty target, much lower than Bitcoin's. A
Bitcoin block satisfies RSK if `bitcoin_block_hash <= rsk_target`. Since
`bitcoin_target ≤ rsk_target` typically by ~10^4–10^5×, every Bitcoin
block also satisfies RSK; effectively each Bitcoin block can produce
several RSK blocks (RSK average 30 s vs Bitcoin's 10 min).

### Submission flow on RSK

1. Miner discovers a candidate Bitcoin block satisfying RSK target.
2. Miner constructs the AuxPoW proof: Bitcoin header + coinbase tx +
   Merkle path.
3. Submits via JSON-RPC `mnr_submitBitcoinBlock` to an RSK node.
4. RSK verifies:
   - Bitcoin header PoW vs Bitcoin target (or vs RSK target if
     specifically auxiliary-only).
   - Coinbase carries the `RSKBLOCK:` tag with correct RSK block hash.
   - Merkle proof binds the coinbase to the header.
5. If valid, the new RSK block is appended; miner receives RBTC reward
   + RSK transaction fees.

### Pool implementation

A Bitcoin pool that wishes to merge-mine RSK adds RSK as an auxiliary
chain in their stratum work distribution. The pool template generator:

1. Asks RSK node for `mnr_getWork` -> RSK block target + tag bytes.
2. Embeds the tag in the Bitcoin coinbase.
3. Hands work to hashers as usual.
4. On a candidate share that meets RSK difficulty (but not necessarily
   Bitcoin difficulty), the pool submits to RSK only.
5. On a share that meets both, the pool submits to both chains and pays
   the miner extra for RBTC reward.

Pools accounting: ~75 % of Bitcoin hashrate participated in RSK merge
mining as of 2024–2025.

## Worked example

Bitcoin block at height 850 000:

```
Coinbase scriptSig:
  03 <height: 0x0CFA50>      OP push BIP34
  ...
  ... 0x52534B424C4F434B3A   "RSKBLOCK:"
  ...                       <RSK block hash>
  <miner tag>
```

The Bitcoin block hash `0x0000...abcd` <= Bitcoin target -> Bitcoin
miner wins both Bitcoin coinbase reward and one RSK block (because
RSK target is far easier).

Pool also produced 12 share-only candidates this Bitcoin window that hit
RSK target but not Bitcoin target -> 12 RSK-only blocks claimed.

## Common pitfalls

- **Incorrect coinbase tag offset**: the tag must be the **only** RSK
  reference in the coinbase; if multiple, RSK rejects. Pool software
  must place tag deterministically.
- **Difficulty mismatch**: RSK auto-adjusts difficulty per block; if pool
  caches old work, late-submitted blocks may be rejected.
- **Uncle / GHOST**: RSK uses GHOST-like uncle inclusion. Pools must
  submit "uncle" blocks (siblings of the canonical chain) to claim
  partial rewards.
- **Selfish-mining attack**: a pool with > ~25 % Bitcoin hashrate can
  selfishly mine RSK blocks they withhold from broadcast, increasing
  their RSK share. Mitigation: random-seed selection in next-block
  difficulty.
- **Reorg cascade**: a 2-block Bitcoin reorg can reorg ~40 RSK blocks
  due to the timing ratio. RSK confirms peg-outs only after deep depth.

## References

- Sergio Demian Lerner. "RSK White Paper" 2018.
- BIP30/AuxPoW (Namecoin original spec).
- rsknet/dev.rootstock.io merge-mining docs.
