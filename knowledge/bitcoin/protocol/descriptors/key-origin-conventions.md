# Key Origin Conventions in Descriptors - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/descriptors`.
> Canonical source: BIP380 (Output Script Descriptors), BIP44/49/84/86 (derivation purposes)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/descriptors/SKILL.md

## Concept

Key origin metadata in descriptors - the `[fingerprint/path]` prefix
before an xpub - is what distinguishes a portable, coordinator-friendly
descriptor from an opaque "trust me, this xpub maps to those addresses"
declaration. The skill states that key origin is mandatory for
multisig; this article explains the BIP380 grammar, the master
fingerprint computation, the purpose-by-script-type conventions, and
why some wallets accept descriptors without origin while strict ones
refuse.

## Walkthrough / mechanics

Grammar (BIP380):

```
KEY        = ORIGIN? KEYDATA
ORIGIN     = "[" FINGERPRINT KEYORIGIN_PATH "]"
FINGERPRINT = 8 hex chars
KEYORIGIN_PATH = ("/" CHILD)*
CHILD      = NUMBER ("h" | "H" | "'")?
KEYDATA    = HEX_PUBKEY | XPUB | XPRV | WIF
            ("/" CHILD)* ("/*")?            # post-xpub derivation
```

The fingerprint is the first 4 bytes of `hash160(pubkey)` of the
**master** (root) key. This is constant for a wallet seed; it lets
coordinators identify "which device knows this key" without exposing
the master pubkey.

```
master_seed
   |
master_xprv (depth=0)        --> fingerprint = hash160(master_xprv.pub)[:4]
   |
   m/48'/0'/0'/2'              <-- KEYORIGIN_PATH for the [fp/...] prefix
   |
xpub at this point             <-- KEYDATA: the xpub being shared
   |
   /0/*                         <-- post-xpub derivation, the "ranged" part
```

**Conventional purposes:**

| Purpose | Script type | Address | Derivation path |
|---------|-------------|---------|-----------------|
| 44'    | P2PKH         | `1...` | `m/44'/coin'/account'/<0;1>/*` |
| 49'    | P2SH-P2WPKH   | `3...` | `m/49'/coin'/account'/<0;1>/*` |
| 84'    | P2WPKH        | `bc1q` | `m/84'/coin'/account'/<0;1>/*` |
| 86'    | P2TR          | `bc1p` | `m/86'/coin'/account'/<0;1>/*` |
| 48'    | multisig wrapper | varies | `m/48'/coin'/account'/script_type'/<0;1>/*` |

For BIP48 multisig, `script_type'` is `0` (P2SH legacy multisig),
`1` (P2SH-P2WSH), or `2` (P2WSH). For Taproot multisig under BIP48,
the convention is still solidifying; some wallets use `m/48'/0'/0'/2'`
and tag descriptor with `tr(...multi_a...)`.

`coin` is `0'` mainnet and `1'` testnet (BIP44).

## Worked example

Single-sig P2WPKH descriptor from a hardware wallet:

```
[d34db33f/84'/0'/0']xpub6CUGRUonZSQ4TWtTMmzXdrXDtypWKiKrhko4egpiMZbpiaQL2jkwSB1icqYh2cfDfVxdx4df189oLKnC5fSwqPfgyP3hooxujYzAu3fDVmz/<0;1>/*
```

- `d34db33f` -> master fingerprint of the wallet's seed.
- `/84'/0'/0'` -> BIP84 mainnet account 0 path.
- The xpub is at depth 3.
- `<0;1>/*` is multipath (BIP389): receive (0) and change (1) chains.

The signer (Coldcard, Ledger) MUST verify on-device that the xpub it
was asked to sign with is reached from its master via the declared
path. The fingerprint alone is not authentication; the path lets the
device re-derive and compare.

For 2-of-3 multisig:

```
wsh(sortedmulti(2,
  [d34db33f/48'/0'/0'/2']xpubAlice.../<0;1>/*,
  [b15ec0de/48'/0'/0'/2']xpubBob.../<0;1>/*,
  [c0ffee00/48'/0'/0'/2']xpubCarol.../<0;1>/*))#chk
```

Each cosigner's hardware wallet receives this descriptor. On signing,
the wallet finds its own fingerprint (`d34db33f` for Alice's device),
re-derives the path locally, confirms the resulting xpub matches the
descriptor, and only then signs.

Computing fingerprint from a master seed (Python with python-bip32):

```python
from bip32 import BIP32
import hashlib

seed = bytes.fromhex("000102030405060708090a0b0c0d0e0f")
master = BIP32.from_seed(seed)
master_pub = master.pubkey
fp = hashlib.new('ripemd160', hashlib.sha256(master_pub).digest()).digest()[:4]
print(fp.hex())   # 3442193e for the BIP32 test seed
```

## Common bugs / pitfalls

1. **Wrong hardening notation.** Descriptors accept `'`, `h`, or `H`
   for hardened paths. Some older wallets only parse `'`. When sharing
   descriptors, prefer `h` (URL-safe, no quote escaping).
2. **Fingerprint of intermediate xpub instead of master.** The
   fingerprint MUST be of the master; using an intermediate xpub's
   fingerprint causes signers to fail re-derivation.
3. **Hardened path on shared xpub.** Once you share an xpub at
   `m/48'/0'/0'/2'`, you cannot derive further hardened children
   from xpub alone (xpub doesn't carry the chain code in a way that
   permits hardened derivation). Hardened components MUST be in the
   key-origin path; only unhardened in the post-xpub part.
4. **Checksum reuse across modifications.** Editing any character of
   a descriptor invalidates its checksum. Tools must recompute via
   `getdescriptorinfo`. Sharing a descriptor with a stale checksum
   causes Bitcoin Core to reject `importdescriptors`.
5. **Coin path mismatch on regtest.** Regtest uses coin path `1'`
   (testnet semantics). Mixing `m/84'/0'/...` on regtest produces
   addresses that don't match the device's UI display.
6. **Origin path that doesn't match the actual xpub depth.** If
   declaring `[fp/84'/0'/0']` but the xpub is actually at depth 4,
   re-derivation fails. The path length equals xpub.depth.

## References

- BIP380: https://github.com/bitcoin/bips/blob/master/bip-0380.mediawiki
- BIP32: https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki
- BIP44/49/84/86: https://github.com/bitcoin/bips
- BIP48: https://github.com/bitcoin/bips/blob/master/bip-0048.mediawiki
- bdk descriptor parser: https://docs.rs/bdk/latest/bdk/descriptor/
