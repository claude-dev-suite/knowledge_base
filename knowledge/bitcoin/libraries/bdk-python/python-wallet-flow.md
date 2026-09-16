# bdkpython Wallet Flow - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/bdk-python`.
> Canonical source: https://github.com/bitcoindevkit/bdk-python (bdk-ffi as a submodule)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/bdk-python/SKILL.md

## Concept

`bdkpython` is a UniFFI-generated Python binding around the Rust BDK
crate. The API surface is intentionally close to Rust BDK -- the same
descriptors, the same wallet/chain-client split, the same chain
backends -- but expressed as Python classes that wrap a native shared
library shipped per-platform and loaded through `ctypes`
(`bdkpython/libbdkffi.so` on Linux, `libbdkffi.dylib` on macOS,
`bdkffi.dll` on Windows in the 3.1.0 wheels -- a plain cdylib, not a
Python extension module).

This gives you BDK's audited descriptor-first wallet engine and
libsecp256k1-backed signing speed in a `pip install bdkpython`
package. The trade-off is a binary dependency: distribution requires
a wheel for each platform/arch combo, and each binding release pins
one `bdk_wallet` version rather than tracking the crate continuously.

Everything below targets **bdkpython 3.1.0** (PyPI, 2026-09-09), which
wraps bdk-ffi 3.1.0 (tagged 2026-09-11) over `bdk_wallet` 3.1.0 and
requires Python >= 3.10. The pre-1.0 API this article previously
documented (`DatabaseConfig`, `BlockchainConfig`, `Blockchain`,
`wallet.sync(blockchain, progress)`) was removed in the 1.0 redesign;
the mapping is in "Migrating from the pre-1.0 API" below.

## API walkthrough

```python
from bdkpython import (
    Address, Amount, Descriptor, EsploraClient, FeeRate,
    KeychainKind, Network, NetworkKind, Persister,
    TxBuilder, Wallet,
)

NET = Network.TESTNET

descriptor        = Descriptor(
    "wpkh([83b3e5a5/84h/1h/0h]tpub.../0/*)", NetworkKind.TEST)
change_descriptor = Descriptor(
    "wpkh([83b3e5a5/84h/1h/0h]tpub.../1/*)", NetworkKind.TEST)

persister = Persister.new_sqlite("./wallet.sqlite")
wallet = Wallet(descriptor, change_descriptor, NET, persister)

client = EsploraClient("https://mempool.space/testnet/api")

# Three steps: the wallet builds a request, the client executes it,
# the wallet applies the resulting Update and stages a ChangeSet.
request = wallet.start_full_scan().build()
update  = client.full_scan(request, stop_gap=20, parallel_requests=4)
wallet.apply_update(update)
wallet.persist(persister)

print("balance:", wallet.balance().total.to_sat())
print("address:", wallet.reveal_next_address(KeychainKind.EXTERNAL).address)
```

The wallet no longer owns a chain backend. `EsploraClient` and
`ElectrumClient` are standalone clients that take a request and return
an `Update`; `apply_update` is the only way chain data enters the
wallet, and `persist(persister)` is the only thing that writes it to
disk.

## Worked example: build, sign, broadcast

```python
recipient = Address("tb1q...recipient...", NET)

psbt = TxBuilder() \
    .add_recipient(recipient.script_pubkey(), Amount.from_sat(12_345)) \
    .fee_rate(FeeRate.from_sat_per_vb(2)) \
    .finish(wallet)

ok = wallet.sign(psbt)                    # sign_options is optional
assert ok, "signing failed"

tx = psbt.extract_tx()
client.broadcast(tx)
print("broadcast:", tx.compute_txid())
```

`finish()` returns the `Psbt` directly -- there is no result wrapper
to unpack. Amounts and fee rates are typed: `add_recipient` takes an
`Amount`, `fee_rate` takes a `FeeRate`, and bare ints/floats are
rejected. `SignOptions` has no defaults either, so omit it (as above)
unless you intend to set every field.

The `TxBuilder` is fluent; chain `add_recipient`, `fee_rate`,
`add_utxo`, `coin_selection`, etc. before calling `.finish(wallet)`.
Coin selection is automatic; supply `add_utxo` and
`manually_selected_only` for explicit selection, or choose an
algorithm with
`coin_selection(CoinSelectionAlgorithm.BRANCH_AND_BOUND)`. There is no
`enable_rbf` -- RBF signalling is the default.

## Persistence + restart

```python
# On the next process start, reopen the same SQLite file with load().
# load() takes no network argument -- it comes from the ChangeSet.
persister = Persister.new_sqlite("./wallet.sqlite")
wallet2 = Wallet.load(descriptor, change_descriptor, persister)

request = wallet2.start_sync_with_revealed_spks().build()   # incremental
wallet2.apply_update(client.sync(request, parallel_requests=4))
wallet2.persist(persister)
```

`Persister.new_in_memory()` is fine for tests. `Persister.new_sqlite`
is the canonical persistent option; the underlying schema is BDK's,
not user-facing. `Persister.custom(...)` backs the store with your own
`Persistence` implementation.

