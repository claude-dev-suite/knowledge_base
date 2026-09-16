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

1. **Full node scan** (no trust, full block download). Mainnet blocks
   averaged \~1.59 MB over the 30 days to 15 September 2026 (heights
   962,703-967,258, mempool.space block-size series), i.e. \~230 MB/day
   and \~6.9 GB/month of new block data. Compute: for each tx with
   eligible input, 1 EC mult + N hash + N EC adds where N = labels + 1.
2. **Tweak server (BIP352 Appendix A)**. Server emits per-tx
   `(input_hash * A_sum)` tweak and the candidate P2TR outputs only (no
   non-taproot data). BIP352 Appendix A measures the tweak stream at
   7-12 kB per block (30-50 MB/month) on January 2023 - July 2024 data,
   against a \~100 kB/block (\~450 MB/month) ceiling if every tx in a
   \~3,500-tx block were SP-eligible.
3. **Filter-based with BIP158**. Scanner downloads block filters, queries
   for taproot outputs matching a derived candidate set. This is **not
   directly compatible** with SP because the candidate keys are not known
   ahead of time without the tweak.

### Deployed servers (September 2026)

The models above are no longer hypothetical. The **BIP0352 Index Server
Specification** (`silent-payments/BIP0352-index-server-specification`,
still marked WIP, last pushed 11 May 2026) standardises three stacks:

| Stack | What the client gives the server | Trust |
|-------|----------------------------------|-------|
| Remote Scanner (ephemeral) | scan key pair + spend pubkey, for the session | Server is trusted; rotate the wallet on leaving |
| Tweak Server (anonymous) | nothing identifying; pulls tweaks per block or block range | Server cannot be fully trusted; data must be cross-checkable |
| My Scanner (personalised) | registers scan key pair + spend pubkey + birthdate on its own infra | Self-hosted |

The spec names three indexers already in use by wallets:
`setavenger/blindbit-oracle` (Go), Cake Wallet's `blockstream-electrs`
fork, and Sparrow's `frigate`. **Frigate** is the Remote Scanner case:
it scans server-side with ephemeral in-RAM client keys and returns hits
over an extension to the Electrum JSON-RPC protocol, indexing from
Taproot activation, with optional CUDA / Metal / OpenCL backends.

### Streaming vs filters: the 2026 benchmark

Rob Segers benchmarked **BlindBit Oracle v2** - a BIP352 indexing server
that drops filters entirely and instead streams per-output data (txid,
tweak, and an 8-byte output prefix) - against BIP158 filters and
taproot-only filters. On unsampled block data from Taproot activation
through block 965,089 (255,434 blocks), the streaming approach downloads
about **2.1x** as many bytes as a taproot-only filter plus the raw tweak
data a filter client still needs, and that comparison excludes the full
block a filter client must fetch on every match. What the extra bytes buy
is no false positives and no per-match block fetches. Optech #422
(11 September 2026). For the filter side of the same comparison see
`../../protocol/p2p/compact-filters-bip157-158.md`, section "Alternative
designs and measurements (2026)".

### Per-tx cost numbers (rough)

Mainnet blocks carried \~4,500 transactions on average across 90 blocks
sampled between heights 962,186 and 967,250 (mempool.space, 15 September
2026); BIP352 Appendix A works from "a typical bitcoin block of ~3500
txs". Only a small slice needs an ECDH: over 1,550 txs sampled between
heights 965,400 and 967,250 (mempool.space, 16 September 2026), ~85-93%
had at least one SP-eligible input (the spread is whether P2SH-P2WPKH is
counted), but only ~7-9% also had a taproot output, and BIP352 requires
both. That second figure counts taproot outputs at creation where the BIP
wants an *unspent* one, so it is an upper bound. ~7-9% of ~4,500 txs is
~10-13 kB of tweak data per block, consistent with Appendix A's 7-12 kB
on 2023-2024 data. Per scanned tx: 1 ECDH + 1 SHA-256 + 1 EC add per
label.

