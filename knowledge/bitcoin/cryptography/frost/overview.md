# Cryptography - Frost - Overview

> Canonical content lives in the dev-suite skill: **`bitcoin/cryptography/frost`**
> Source link: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/cryptography/frost/SKILL.md

## What this covers

FROST (Flexible Round-Optimised Schnorr Threshold): t-of-n threshold signing on secp256k1. n participants share a key via Distributed Key Generation (DKG) or trusted dealer; any t can produce a Schnorr signature. Differs from MuSig2 (which is n-of-n).

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
