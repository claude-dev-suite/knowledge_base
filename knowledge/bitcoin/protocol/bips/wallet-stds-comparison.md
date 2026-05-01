# Wallet Standards Comparison - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/bips`.
> Canonical source: BIP32, 39, 43, 44, 49, 84, 85, 86, 174, 322, 329, 388
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/bips/SKILL.md

## Concept

A "modern" Bitcoin wallet is a stack of interoperability BIPs covering
seed entropy, key derivation, address types, signing, label portability,
and message authentication. The skill lists the BIPs by topic; this
article compares overlapping/competing standards and shows which combos
yield interoperable wallets in 2025-2026.

## Walkthrough / mechanics

**Seed and entropy:**

| BIP | Purpose | Status |
|-----|---------|--------|
| 39  | Mnemonic seed phrase (12/15/18/21/24 words) | Universal |
| 85  | Deterministic child seeds from BIP32 master | Common in Coldcard, ColdCard, BTCPay |
| 39 + Electrum-style | Electrum legacy seed | Legacy only, not recommended for new |

**Key derivation:**

| BIP | Path | Address type |
|-----|------|--------------|
| 44  | `m/44'/0'/0'/0/*` | P2PKH `1...` |
| 49  | `m/49'/0'/0'/0/*` | P2SH-P2WPKH `3...` |
| 84  | `m/84'/0'/0'/0/*` | P2WPKH `bc1q...` |
| 86  | `m/86'/0'/0'/0/*` | P2TR `bc1p...` |
| 48  | `m/48'/0'/0'/{0,1,2}'/...` | Multisig (legacy/wrapped/native) |

Most wallets ship 84 + 86 by default; 49 retained for compatibility.
44 only offered for legacy import.

**Output description:**

| BIP | Purpose |
|-----|---------|
| 380 | Output descriptor language (`wpkh`, `wsh`, `tr`, ...) |
| 381 | Tapscript descriptor extensions |
| 385 | Multi-key descriptor (`combo`) |
| 388 | Wallet policies (Ledger-style) - structured descriptors with placeholders |
| 389 | Multipath descriptors `<0;1>` |

BIP388 is unique: it splits the descriptor into a TEMPLATE
(wpkh(@0/<0;1>/*)) and KEY MAP (@0 -> [fp/...]xpub...). Hardware
wallets can review and store the template; sender devices fill in
placeholder keys. Used by Ledger and slowly adopted elsewhere.

**Transaction signing:**

| BIP | Purpose |
|-----|---------|
| 174 | PSBT v0 |
| 370 | PSBT v2 |
| 371 | Taproot fields in PSBT |

**Hardware wallet integration:**

| BIP | Purpose |
|-----|---------|
| 388 | Wallet policies (signing review) |
| 174/371 | PSBT vehicle |
| 32  | Derivation root |

**Message signing:**

| BIP | Range | Status |
|-----|-------|--------|
| 137 | P2PKH only | Legacy, ambiguous prefix |
| 322 | All output types | Modern, recommended |

**Labels and portability:**

| BIP | Purpose |
|-----|---------|
| 329 | Wallet label export (JSON-lines: addr -> label, txid -> note, ...) |
| 47  | Reusable Payment Codes (PayNyms) |
| 352 | Silent Payments |

**URI / payment requests:**

| BIP | Purpose |
|-----|---------|
| 21  | `bitcoin:address?amount=...&label=...` |
| 78  | PayJoin (URL parameter `pj=`) |
| 21  | LNURL fallback via `lightning=` parameter |

## Worked example

**A modern wallet stack (2025-2026 conventions):**

```
Seed:                BIP39 (24 words preferred for entropy)
Master derivation:   BIP32
Receive (single-sig): BIP86 -> Taproot, m/86'/0'/0'/<0;1>/*
Receive (multisig):  BIP48 -> P2WSH or Taproot multi_a, m/48'/0'/0'/2'/<0;1>/*
Descriptor format:   BIP380 + BIP389 (multipath)
Hardware policy:     BIP388 (with placeholders)
Signing transport:   BIP174 + BIP371 (PSBT v0 with Taproot fields)
Message signing:     BIP322 (modern)
Address sharing:     BIP21 URI
Label backup:        BIP329
```

**Cross-wallet compatibility matrix (rough, late 2025):**

| Wallet | BIP86 (Taproot) | BIP371 PSBT | BIP322 verify | BIP329 export | BIP388 |
|--------|-----------------|-------------|---------------|---------------|--------|
| Sparrow | yes | yes | yes | yes | partial |
| Specter | yes | yes | partial | yes | no |
| Coldcard | yes | yes | sign-only | no | partial |
| Trezor | yes | yes | sign-only | no | no |
| Ledger | yes | yes | yes | no | yes (native) |
| BlueWallet | yes | partial | partial | no | no |
| Bitcoin Core | yes | yes | partial | no | no |

Multisig coordination tools (Caravan, Nunchuk, Liana) typically
support BIP380+388+389 plus BIP174/371. Some don't yet cover BIP389
multipath, treating it as separate descriptors.

## Common bugs / pitfalls

1. **Mixing seed standards.** Importing a BIP39 phrase into an
   Electrum-mnemonic wallet produces different keys silently. Each
   standard has its own seed-to-master derivation.
2. **Wrong purpose path.** A wallet hardcoded for `m/84'/0'/0'/0/*`
   that imports an xpub originally derived for `m/49'/0'/0'/0/*`
   computes wrong addresses. Always rely on key-origin metadata in
   the descriptor.
3. **BIP137 used where BIP322 needed.** A SegWit-only address can't
   be signed with BIP137. Some exchanges still demand BIP137; users
   then derive a P2PKH from the same key (semantically wrong but
   common).
4. **PSBT v0 vs v2 mismatch.** A coordinator using v2 sends to a
   signer that only parses v0. Result: "unknown global field". Most
   2024+ tools accept both; old hardware may not.
5. **Multipath descriptor exported but signer doesn't support
   `<0;1>`.** Must split to two descriptors at coordinator level.
6. **BIP329 import conflicts.** When merging label files from
   multiple wallets, address collisions must be resolved (which
   label wins). BIP329 doesn't specify; tools choose latest-write
   or first-write.

## References

- BIP repo: https://github.com/bitcoin/bips
- BIP329 reference impl: https://github.com/seedsigner/seedsigner
- BIP388: https://github.com/bitcoin/bips/blob/master/bip-0388.mediawiki
- Wallets Recovery (cross-wallet seed paths): https://walletsrecovery.org/
