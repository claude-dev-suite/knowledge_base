# python-bitcoinlib RPC and Tx Flow - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/python-bitcoinlib`.
> Canonical source: https://github.com/petertodd/python-bitcoinlib
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/python-bitcoinlib/SKILL.md

## Concept

`python-bitcoinlib` (Peter Todd) is the long-standing low-level Bitcoin
library for Python. It models Bitcoin types directly (`CTransaction`,
`CTxIn`, `CTxOut`, `CScript`, `CBitcoinAddress`) and bundles a JSON-RPC
client (`bitcoin.rpc.Proxy`) that talks to a local `bitcoind`.

The library uses **internal byte order** everywhere (the wire format).
RPC calls and block explorers display txids in **reversed** order; the
helpers `lx()` (string -> bytes, reverse) and `b2lx()` (bytes -> string,
reverse) bridge the two. Forgetting this is the single most common
source of bugs.

`SelectParams("mainnet" | "testnet" | "regtest" | "signet")` is global
state -- pick once at process start.

## Maintenance status (as of September 2026)

The library is dormant. Its last PyPI release is **0.12.2 (3 June 2023)**
and its last `master` commit is **14 March 2025**; `master` still declares
`__version__ = '0.12.2'`. The repo is not archived (38 open issues and 23
open pull requests as of September 2026) but nothing upstream has moved in
eighteen months.

Practical consequences for the flows below:

- **Segwit v0 is the ceiling.** `bitcoin/core/script.py` defines only
  `SIGVERSION_BASE` and `SIGVERSION_WITNESS_V0` -- there is no taproot
  sighash and no `OP_CHECKSIGADD`. `bitcoin/wallet.py` has no P2TR
  address class and `bitcoin/bech32.py` has no bech32m constant, so
  `bc1p...` outputs cannot be parsed or constructed.
- **No PSBT.** The package ships no BIP174 module, so a PSBT handoff to a
  hardware signer has to go through `bitcoind` RPC or another library.
- **RPC baseline is Core v24.0.** The upstream README states
  `bitcoin.rpc` "should work with Bitcoin Core v24.0 or later" and has
  not revised that since, so RPC shape changes in Core majors after v24
  are untested upstream. `Proxy.call()` passes arguments through
  verbatim, so a newer RPC still reaches the node -- it is the typed
  convenience wrappers that carry the risk.

For Taproot, PSBT or descriptor wallets, use `bdkpython` (3.1.0,
9 September 2026). `python-bitcointx` (Simplexum) is the low-level fork
with more taproot script handling, but it is dormant too -- last release
1.1.5, 22 January 2024.

## API walkthrough

```python
from bitcoin import SelectParams
from bitcoin.core import (CMutableTransaction, CMutableTxIn, CMutableTxOut,
                          COutPoint, COIN, lx, b2lx, b2x)
from bitcoin.core.script import CScript, SignatureHash, SIGHASH_ALL, OP_CHECKSIG
from bitcoin.core.scripteval import VerifyScript, SCRIPT_VERIFY_P2SH
from bitcoin.wallet import CBitcoinSecret, P2WPKHBitcoinAddress
from bitcoin.rpc import Proxy

SelectParams("regtest")
rpc = Proxy()                         # reads ~/.bitcoin/bitcoin.conf

info = rpc.call("getblockchaininfo")
print("height:", info["blocks"])

# Key + address
sk = CBitcoinSecret.from_secret_bytes(b"\x01" * 32)
addr = P2WPKHBitcoinAddress.from_pubkey(sk.pub)
print("addr:", str(addr))
```

## Worked example: build, sign, broadcast a P2WPKH spend

```python
from bitcoin import SelectParams
from bitcoin.core import (b2x, lx, COIN,
                          CMutableTransaction, CMutableTxIn,
                          CMutableTxOut, COutPoint)
from bitcoin.core.script import (CScript, SignatureHash,
                                 SIGHASH_ALL, SIGVERSION_WITNESS_V0)
from bitcoin.wallet import CBitcoinSecret, P2WPKHBitcoinAddress
from bitcoin.rpc import Proxy

SelectParams("regtest")
rpc = Proxy()

sk = CBitcoinSecret("cVt4o7BG...")              # WIF
src_addr = P2WPKHBitcoinAddress.from_pubkey(sk.pub)

# Pick a UTXO via RPC
utxos = rpc.call("listunspent", 1, 9999, [str(src_addr)])
u = utxos[0]
prev_value = int(u["amount"] * COIN)

txin = CMutableTxIn(COutPoint(lx(u["txid"]), u["vout"]))
fee = 1000
dest = P2WPKHBitcoinAddress.from_pubkey(sk.pub)  # send-to-self for demo
txout = CMutableTxOut(prev_value - fee, dest.to_scriptPubKey())
tx = CMutableTransaction([txin], [txout])

# Segwit-v0 sighash
script_code = src_addr.to_redeemScript()         # P2PKH-like script
sighash = SignatureHash(script_code, tx, 0, SIGHASH_ALL,
                        amount=prev_value,
                        sigversion=SIGVERSION_WITNESS_V0)
sig = sk.sign(sighash) + bytes([SIGHASH_ALL])
txin.witness = CScript([sig, sk.pub])

raw_hex = b2x(tx.serialize())
txid = rpc.call("sendrawtransaction", raw_hex)
print("broadcast:", txid)
```

`P2WPKHBitcoinAddress.to_redeemScript()` returns the implicit P2PKH-shaped
script used as the BIP143 sighash script code; for P2WSH inputs supply
the actual witness script.

## Common pitfalls

- Byte order: `txid` from RPC is reversed-display; pass through `lx()`
  before placing in `COutPoint`. Reverse with `b2lx(tx.GetTxid())` to
  print.
- `SelectParams` is global; pytest test files that don't call it inherit
  whatever was last set. Either call at top of every test or use a
  module-scoped fixture.
- The `Proxy()` constructor reads `bitcoin.conf` for `rpcuser`/`rpcpassword`
  by default. For CI / non-default datadir, pass
  `service_url="http://user:pass@host:port"` explicitly.
- `bitcoin.rpc.Proxy` returns floats for amounts (BTC); convert to sats
  via `int(round(x * COIN))` to avoid drift.
- The library is low-level: there's no built-in coin selection or fee
  estimation. For wallet-grade behaviour, layer `bdkpython` or
  `bitcoinlib` on top.
- Anything taproot-shaped fails early rather than subtly:
  `CBitcoinAddress` cannot decode a `bc1p...` string and there is no
  witness-v1 sighash to sign with. Route P2TR work to another library
  (as of September 2026).

## References

- Repo: https://github.com/petertodd/python-bitcoinlib
- Examples: https://github.com/petertodd/python-bitcoinlib/tree/master/examples
- Bitcoin Core RPC reference: https://developer.bitcoin.org/reference/rpc/
- Companion: [embit/SKILL.md](../embit/SKILL.md), [bdk-python/SKILL.md](../bdk-python/SKILL.md)