The per-operation timings that follow are **order-of-magnitude estimates,
not measurements**: libsecp256k1 does ship a scan benchmark
(`src/modules/silentpayments/bench_impl.h`, in the `silentpayments`
module since v0.8.0, August 2026, exposing `scan_nomatch` and a
23,250-P2TR-output `scan_worstcase`), but no published result from it is
cited here. At an assumed ≈30 µs per ECDH, a single-label scan of one
block is tens of milliseconds and a day (~144 blocks) is single-digit
seconds of CPU. With L labels (or k payments per tx) the cost scales
linearly in L.

### Storage

Wallet stores per detected output `(scalar t_k, k, txid:vout)`. \~80 B per
detected payment. The scanner does **not** need to store rejected
candidates.

## Worked example

Bandwidth figures are measured or cited where the source is given; CPU is
an estimate (see above).

| Strategy | Bandwidth/day | CPU/day (1 label) | Privacy |
|----------|---------------|-------------------|---------|
| Full block download | ~230 MB (mempool.space, Sept 2026) | seconds (est.) | Best (no server queries) |
| Tweak server | ~1-1.7 MB (BIP352 App. A, 2024 data) | seconds (est.) | Server learns scan-key holder ranges via query patterns |
| Tweak server + OHTTP | ~1-1.7 MB | seconds (est.) | Server cannot link queries to IP |
| BIP158 filters + tweak fetch | filters + tweak stream | seconds (est.) | Full block fetched on every match, false positives included |
| Remote scanner (Frigate) | hits only | none on client | Server holds the scan key for the session |

A mobile wallet running 24h tweak-server scans + OHTTP can reasonably keep
a Silent Payment receiver up to date with low battery impact.

## Common pitfalls

- **Reorg handling**: when chain reorgs, scanner must roll back detected
  outputs whose containing block disappeared. SP wallets need a
  rollback log keyed by block hash.
- **Trust-minimised tweak servers**: server can selectively withhold tweaks
  to censor specific receivers. Run multiple servers and cross-check, or
  fall back to full scan periodically. This omission is **undetectable by
  the client** and silently loses the receiver money; as of September 2026
  it is unsolved. BlindBit Oracle v2 publishes per-block commitments over
  the sorted tweak set and checkpoints them to nostr every six hours,
  which makes omissions attributable after the fact but does not prevent
  them - clients should still fetch the full block on a match (Optech
  #422, 11 September 2026).
- **Label management**: each label `m` adds an EC add per tx in the inner
  loop. Wallets that publish many "v0/v1/..." labels (e.g. for accounting)
  pay linearly more CPU.
- **Pre-Taproot blocks**: SP outputs are P2TR; blocks before activation
  height (709632 mainnet) need not be scanned at all.

## References

- BIP352 Appendix A, "Light Client Support": https://github.com/bitcoin/bips/blob/master/bip-0352.mediawiki#appendix-a-light-client-support
- Underlying data for Appendix A (Jan 2023 - Jul 2024): https://github.com/josibake/bitcoin-data-analysis/blob/main/notebooks/silent-payments-light-client-data.ipynb
- BIP0352 Index Server Specification (WIP): https://github.com/silent-payments/BIP0352-index-server-specification
- Frigate Electrum server (Sparrow): https://github.com/sparrowwallet/frigate
- BlindBit Oracle (indexing server): https://github.com/setavenger/blindbit-oracle
- BlindBit Oracle v2 vs BIP158 benchmark (Optech #422, 2026-09-11): https://bitcoinops.org/en/newsletters/2026/09/11/
- libsecp256k1 `silentpayments` scan benchmark: https://github.com/bitcoin-core/secp256k1/blob/master/src/modules/silentpayments/bench_impl.h
- Mainnet block sizes, 30 days to 2026-09-15: https://mempool.space/api/v1/mining/blocks/sizes-weights/1m
