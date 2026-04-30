# Cryptography - Musig2 - Overview

> Canonical content lives in the dev-suite skill: **`bitcoin/cryptography/musig2`**
> Source link: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/cryptography/musig2/SKILL.md

## What this covers

MuSig2 (BIP327): two-round Schnorr key aggregation for n-of-n multisig. Aggregates n public keys into a single 32-byte x-only pubkey indistinguishable from a single-sig. Used in Taproot cooperative spends. Quick refs: protocol round-trip, key aggregation math, common attacks.

## When to use

This is a Phase A overview - quick orientation only. The full deep-dive
content (concept, code patterns, anti-patterns, common bugs) lives in the
dev-suite `SKILL.md` linked above. Topic-specific Phase B articles
(when present) appear as siblings of this `overview.md` in the same
directory.

## Cross-references

See related skills under [`knowledge/bitcoin/`](../) - protocol,
cryptography, wallets, core, lightning, l2, metaprotocols, privacy,
mining, hardware, infrastructure, testing, libraries.
