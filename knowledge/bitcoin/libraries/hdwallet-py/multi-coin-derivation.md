# hdwallet (Python) Multi-Coin Derivation - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/hdwallet-py`.
> Canonical source: https://github.com/hdwallet-io/python-hdwallet
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/hdwallet-py/SKILL.md

## Concept

`hdwallet` is a pure-Python HD wallet derivation library covering 200+
chains (Bitcoin, Ethereum, Cardano, Tron, Litecoin, ...) and the
relevant BIP standards: BIP32 (extended keys), BIP39 (mnemonic), BIP44
(account hierarchy), BIP49 (P2WPKH-in-P2SH), BIP84 (native P2WPKH),
BIP86 (Taproot), BIP141 (segwit semantics). Non-BIP hierarchies -
Cardano/CIP1852, Electrum-V1/V2, Monero - are covered too.

It is **derivation-only**: it generates keys + addresses but doesn't
sign transactions, manage UTXOs, or talk to the network. The typical
use case is generating addresses across many chains for an exchange
deposit system, building HD test vectors, or extracting one specific
key from a backup.

## Version note: v2 vs v3

All code below targets **v3** (v3.0.0, November 2024; latest release
v3.6.1, August 2025 - checked September 2026). v3 was a rewrite with
no backward compatibility. The v2 idiom

```python
# v2 ONLY (`pip install 'hdwallet<3'`, last release 2.2.1, Dec 2022)
from hdwallet import HDWallet
from hdwallet.symbols import BTC
HDWallet(symbol=BTC).from_mnemonic(mnemonic).from_path("m/84'/0'/0'/0/0")
```

raises `TypeError` on v3: the constructor is now
`HDWallet(cryptocurrency, hd=..., network=..., address=..., **kwargs)`
and `symbol` is no longer a parameter. Note that `hdwallet/symbols.py`
still ships in 3.6.1, so the *import* succeeds - the break shows up one
line later, which makes the failure easy to misread.

## API walkthrough

```python
from hdwallet import HDWallet
from hdwallet.cryptocurrencies import Bitcoin, Litecoin, Ethereum
from hdwallet.hds import BIP44HD, BIP84HD
from hdwallet.mnemonics import BIP39Mnemonic
from hdwallet.derivations import BIP44Derivation, BIP84Derivation, CHANGES

mnemonic = BIP39Mnemonic(mnemonic=(
    "abandon abandon abandon abandon abandon abandon "
    "abandon abandon abandon abandon abandon about"
))

btc = HDWallet(
    cryptocurrency=Bitcoin, hd=BIP84HD, network=Bitcoin.NETWORKS.MAINNET
).from_mnemonic(mnemonic).from_derivation(
    BIP84Derivation(coin_type=Bitcoin.COIN_TYPE, account=0,
                    change=CHANGES.EXTERNAL_CHAIN, address=0)
)
print("BTC path:", btc.path(), "addr:", btc.address())   # m/84'/0'/0'/0/0, bc1q...

ltc = HDWallet(
    cryptocurrency=Litecoin, hd=BIP84HD, network=Litecoin.NETWORKS.MAINNET
).from_mnemonic(mnemonic).from_derivation(
    BIP84Derivation(coin_type=Litecoin.COIN_TYPE)
)
print("LTC addr:", ltc.address())                        # m/84'/2'/0'/0/0, ltc1...

eth = HDWallet(
    cryptocurrency=Ethereum, hd=BIP44HD, network=Ethereum.NETWORKS.MAINNET
).from_mnemonic(mnemonic).from_derivation(
    BIP44Derivation(coin_type=Ethereum.COIN_TYPE)
)
print("ETH addr:", eth.address())                        # m/44'/60'/0'/0/0, 0x... via Keccak
```

`address()` needs no argument: BIP49/84/86 force an encoding
(P2WPKH-in-P2SH / P2WPKH / P2TR), while BIP32/BIP44 fall back to
`Cryptocurrency.DEFAULT_ADDRESS` - P2PKH for Bitcoin and Litecoin, but
`Ethereum` (Keccak) for Ethereum, as the ETH line above shows. Pass
`address("P2TR")` to override, within the set the chain declares in
`Cryptocurrency.ADDRESSES`.

## Worked example: bulk-derive 50 BTC receive addresses

v3 derivations take an index **range** - `address=(start, end)`, or the
string `"0-49"` - so one `from_derivation()` covers the whole batch and
`dumps()` returns one entry per index:

```python
from hdwallet import HDWallet
from hdwallet.cryptocurrencies import Bitcoin
from hdwallet.hds import BIP84HD
from hdwallet.mnemonics import BIP39Mnemonic
from hdwallet.derivations import BIP84Derivation, CHANGES

def derive_range(mnemonic: str, account: int, change: str,
                 start: int, count: int) -> list[dict]:
    hdwallet = HDWallet(
        cryptocurrency=Bitcoin, hd=BIP84HD, network=Bitcoin.NETWORKS.MAINNET
    ).from_mnemonic(
        BIP39Mnemonic(mnemonic=mnemonic)
    ).from_derivation(
        BIP84Derivation(
            coin_type=Bitcoin.COIN_TYPE, account=account, change=change,
            address=(start, start + count - 1)
        )
    )
    return [
        {"path": d["at"]["path"], "address": d["address"],
         "pubkey": d["public_key"]}
        for d in hdwallet.dumps(exclude={"root", "indexes"})
    ]

addrs = derive_range(mnemonic="abandon ... about", account=0,
                     change=CHANGES.EXTERNAL_CHAIN, start=0, count=50)
for a in addrs[:3]:
    print(a)
```

For Taproot (BIP86) swap in `BIP86HD` + `BIP86Derivation`; the paths
become `m/86'/0'/0'/0/i` and `address()` yields `bc1p...`.

## BIP85 sub-seeds: not supported here

BIP85 derives child mnemonics from a parent seed - useful for
generating distinct wallets for distinct purposes from one backup.
`hdwallet` does **not** implement it: there is no BIP85 module or
method in either the 2.2.1 or the 3.6.1 source tree (checked September
2026). Reach for a dedicated BIP85 implementation, or a signer that
exposes BIP85 itself, instead.

## Common pitfalls

- Server-side mnemonic generation: don't. Generate keys on a HW signer
  or air-gapped machine. `hdwallet`'s random source is the standard
  `os.urandom`, but a network-connected server is a bad place to mint
  keys.
- Path defaults differ per chain; ETH uses `m/44'/60'/0'/0/i`, BTC
  segwit uses `m/84'/0'/0'/0/i`, BTC Taproot uses `m/86'/0'/0'/0/i`.
  `Cryptocurrency.DEFAULT_PATH` exposes the chain default (BIP44 for
  Bitcoin), but an explicit `BIP84Derivation(...)` is safer for clarity.
- The HD class and the derivation must agree, and the type check only
  catches one direction. `BIP84HD.from_derivation()` raises
  `DerivationError` on a `BIP44Derivation`, but `BIP44Derivation` is
  the base class of the BIP49/84/86 ones, so `BIP44HD` accepts a
  `BIP84Derivation` without complaint: you get the `m/84'/...` key
  encoded as P2PKH - a valid `1...` address at a native-segwit path.
- In a `CustomDerivation(path=...)` string, v3 marks hardened depths
  with `'` only - `h`/`H` is not accepted and `m/84h/0h/...` dies in
  `int()`. The same parser does accept an index range per depth, so
  `m/84'/0'/0'/0/0-49` is a legal 50-address path.
- `from_mnemonic()` rejects a bare `str`. It wants an `IMnemonic`, i.e.
  `BIP39Mnemonic(mnemonic="...")`, and rejects mnemonic classes the
  chain does not list in `Cryptocurrency.MNEMONICS`.
- The v2 per-type accessors `p2pkh_address()` / `p2wpkh_address()` are
  gone; v3 has a single `address(...)`.
- Memory: derived `HDWallet` objects retain the seed in memory. Wipe
  references with `del w` and consider running on an air-gapped box.
- Dependency hygiene: `master` raised the PyNaCl floor to `>=1.6.2`
  citing CVE-2025-69277 (January 2026), but no tagged release carries
  that bump as of September 2026 - pin PyNaCl yourself if you use the
  Ed25519 chains.

## References

- Repo: https://github.com/hdwallet-io/python-hdwallet
- Docs: https://hdwallet.readthedocs.io
- BIP32: https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki
- BIP44: https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki
- BIP85 (not implemented by `hdwallet`): https://github.com/bitcoin/bips/blob/master/bip-0085.mediawiki
- Companion: [../../wallets/hd/SKILL.md](../../wallets/hd/SKILL.md), [../../cryptography/bip32/SKILL.md](../../cryptography/bip32/SKILL.md)
