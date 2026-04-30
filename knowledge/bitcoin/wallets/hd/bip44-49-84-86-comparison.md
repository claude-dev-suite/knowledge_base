# BIP44 vs BIP49 vs BIP84 vs BIP86 - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/wallets/hd`.
> Canonical sources: BIP44 https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki ; BIP49 ; BIP84 ; BIP86
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/wallets/hd/SKILL.md

## Concept

The four "purpose" BIPs define hardened derivation prefixes that bind a derivation
tree to a specific output script type. They exist because the same xprv produces
different addresses under different script descriptors — and there must be a
canonical way for wallets to find each other's UTXOs after a recovery.

| BIP | Purpose | Script | xpub serialization | Address |
|-----|---------|--------|--------------------|---------|
| 44  | `m/44'` | P2PKH                | xpub / tpub      | `1...` / `m,n...` |
| 49  | `m/49'` | P2SH-wrapped P2WPKH  | ypub / upub      | `3...` / `2...`   |
| 84  | `m/84'` | P2WPKH (native v0)   | zpub / vpub      | `bc1q...` / `tb1q...` |
| 86  | `m/86'` | P2TR (single-key)    | xpub (BIP-bound) | `bc1p...` / `tb1p...` |

The differing SLIP-132 prefixes (ypub/zpub) carry no extra data; they are pure
metadata to prevent users from importing a v0-segwit account into a legacy
wallet and seeing zero balance. Modern descriptor wallets use plain `xpub` plus
the `wpkh()`/`tr()` script wrapper to encode intent unambiguously.

## Walkthrough / mechanics

Index `0'` of each tree is the canonical "first account". For one mnemonic, you
can populate all four trees simultaneously — Bitcoin Core does this by default.
The leaf path `account'/change/index` is identical across all four BIPs; only
the purpose hardened step differs.

```
seed
 |- m/44'/0'/0'/0/0  -> SECP key K_legacy
 |- m/49'/0'/0'/0/0  -> SECP key K_p2sh    (different child key!)
 |- m/84'/0'/0'/0/0  -> SECP key K_v0
 |- m/86'/0'/0'/0/0  -> SECP key K_taproot (then BIP86 key tweak applied)
```

Each derivation produces a different secp256k1 key because the hardened steps
mix the parent chain code and index. Therefore the four resulting addresses
are completely unrelated on-chain.

BIP86 adds a key step on top: the leaf pubkey `P` is converted to the taproot
output via `Q = P + int(hashTapTweak(P)) * G` with no script path. Wallets that
forget the tweak will derive the address but be unable to sign.

## Worked example

Using a known test vector (BIP84 abandon-x12 mnemonic):

```bash
# zpub for account 0
xprv=$(bx mnemonic-to-seed "abandon abandon ... about" | bx hd-new)
account_xpub=$(bx hd-private --hard 84 | bx hd-private --hard 0 \
              | bx hd-private --hard 0 | bx hd-to-public)

# Bitcoin Core descriptor form (preferred)
desc="wpkh([73c5da0a/84h/0h/0h]${account_xpub}/0/*)"
bitcoin-cli getdescriptorinfo "$desc"
# returns checksum, e.g. #t9luqynm

bitcoin-cli deriveaddresses "${desc}#t9luqynm" '[0,2]'
# ["bc1qcr8te4kr609gcawutmrza0j4xv80jy8z306fyu",
#  "bc1qnjg0jd8228aq7egyzacy8cys3knf9xvrerkf9g",
#  "bc1q3g3...","..."]
```

Compare to BIP86 same seed:

```bash
desc86="tr([73c5da0a/86h/0h/0h]${account_xpub_86}/0/*)"
bitcoin-cli deriveaddresses "${desc86}#abc12345" '[0,0]'
# ["bc1p5cyxnuxmeuwuvkwfem96lqzszd02n6xdcjrs20cac6yqjjwudpxqkedrcr"]
```

Different prefixes, different keys, different addresses, same seed.

## Common pitfalls

- Hardware wallets often default to BIP44 even when the user wants segwit;
  Ledger Live before v2.x exposed only BIP84 via "Bitcoin (segwit)" account
  type — picking plain "Bitcoin" gave BIP44.
- BIP49 (P2SH-wrapped) is now legacy. New wallets should not generate fresh
  BIP49 addresses; only honor existing ones for backward compatibility.
- Deriving from `xpub` instead of `ypub`/`zpub` is fine when paired with the
  correct script descriptor. It is **not** fine when the wallet auto-selects
  script type from xpub prefix only — verify by inspecting the first address.
- Importing a BIP86 xpub into a BIP84 wallet shows balance = 0 even though
  the seed is correct.
- The coin-type field (`/0'` vs `/1'`) is mandatory for testnet/signet; many
  Linux test scripts forget and accidentally derive mainnet addresses on
  signet.

## References

- BIP32: https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki
- SLIP-132: https://github.com/satoshilabs/slips/blob/master/slip-0132.md
- Descriptors: https://github.com/bitcoin/bitcoin/blob/master/doc/descriptors.md
- Cross-reference: [derivation-path-pitfalls.md](derivation-path-pitfalls.md), [watch-only-flow.md](watch-only-flow.md)
