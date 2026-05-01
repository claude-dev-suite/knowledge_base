# Silent Payments Scan Cost Analysis - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/privacy/silent-payments`.
> Canonical source: https://github.com/bitcoin/bips/blob/master/bip-0352.mediawiki
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/privacy/silent-payments/SKILL.md

## Concept

Silent Payments shift cost from chain (no on-chain registration) to
**recipient scanning**. For every block, a scanning client must compute one
ECDH per **eligible-input transaction** and compare derived taproot keys
against the block's taproot outputs. This article analyses bandwidth, CPU,
and storage.

## Walkthrough / mechanics

### What the scanner needs per transaction

For each transaction `T` that has at least one eligible input:

| Item | Size | Source |
|------|------|--------|
| `A_sum` (33 bytes) | 33 B | sum of input pubkeys (need scriptPubKeys + witnesses) |
| smallest outpoint (lex) | 36 B | tx |
| each P2TR output x-only key | 32 B | tx outputs |

A "tweak server" can pre-compute `(input_hash * A_sum)` per transaction and
serve a 33-byte tweak per tx. Then the scanner only needs the per-block list
of (tweak, output keys), not full blocks.

### Three scanning models

1. **Full node scan** (no trust, full block download). \~1 GB/month new
   data. Compute: for each tx with eligible input, 1 EC mult + N hash +
   N EC adds where N = labels + 1.
2. **Tweak server (BIP352 §light client)**. Server emits per-tx
   `(input_hash * A_sum)` tweak and the candidate P2TR outputs only (no
   non-taproot data). Bandwidth ≈ tens of MB/month.
3. **Filter-based with BIP158**. Scanner downloads block filters, queries
   for taproot outputs matching a derived candidate set. This is **not
   directly compatible** with SP because the candidate keys are not known
   ahead of time without the tweak.

### Per-tx cost numbers (rough)

Assume ~3000 txs/block, ~70% have at least one taproot or SegWit input
eligible for SP (≈2100 txs). Per tx: 1 ECDH (≈30 µs on a modern CPU
with libsecp256k1) + 1 SHA-256 + 1 EC add per label (≈few µs). For a
single-label scanner this is ≈70 ms per block. Over a day (~144 blocks)
≈10 s of CPU. With L labels (or k payments per tx) the cost scales linearly
in L.

### Storage

Wallet stores per detected output `(scalar t_k, k, txid:vout)`. \~80 B per
detected payment. The scanner does **not** need to store rejected
candidates.

## Worked example

| Strategy | Bandwidth/day | CPU/day (1 label) | Privacy |
|----------|---------------|-------------------|---------|
| Full block download | ~150 MB | ~10 s | Best (no server queries) |
| Tweak server v1 | ~2-5 MB | ~10 s | Server learns scan-key holder ranges via query patterns |
| Tweak server + OHTTP | ~2-5 MB | ~10 s | Server cannot link queries to IP |
| BIP158 + filter-only | ~30 MB | ~10 s | Compatible only via per-tweak fetch |

A mobile wallet running 24h tweak-server scans + OHTTP can reasonably keep
a Silent Payment receiver up to date with low battery impact.

## Common pitfalls

- **Reorg handling**: when chain reorgs, scanner must roll back detected
  outputs whose containing block disappeared. SP wallets need a
  rollback log keyed by block hash.
- **Trust-minimised tweak servers**: server can selectively withhold tweaks
  to censor specific receivers. Run multiple servers and cross-check, or
  fall back to full scan periodically.
- **Label management**: each label `m` adds an EC add per tx in the inner
  loop. Wallets that publish many "v0/v1/..." labels (e.g. for accounting)
  pay linearly more CPU.
- **Pre-Taproot blocks**: SP outputs are P2TR; blocks before activation
  height (709632 mainnet) need not be scanned at all.

## References

- BIP352 §"Light client support".
- "Silent Payments scanning performance" — bitcoin-dev mailing list 2023.
- libsecp256k1 silentpayments benchmarks.
