# Protocol - Package Relay - Overview

> Canonical content lives in the dev-suite skill: **`bitcoin/protocol/package-relay`**
> Source link: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/package-relay/SKILL.md

## What this covers

BIP331 package relay + accompanying mempool policy: package validation, ancestor/descendant limits, submitpackage RPC, package CPFP, sibling eviction with TRUC v3 (BIP431), ephemeral anchors. Critical for Lightning fee bumping and any multi-tx workflow.

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
