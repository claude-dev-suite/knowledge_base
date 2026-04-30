# Protocol - Psbt - Overview

> Canonical content lives in the dev-suite skill: **`bitcoin/protocol/psbt`**
> Source link: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/psbt/SKILL.md

## What this covers

Partially Signed Bitcoin Transaction format: BIP174 (v0), BIP370 (v2), BIP371 (Taproot fields). Roles (Creator, Updater, Signer, Combiner, Finalizer, Extractor), required fields per input/output type, Taproot-specific fields (tap_key_sig, tap_script_sig, tap_leaf_script, tap_bip32_derivation), magic bytes, separators.

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