## Common pitfalls

- Native wheel availability: when a new Python version ships, you may
  hit pip resolution failures until the binding maintainers rebuild
  wheels. Pin Python and `bdkpython` versions for prod. 3.1.0 declares
  `Requires-Python >=3.10` and ships cp310-cp313 wheels for manylinux
  x86_64, macOS 11+ (arm64 and x86_64) and win_amd64.
- Version pinning: each binding release pins one `bdk_wallet` version
  (bdk-ffi 3.1.0 pins `bdk_wallet` 3.1.0, September 2026) rather than
  tracking the crate continuously; check the
  [bdk-ffi release notes](https://github.com/bitcoindevkit/bdk-ffi/releases)
  before assuming a Rust feature is exposed.
- `sync` vs `full_scan`: full scan rebuilds the keychain from scratch
  (slow, used when restoring); `sync` -- built from
  `start_sync_with_revealed_spks()` -- only re-checks already-revealed
  scripts. Always full-scan on first run with an existing wallet seed.
- Nothing hits disk until `wallet.persist(persister)`. `apply_update`
  only stages a `ChangeSet` (inspect it with `staged()` /
  `take_staged()`), so a process that syncs and exits without
  persisting loses the work.
- Database compatibility: pre-v1 SQLite wallets do not open directly.
  `Persister.get_pre_v1_wallet_keychains()` (added in bdk-ffi 3.0.0)
  exists to recover their keychains for migration. The `SLED` backend
  was removed along with the rest of the 0.x persistence layer.
- Threading: the underlying Rust types are `Send + Sync` but UniFFI
  surfaces blocking calls. Wrap heavy ops in
  `concurrent.futures.ThreadPoolExecutor` or `asyncio.to_thread`.

## What 3.x added

bdk-ffi 3.0.0 (June 2026) added, among others:

- `NetworkKind` -- and `Descriptor` / `DescriptorSecretKey`
  constructors now *require* it instead of a `Network` (breaking).
- Locked-outpoint APIs on `Wallet` (`lock_outpoint`,
  `unlock_outpoint`, `list_locked_outpoints`, `is_outpoint_locked`),
  with the locks persisted in the `ChangeSet`.
- `apply_unconfirmed_txs_events` and `apply_evicted_txs_events`,
  joining `apply_update_events` (which shipped earlier, in bdk-ffi
  2.3.0, February 2026). All three return a list of `WalletEvent`
  (`CHAIN_TIP_CHANGED`, `TX_CONFIRMED`, `TX_UNCONFIRMED`,
  `TX_REPLACED`, `TX_DROPPED`).
- `start_full_scan_at` / `start_sync_with_revealed_spks_at` and
  `Wallet.checkpoints()`.

bdk-ffi 3.1.0 (September 2026) then exposed `Wallet.keychains()`,
TxBuilder coin-selection choice, two-path descriptor wallet loading,
and `Wallet` create/load params objects.

## Migrating from the pre-1.0 API

| 0.x (removed) | 3.x |
|---------------|-----|
| `DatabaseConfig.SQLITE` / `.MEMORY` / `SLED` | `Persister.new_sqlite(path)` / `Persister.new_in_memory()` |
| `BlockchainConfig.ESPLORA(EsploraConfig(...))` + `Blockchain(cfg)` | `EsploraClient(url)` |
| `wallet.sync(blockchain, progress)` | `start_sync_with_revealed_spks().build()`, `client.sync(...)`, `wallet.apply_update(update)` |
| `wallet.get_balance()` | `wallet.balance()` (fields are `Amount`; use `.to_sat()`) |
| `wallet.get_address(AddressIndex.NEW())` | `wallet.reveal_next_address(KeychainKind.EXTERNAL)` |
| `builder.finish(wallet).psbt` | `builder.finish(wallet)` returns the `Psbt` |
| `builder.enable_rbf()` | removed; RBF signalling is the default |
| `Descriptor(str, Network)` | `Descriptor(str, NetworkKind)` (bdk-ffi 3.0.0) |
| `address.as_string()` | `str(address)` |
| `tx.txid()` | `tx.compute_txid()` |

The last 0.x release was bdkpython 0.32.1 (February 2025); the
intervening majors are 1.x (March 2025), 2.x (July 2025) and 3.x
(June 2026).

## References

- Repo: https://github.com/bitcoindevkit/bdk-python
- Rust FFI layer: https://github.com/bitcoindevkit/bdk-ffi
- PyPI: https://pypi.org/project/bdkpython/
- Book: https://bookofbdk.com -- Python snippets live under
  Cookbook > Bindings; `bitcoindevkit.org/docs` 404s as of
  September 2026 (bitcoindevkit.org is now the foundation/blog site)
- Python API reference: https://bitcoindevkit.github.io/bdk-python/
- Companion: [bdk/SKILL.md](../bdk/SKILL.md), [bdk-jvm/SKILL.md](../bdk-jvm/SKILL.md)
