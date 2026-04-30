# Derivation Path Pitfalls - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/wallets/hd`.
> Canonical source: BIP43 https://github.com/bitcoin/bips/blob/master/bip-0043.mediawiki
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/wallets/hd/SKILL.md

## Concept

A derivation path is a sequence of (index, hardened-flag) tuples applied to the
master extended key. Subtle bugs at the path level are responsible for the
majority of "where are my coins" support tickets. They survive because the
math succeeds silently — you always get *some* address back, just not the one
holding your money.

Three classes of pitfall recur:

1. **Hardening-flag mismatch** — `m/84'/0'/0'` vs `m/84/0/0` produce different
   keys with no error.
2. **Prefix confusion** — the wallet derives at `m/0/0'/0'` (the old
   pre-BIP43 path) and you imported expecting `m/84'/0'/0'`.
3. **Off-by-one account index** — last hardened step starts at 0, not 1.

## Walkthrough / mechanics

A path like `m/84'/0'/0'/0/0` parses as five steps:

```
step  index    hardened
1     84       yes   (84 + 2^31 = 0x80000054)
2     0        yes   (0  + 2^31 = 0x80000000)
3     0        yes
4     0        no    (raw 0)
5     0        no    (raw 0)
```

Every BIP32 implementation must serialize hardened indices by adding 0x80000000
before HMAC. A path text parser that ignores the apostrophe (or `h`, or `H`) on
the third step produces a totally different key. Audit your parser by deriving
the BIP32 test vectors before trusting it.

Common alternate notations:
- `m/84'/0'/0'` (apostrophe)
- `m/84h/0h/0h` (lowercase h, used in descriptors)
- `m/84H/0H/0H` (uppercase H, also valid)

Bitcoin Core descriptors require `h` (lowercase). Some hardware wallets emit
`'` only. Both should round-trip without changing the derived key.

## Worked example

Demonstrate the bug on a sample seed:

```python
# Using python-bip32
from bip32 import BIP32

mnemonic = "abandon abandon abandon abandon abandon abandon " \
           "abandon abandon abandon abandon abandon about"
seed = mnemonic_to_seed(mnemonic)
m = BIP32.from_seed(seed)

# Correct BIP84
addr_correct = derive_p2wpkh(m.get_pubkey_from_path("m/84'/0'/0'/0/0"))
# bc1qcr8te4kr609gcawutmrza0j4xv80jy8z306fyu

# Bug 1: lost hardening on coin_type
addr_bug = derive_p2wpkh(m.get_pubkey_from_path("m/84'/0/0'/0/0"))
# bc1q...   (totally different)

# Bug 2: pre-BIP43 path
addr_legacy = derive_p2wpkh(m.get_pubkey_from_path("m/0'/0/0"))
# Some Electrum-imported wallets sit here.
```

Round-trip through descriptors:

```bash
desc="wpkh([73c5da0a/84h/0h/0h]xpub6CV2.../0/*)"
bitcoin-cli getdescriptorinfo "$desc" | jq .descriptor
# wpkh([73c5da0a/84h/0h/0h]xpub6CV2.../0/*)#t9luqynm
```

The `73c5da0a` is the **fingerprint** of the master key (HASH160 of the master
pubkey, first 4 bytes). It lets a wallet recognise whether an xpub belongs to
the seed it has loaded — a critical sanity check during multisig coordination.

## Common pitfalls

- Importing an xpub without the bracketed `[fp/path]` origin prefix into a
  multisig wallet means the wallet cannot prove the xpub came from the same
  seed as the others. BIP380 makes the origin mandatory in wallet descriptors.
- Deriving non-hardened from the master key directly (`m/0/0`) exposes child
  privkeys to anyone who has the parent xpub plus one child privkey
  (BIP32 vulnerability). Always start non-hardened derivation **after** an
  account-level hardened boundary.
- A bug in the early Trezor firmware emitted `/0'` instead of `/0` for change
  outputs. Wallet recovery from such an account requires explicit override.
- Coldcard `multi.json` files store the path as `m/48'/0'/0'/2'`; some
  coordinators expect `m/48h/0h/0h/2h`. Both produce the same address but
  hash mismatch breaks descriptor checksums.
- Mixing testnet `coin_type=1'` with mainnet wallets: Bitcoin Core silently
  derives mainnet addresses from a testnet xpub; the seed and path are valid,
  the address just lives on the wrong chain.

## References

- BIP32 vulnerability discussion: https://bitcoin.stackexchange.com/questions/a/49081
- BIP380 Output Descriptors: https://github.com/bitcoin/bips/blob/master/bip-0380.mediawiki
- python-bip32 test vectors: https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki#test-vectors
- See also: [bip44-49-84-86-comparison.md](bip44-49-84-86-comparison.md), [watch-only-flow.md](watch-only-flow.md)
