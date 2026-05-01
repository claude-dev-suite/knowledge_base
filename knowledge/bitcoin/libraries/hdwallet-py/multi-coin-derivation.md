# hdwallet (Python) Multi-Coin Derivation - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/hdwallet-py`.
> Canonical source: https://github.com/meherett/python-hdwallet
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/hdwallet-py/SKILL.md

## Concept

`hdwallet` is a pure-Python HD wallet derivation library covering 100+
chains (Bitcoin, Ethereum, Cardano, Tron, Litecoin, ...) and the
relevant BIP standards: BIP32 (extended keys), BIP39 (mnemonic), BIP44
(account hierarchy), BIP49 (P2WPKH-in-P2SH), BIP84 (native P2WPKH),
BIP86 (Taproot), BIP85 (sub-seed derivation).

It is **derivation-only**: it generates keys + addresses but doesn't
sign transactions, manage UTXOs, or talk to the network. The typical
use case is generating addresses across many chains for an exchange
deposit system, building HD test vectors, or extracting one specific
key from a backup.

## API walkthrough

```python
from hdwallet import HDWallet
from hdwallet.symbols import BTC, LTC, ETH, BCH

mnemonic = ("abandon abandon abandon abandon abandon abandon "
            "abandon abandon abandon abandon abandon about")

btc = HDWallet(symbol=BTC, use_default_path=False)
btc.from_mnemonic(mnemonic=mnemonic, passphrase="")
btc.from_path("m/84'/0'/0'/0/0")        # native segwit
print("BTC addr:", btc.p2wpkh_address())

ltc = HDWallet(symbol=LTC).from_mnemonic(mnemonic=mnemonic).from_path("m/84'/2'/0'/0/0")
print("LTC addr:", ltc.p2wpkh_address())

eth = HDWallet(symbol=ETH).from_mnemonic(mnemonic=mnemonic).from_path("m/44'/60'/0'/0/0")
print("ETH addr:", eth.p2pkh_address())   # 0x... via Keccak
```

## Worked example: bulk-derive 50 BTC receive addresses

```python
from hdwallet import HDWallet
from hdwallet.symbols import BTC

def derive_range(mnemonic: str, account: int, change: int,
                 start: int, count: int) -> list[dict]:
    out = []
    for i in range(start, start + count):
        path = f"m/84'/0'/{account}'/{change}/{i}"
        w = HDWallet(symbol=BTC).from_mnemonic(mnemonic).from_path(path)
        out.append({
            "path":    path,
            "address": w.p2wpkh_address(),
            "pubkey":  w.public_key(),
        })
    return out

addrs = derive_range(mnemonic="abandon ... about", account=0,
                     change=0, start=0, count=50)
for a in addrs[:3]:
    print(a)
```

For Taproot (BIP86), use `m/86'/0'/0'/0/i` and `w.p2tr_address()`.

## BIP85 sub-seeds

BIP85 derives child mnemonics from a parent seed -- useful for
generating distinct wallets for distinct purposes from one backup:

```python
from hdwallet import HDWallet
from hdwallet.symbols import BTC

parent = HDWallet(symbol=BTC).from_mnemonic("abandon ... about")
# Derive a new 12-word mnemonic at index 0
child_mnemonic = parent.bip85(language="english", words=12, index=0)
print("derived mnemonic:", child_mnemonic)
```

## Common pitfalls

- Server-side mnemonic generation: don't. Generate keys on a HW signer
  or air-gapped machine. `hdwallet`'s random source is the standard
  `os.urandom`, but a network-connected server is a bad place to mint
  keys.
- Path defaults differ per chain; ETH uses `m/44'/60'/0'/0/i`, BTC
  segwit uses `m/84'/0'/0'/0/i`, BTC Taproot uses `m/86'/0'/0'/0/i`.
  `use_default_path=True` picks the chain-default; explicit
  `from_path(...)` is safer for clarity.
- Hardened indices need either `'` or `h` suffix; mixing forms in the
  same path string breaks parsing.
- Output address types: a single derived key has both `p2pkh_address()`
  and `p2wpkh_address()` methods. Calling the wrong one for your
  descriptor produces an unrelated address.
- Memory: derived `HDWallet` objects retain the seed in memory. Wipe
  references with `del w` and consider running on an air-gapped box.

## References

- Repo: https://github.com/meherett/python-hdwallet
- BIP32: https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki
- BIP44: https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki
- BIP85: https://github.com/bitcoin/bips/blob/master/bip-0085.mediawiki
- Companion: [../../wallets/hd/SKILL.md](../../wallets/hd/SKILL.md), [../../cryptography/bip32/SKILL.md](../../cryptography/bip32/SKILL.md)
