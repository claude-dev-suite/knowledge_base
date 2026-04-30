# Lightning - Replacement Cycling - Overview

> Canonical content lives in the dev-suite skill: **`bitcoin/lightning/replacement-cycling`**
> Source link: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/replacement-cycling/SKILL.md

## What this covers

Replacement cycling attack on Lightning (Riard 2023): exploits BIP125 rule 5 to indefinitely delay an honest HTLC-timeout, costing the victim. Mitigated by TRUC v3 + ephemeral anchors.

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
