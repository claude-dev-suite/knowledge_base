# Block Validation Flow - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/consensus`.
> Canonical source: https://github.com/bitcoin/bitcoin/blob/master/src/validation.cpp
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/consensus/SKILL.md

## Concept

When a Bitcoin Core node receives a candidate block it runs a multi-stage pipeline that separates cheap context-free checks from expensive UTXO-dependent checks. Understanding the order matters: it controls when a peer is banned for sending garbage, when a block is stored on disk, and when the chain tip actually moves. This article traces the path from `ProcessNewBlock` through `ConnectBlock`, with the relevant function names and the order of consensus assertions.

## Walkthrough / mechanics

### Stage 1 - context-free header check (`CheckBlockHeader`)

- Header is exactly 80 bytes.
- `nBits` decodes to a target with the right format (no negative, no overflow).
- `SHA256d(header) <= target` derived from `nBits`.

Failure here -> peer is misbehaving (DoS); no disk write.

### Stage 2 - contextual header check (`ContextualCheckBlockHeader`)

Requires the parent header in the `BlockMap`:

- `nBits` matches the difficulty derived from the previous 2016-block window.
- Block timestamp `> MTP(prev_11_blocks)` (BIP113 - applies to nLockTime since 2016).
- Block timestamp `<= NetworkAdjustedTime() + 2*60*60`.
- Block version meets the floor for active soft forks (e.g. v >= 4 since BIP65).

### Stage 3 - structural block check (`CheckBlock`)

- Merkle root in header matches `MerkleRoot(txids)`.
- No duplicate txids inside the block (CVE-2012-2459 mitigation: also check the merkle tree shape using the `mutated` flag).
- First transaction is coinbase, all others are not.
- Total serialized size and weight: `weight <= 4_000_000`.
- Sigops counted via `GetLegacySigOpCount` and `GetP2SHSigOpCount` once UTXOs are known; after segwit, sigops counted as `weight units / 50` cap of 80,000.

### Stage 4 - contextual block check (`ContextualCheckBlock`)

- BIP34: coinbase scriptSig starts with the block height as a `CScriptNum`.
- BIP141 witness commitment: coinbase has an output `OP_RETURN 0x24 0xaa21a9ed <wtxid_root>` (the magic prefix is `aa21a9ed`).
- Final tx checks (locktime / sequence) using `LockTimePoint = max(MTP, height)`.

### Stage 5 - UTXO validation (`ConnectBlock`)

This is where money is checked.

```
for each tx in block (skipping coinbase):
    fetch each input's prevout from the UTXO set (CCoinsViewCache)
    if any missing:
        if first time we see this block: store as orphan, request parents
        else: invalid (UTXO double-spend within block)
    enforce maturity: coinbase prevouts must be 100+ confirmations old
    enforce BIP68 relative locktime via tx.nSequence
    sum_in  += prevout.value
    sum_out += sum(out.value for out in tx.vout)
    fees += sum_in - sum_out  (must be >= 0)
    queue script verification for each input (parallelisable)
coinbase value check:
    coinbase.vout total <= subsidy(height) + fees
script verification:
    run scriptSig + scriptPubKey + witness with active flags
    flags: P2SH, DERSIG, CLTV, CSV, NULLDUMMY, WITNESS, TAPROOT (per height)
update UTXO set: remove spent, add new
```

The block is then flushed to disk, and `ActivateBestChain` may move the tip if the block has the highest cumulative work.

## Worked example

Hex header for block 100000:
```
01000000  (version 1)
50120119172a610421a6c3011dd330d9df07b63616c2cc1f1cd00200000000  (prev hash, LE)
6657a9252aacd5c0b2940996ecff952228c3067cc38d4885efb5a4ac4247e9f3  (merkle, LE)
67cf2f4c (time)
3e9501a4 (nBits)
b6f3b1de (nonce)
```

Validator: target = `0xa4019500 << (8 * (0x1d - 3))`, hash `SHA256d(header)` little-endian must be `<= target`. After header passes, the body's 4 transactions are merkle-hashed; root must equal `f3e94742...`. ConnectBlock then opens the UTXO view, finds the 3 spent outputs across blocks <100000, verifies each `OP_DUP OP_HASH160 ... OP_EQUALVERIFY OP_CHECKSIG`, sums fees (50 BTC subsidy + small fees == coinbase value).

## Common bugs / anti-patterns

- Verifying the witness merkle commitment before checking the coinbase exists - segfault on empty vtx.
- Using node-local time instead of MTP for BIP65/BIP112 evaluation; consensus failure on reorg through a daylight-saving boundary.
- Counting sigops before contextual checks: a malformed block could OOM the sigop counter.
- Trusting `nBits` from the header without recomputing the expected difficulty; old soft forks (BIP30 at heights 91842, 91880) needed special-casing for duplicate txids.
- Re-running script validation on every reconsideration; Core caches script verification results in `g_scriptExecutionCacheHasher`.
- Forgetting that BIP34 height encoding is **minimal** CScriptNum: a block at height 1000 must encode `0xe803` not `0xe80300`.

## References

- Bitcoin Core: `src/validation.cpp`, `src/consensus/tx_verify.cpp`, `src/script/interpreter.cpp`
- BIP141 (witness commitment): https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki
- BIP34 (height in coinbase): https://github.com/bitcoin/bips/blob/master/bip-0034.mediawiki
- BIP113 (MTP): https://github.com/bitcoin/bips/blob/master/bip-0113.mediawiki
