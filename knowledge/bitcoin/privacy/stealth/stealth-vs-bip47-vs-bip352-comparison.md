# Stealth vs BIP47 vs BIP352 - Comparison Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/privacy/stealth`.
> Canonical sources: BIP47 mediawiki, BIP352 mediawiki, dual-key stealth (Peter Todd 2014)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/privacy/stealth/SKILL.md

## Concept

Three families of "static address, fresh outputs" schemes for Bitcoin:

1. **Dual-key stealth addresses** (DKSA, 2014). Pre-BIP47, pre-Taproot.
   Sender publishes an "ephemeral key" in an `OP_RETURN`; receiver scans
   `OP_RETURN`s.
2. **BIP47 PayNyms**. ECDH with pre-shared "payment code". Requires a
   one-time on-chain **notification transaction**.
3. **BIP352 Silent Payments**. ECDH derived from sender's spent inputs;
   no on-chain registration, taproot-only outputs.

## Walkthrough / mechanics

| Feature | DKSA (stealth) | BIP47 | BIP352 (Silent Payments) |
|---------|----------------|-------|--------------------------|
| Address format | base58 prefixed | base58 "PM..." 81 B code | bech32m "sp1q..." 117 chars |
| On-chain registration | none | yes (notification tx) | none |
| Per-payment chain bloat | +OP_RETURN (~40 B) | none after notification | none |
| Output type | P2PKH historically | P2PKH/P2WPKH | P2TR only |
| Eligible inputs | any | any | only certain SegWit/Taproot |
| Receiver scan cost | scan all OP_RETURN | derive next index | per-tx ECDH |
| Sender state | stateless | needs receiver pubkey | stateless |
| Offline receiver? | yes | yes (after notification) | yes |
| Reorg sensitivity | low | medium (notification) | medium |
| Standardised | BIP47-era discussion only | BIP47 (informational) | BIP352 (standards-track) |

### Sender derivation differences

- **DKSA**: sender picks ephemeral `e`, computes `c = H(e * B)` where `B` is
  receiver's "scan key", outputs P2PKH of `Q + c*G` where `Q` is "spend key".
  Includes `e*G` in `OP_RETURN`. Receiver scans for matching `OP_RETURN`s.
- **BIP47**: after notification tx, both sides know each other's payment
  codes; sender derives `S_n = SHA256(s_priv * R_pub_n)` where `R_pub_n` is
  receiver's nth pubkey. Output is P2PKH/P2WPKH with key `R_pub_n + S_n * G`.
- **BIP352**: sender derives shared secret from `a_sum * B_scan` (a_sum =
  sum of eligible input keys) without any per-receiver state.

### Notification-tx cost

Only BIP47 requires it. Cost: ~150 vB to publish a 80-byte OP_RETURN-bearing
tx that the receiver detects via fixed bloom filter. Pays a small fee and
permanently links sender wallet to receiver payment code on chain.

## Worked example

A small business wants to accept donations from 10 000 visitors over 5 years.

| Scheme | One-time cost | Per-donation cost | Cumulative chain bloat |
|--------|---------------|-------------------|------------------------|
| Reused address | 0 | 0 | 0, but **terrible privacy** |
| DKSA | 0 | sender pays ~40 B OP_RETURN | ~400 KB |
| BIP47 | per-sender notification (~150 vB) | 0 | up to 1.5 MB if 10k senders |
| BIP352 | 0 | 0 | 0 |

For a public donation address with thousands of unique senders, BIP352 is
strictly better than BIP47 (no per-sender on-chain notification).

## Common pitfalls

- **DKSA is unmaintained**: no widespread wallet support; Electrum dropped
  it. Treat as historical.
- **BIP47 notification-tx privacy leak**: every sender's notification ties
  their wallet to the receiver's payment code; chain analysis can build a
  graph of "people who paid this PayNym". BIP352 avoids this entirely.
- **BIP352 input-set leak**: the **sender's chosen input set** itself is
  visible. SP does not give sender privacy beyond input clustering.
- **Output-type homogeneity**: BIP352 forces P2TR outputs. If your
  recipient set is unusual (everybody else uses P2WPKH), the SP outputs
  stand out as taproot. Anonymity-set is the global P2TR set.

## References

- BIP47 — Reusable Payment Codes for HD wallets.
- BIP352 — Silent Payments.
- "Stealth addresses" — Peter Todd, 2014 bitcoin-dev posts.
- Samourai PayNym docs (BIP47 in the wild).
