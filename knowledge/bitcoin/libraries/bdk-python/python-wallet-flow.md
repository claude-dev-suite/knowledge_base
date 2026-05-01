# bdkpython Wallet Flow - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/bdk-python`.
> Canonical source: https://github.com/bitcoindevkit/bdk-ffi (python bindings)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/bdk-python/SKILL.md

## Concept

`bdkpython` is a UniFFI-generated Python binding around the Rust BDK
crate. The API surface is intentionally close to Rust BDK -- the same
descriptors, the same `Wallet` / `Blockchain` split, the same chain
backends -- but expressed as Python classes that wrap a native shared
library shipped per-platform (`.so` on Linux, `.dylib` on macOS,
`.pyd` on Windows).

This gives you BDK's audited descriptor-first wallet engine and
libsecp256k1-backed signing speed in a `pip install bdkpython`
package. The trade-off is a binary dependency: distribution requires
a wheel for each platform/arch combo, and the Python API sometimes
lags Rust BDK by a release.

## API walkthrough

```python
from bdkpython import (
    Address, AddressIndex, Blockchain, BlockchainConfig,
    Descriptor, EsploraConfig, Network,
    SignOptions, TxBuilder, Wallet, DatabaseConfig, ScriptAmount,
)

NET = Network.TESTNET

descriptor        = Descriptor(
    "wpkh([83b3e5a5/84h/1h/0h]tpub.../0/*)", NET)
change_descriptor = Descriptor(
    "wpkh([83b3e5a5/84h/1h/0h]tpub.../1/*)", NET)

db_cfg = DatabaseConfig.SQLITE(path="./wallet.sqlite")
wallet = Wallet(descriptor, change_descriptor, NET, db_cfg)

cfg = BlockchainConfig.ESPLORA(EsploraConfig(
    base_url="https://mempool.space/testnet/api",
    proxy=None, concurrency=4, stop_gap=20, timeout=None))
blockchain = Blockchain(cfg)

wallet.sync(blockchain, progress=None)
print("balance:", wallet.get_balance().total)
```

## Worked example: build, sign, broadcast

```python
recipient = Address("tb1q...recipient...", NET)

builder = TxBuilder() \
    .add_recipient(recipient.script_pubkey(), 12_345) \
    .fee_rate(2.0)                        # sat/vB
psbt_result = builder.finish(wallet)
psbt = psbt_result.psbt

ok = wallet.sign(psbt, SignOptions())
assert ok, "signing failed"

tx = psbt.extract_tx()
blockchain.broadcast(tx)
print("broadcast:", tx.txid())
```

The `TxBuilder` is fluent; chain `add_recipient`, `fee_rate`,
`enable_rbf`, `add_utxo`, etc. before calling `.finish(wallet)`. Coin
selection is automatic; supply `add_utxo` and `manually_selected_only`
for explicit selection.

## Persistence + restart

```python
# On the next process start, the same descriptor + db path reopens state
wallet2 = Wallet(descriptor, change_descriptor, NET, db_cfg)
wallet2.sync(blockchain, progress=None)   # incremental
```

`DatabaseConfig.MEMORY` is fine for tests. `DatabaseConfig.SQLITE`
is the canonical persistent option; the underlying schema is BDK's,
not user-facing.

## Common pitfalls

- Native wheel availability: when a new Python version ships, you may
  hit pip resolution failures until the binding maintainers rebuild
  wheels. Pin Python and `bdkpython` versions for prod.
- Version skew: `bdkpython==1.x` does not always track the latest Rust
  BDK; check the [bdk-ffi release notes](https://github.com/bitcoindevkit/bdk-ffi/releases)
  before assuming a Rust feature is exposed.
- `sync` vs `full_scan`: full scan rebuilds the keychain from scratch
  (slow, used when restoring); `sync` is incremental. Always full-scan
  on first run with an existing wallet seed.
- Database compatibility: opening a v0 db with a v1 binary fails. Pick
  `DatabaseConfig.SQLITE` for forward-compat over the `SLED` backend
  (deprecated).
- Threading: the underlying Rust types are `Send + Sync` but UniFFI
  surfaces blocking calls. Wrap heavy ops in
  `concurrent.futures.ThreadPoolExecutor` or `asyncio.to_thread`.

## References

- Repo: https://github.com/bitcoindevkit/bdk-ffi
- PyPI: https://pypi.org/project/bdkpython/
- BDK book: https://bitcoindevkit.org/docs
- Companion: [bdk/SKILL.md](../bdk/SKILL.md), [bdk-jvm/SKILL.md](../bdk-jvm/SKILL.md)
