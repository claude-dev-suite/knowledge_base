# Hardware - Hwi - Overview

> Canonical content lives in the dev-suite skill: **`bitcoin/hardware/hwi`**
> Source link: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/hardware/hwi/SKILL.md

## What this covers

HWI (Hardware Wallet Interface): standardized Python API across HW vendors (Trezor, Ledger, Coldcard, BitBox02, Jade, etc.). Used by Bitcoin Core, Sparrow, Specter for cross-vendor signing.

Status as of September 2026: latest release is 3.2.0 (10 February 2026). HWI is winding down - [issue #850](https://github.com/bitcoin-core/HWI/issues/850) (18 August 2026) announced maintenance-only status, no new features or device support, and eventual archival once a replacement is ready; [BHWI](https://github.com/wizardsardine/bhwi) (Rust, Wizardsardine, WIP) is named as the successor.

## When to use

This is a Phase A overview - quick orientation only. The full deep-dive
content (concept, code patterns, anti-patterns, common bugs) lives in the
dev-suite `SKILL.md` linked above. Topic-specific Phase B articles
(when present) appear as siblings of this `overview.md` in the same
directory.

## Cross-references

See related skills under [`knowledge/bitcoin/`](../../) - protocol,
cryptography, wallets, core, lightning, l2, metaprotocols, privacy,
mining, hardware, infrastructure, testing, libraries.
