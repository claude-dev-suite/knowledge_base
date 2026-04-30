# Cryptography - Schnorr - Overview

> Canonical content lives in the dev-suite skill: **`bitcoin/cryptography/schnorr`**
> Source link: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/cryptography/schnorr/SKILL.md

## What this covers

BIP340 Schnorr signatures over secp256k1: sign/verify, x-only pubkeys, tagged hashes, batch verification, key tweaking. Building block for Taproot, MuSig2, FROST, adaptor signatures, DLCs. Quick refs: signing pseudocode, batch verification, common mistakes.

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
