# bitcoinlib (Python) Wallet Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/bitcoinlib-py`.
> Canonical source: https://github.com/1200wd/bitcoinlib
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/bitcoinlib-py/SKILL.md

## Concept

`bitcoinlib` (1200WD) is a high-level Python wallet library: one
`Wallet` object owns keys, accounts, addresses, UTXO database, and
chain backend. It targets multiple chains (Bitcoin, Litecoin, Dogecoin,
Dash, BCH, others) using the same API, persists to a local SQLite by
default, and abstracts away the descriptor / PSBT / sighash machinery.

This makes it the fastest path to a working Python wallet prototype --
significantly higher-level than `python-bitcoinlib` (which is just
primitives) and roughly comparable to `bdkpython` in API surface
(though pure Python rather than Rust-backed).

## API walkthrough

```python
from bitcoinlib.wallets import Wallet

# Create a new HD wallet (BIP44 by default; pass witness_type for segwit)
w = Wallet.create(
    "DemoWallet",
    network="bitcoin",
    witness_type="segwit",        # P2WPKH addresses
    db_uri=None,                  # default: ~/.bitcoinlib/database/bitcoinlib.sqlite
)

# Receive
addr = w.get_key().address
print("receive at:", addr)

# Sync UTXOs from default service backend (mempool.space, blockstream, ...)
w.utxos_update()
print("balance (sat):", w.balance())

# Transactions history
for t in w.transactions():
    print(t.txid, t.value, t.confirmations)
```

`Wallet.create` writes to a SQLite DB; subsequent runs use
`Wallet("DemoWallet")` to reopen. The `db_uri` argument accepts any
SQLAlchemy URL for Postgres / MySQL.

## Worked example: send a payment with custom fee

```python
from bitcoinlib.wallets import Wallet
from bitcoinlib.services.services import Service

w = Wallet("DemoWallet")
w.utxos_update()

# Estimate fee in sat/vB via service backend
svc = Service(network="bitcoin")
fee_per_kb = svc.estimatefee(blocks=3)        # sat/kvB

dest = "bc1q...recipient..."
amount_sat = 12_345

# Builds, signs, persists, broadcasts in one call
tx = w.send_to(dest, amount_sat, fee=fee_per_kb)
print("txid:", tx.txid)
print("hex:",  tx.raw_hex())

# If you want offline signing instead, build without broadcast
tx2 = w.send_to(dest, amount_sat, fee=fee_per_kb, offline=True)
# tx2.raw_hex() can now be exchanged with another signer
```

## Multi-currency

```python
from bitcoinlib.wallets import Wallet

ltc = Wallet.create("LtcDemo", network="litecoin",
                    witness_type="segwit")
print(ltc.get_key().address)        # ltc1...

doge = Wallet.create("DogeDemo", network="dogecoin")
print(doge.get_key().address)       # D...
```

The same code path works for ~20 chains; defaults (BIP44 path, address
prefixes, fee semantics) come from the per-network config.

## Common pitfalls

- The default SQLite DB is single-writer. Concurrent processes opening
  the same wallet -> `OperationalError: database is locked`. For
  multi-tenant servers, switch to Postgres via `db_uri`.
- `utxos_update` polls public APIs (mempool.space, blockstream.info,
  blockcypher). Heavy use will hit rate limits; configure a private
  Esplora or run `bitcoind` with the `bitcoind` provider.
- The library is pure Python -- signing performance is many times slower
  than `bdkpython` or `python-bitcoinlib` (which use libsecp256k1 via
  bindings). Fine for prototypes, marginal for high-volume servers.
- Encrypted DB mode (`password="..."` argument) protects key material
  at rest but is no substitute for HW-wallet signing for any serious
  amount of value.
- Multi-currency means many chains share the wallet code; less polished
  chains (e.g., Dash) have edge cases the Bitcoin path doesn't trigger.

## References

- Repo: https://github.com/1200wd/bitcoinlib
- Docs: https://bitcoinlib.readthedocs.io
- Service backends: https://bitcoinlib.readthedocs.io/en/latest/source/_static/manuals.service-providers.html
- Companion: [python-bitcoinlib/SKILL.md](../python-bitcoinlib/SKILL.md), [bdk-python/SKILL.md](../bdk-python/SKILL.md)
