# BDK Wallet Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/bdk`.
> Canonical source: https://github.com/bitcoindevkit/bdk_wallet
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/bdk/SKILL.md

## Concept

BDK (Bitcoin Dev Kit) sits on top of `rust-bitcoin` + `rust-miniscript`
and provides a descriptor-first wallet engine: you give it an output
descriptor, it owns the keychain, UTXO set, coin selection, fee
estimation, and PSBT building. Chain data flows in via swappable
backends (Esplora, Electrum, Bitcoin Core RPC, BIP157/158 compact
filters). The 1.0 release split the crate into `bdk_wallet`,
`bdk_chain`, and per-backend crates (`bdk_esplora`, `bdk_electrum`,
`bdk_bitcoind_rpc`).

The split went further in April 2025: `bdk_wallet` moved to its own
repository, `github.com/bitcoindevkit/bdk_wallet`, leaving the chain
crates in `github.com/bitcoindevkit/bdk`. As of September 2026 the
current versions on crates.io are `bdk_wallet` 3.1.0 (June 2026),
`bdk_esplora` 0.22.2, `bdk_electrum` 0.24.0 and `bdk_bitcoind_rpc`
0.22.0; the backends all share `bdk_core` 0.6.x, which `bdk_wallet`
pulls in through `bdk_chain` 0.23.x. MSRV for `bdk_wallet` 3.1.0 is
Rust 1.85.0. The release line ran 1.0 (December 2024) -> 2.0 (June
2025) -> 3.0 (April 2026) -> 3.1 (June 2026), with a 2.4.0 backport
(April 2026) for teams staying on 2.x.

Persistence is pluggable -- `create_wallet_no_persist` for tests,
SQLite/file persisters for prod. The same descriptor across runs gives
the same scriptPubKey set.

## API walkthrough

```rust
use bdk_wallet::{KeychainKind, SignOptions, Wallet};
use bdk_wallet::bitcoin::{Amount, Network};
use bdk_esplora::{esplora_client, EsploraAsyncExt};

const STOP_GAP: usize = 5;
const PARALLEL: usize = 4;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let ext = "wpkh([83b3e5a5/84h/1h/0h]tpubD6.../0/*)";
    let int = "wpkh([83b3e5a5/84h/1h/0h]tpubD6.../1/*)";

    let mut wallet = Wallet::create(ext, int)
        .network(Network::Testnet)
        .create_wallet_no_persist()?;

    let client = esplora_client::Builder::new("https://mempool.space/testnet/api")
        .build_async()?;
    let request = wallet.start_full_scan().build();
    let update = client.full_scan(request, STOP_GAP, PARALLEL).await?;
    wallet.apply_update(update)?;

    println!("balance: {}", wallet.balance().total());
    let next = wallet.reveal_next_address(KeychainKind::External);
    println!("next addr: {}", next.address);
    Ok(())
}
```

## Worked example: build, sign, broadcast a payment

```rust
use bdk_wallet::bitcoin::Address;
use std::str::FromStr;

let recipient = Address::from_str("tb1q...")?
    .require_network(Network::Testnet)?;

let mut builder = wallet.build_tx();
builder
    .add_recipient(recipient.script_pubkey(), Amount::from_sat(10_000))
    .fee_rate(bdk_wallet::bitcoin::FeeRate::from_sat_per_vb(2).unwrap());
let mut psbt = builder.finish()?;

let finalized = wallet.sign(&mut psbt, SignOptions::default())?;
assert!(finalized);

let tx = psbt.extract_tx()?;
client.broadcast(&tx).await?;
println!("broadcast: {}", tx.compute_txid());
```

The wallet maintains internal/external keychains automatically; coin
selection defaults to BnB with a fall-back. Always call `apply_update`
before building a tx -- otherwise UTXOs are stale.

## 3.0 additions (April 2026)

Two API additions in 3.0 change how long-running wallets are written.

**Persistent UTXO locking.** `wallet.lock_outpoint(outpoint)` marks a
UTXO as unavailable to coin selection; `unlock_outpoint` reverses it.
Both only stage a change -- you must persist the staged changeset for
the lock to survive a restart. Backed by a new SQLite table,
`bdk_wallet_locked_outpoints`, added by a backwards-compatible
migration.

**Structured wallet events.** `apply_update_events(update)` is the
event-returning twin of `apply_update`: it returns
`Result<Vec<WalletEvent>, CannotConnectError>`, where `WalletEvent` is
`ChainTipChanged`, `TxConfirmed`, `TxUnconfirmed`, `TxReplaced` or
`TxDropped`. This replaces diffing the balance by hand to work out
what a sync actually did.

3.0 also adopts `NetworkKind` throughout, adds Caravan wallet format
import/export, and ships a migration utility for SQLite databases
created before 1.0. 3.1 (June 2026) adds `Wallet::sign_with_signers`,
which takes a caller-supplied list of signers instead of the wallet's
own `SignersContainer`, and `LoadParams::two_path_descriptor`.

## Common pitfalls

- 0.x to 1.0 migration: API renamed extensively; `Wallet::new` -> 
  `Wallet::create`/`Wallet::load`, `database` -> `connection`/persister.
- Snippets pinned to 1.x are now two majors behind. The builder shape
  above survives into 3.1.0, but 3.0 (April 2026) moved the codebase to
  `NetworkKind`, so code that threads `Network` through its own helpers
  needs review on the way up.
- `apply_update` must be called or the wallet sees no UTXOs -- silent
  empty balance.
- Persister mismatch: opening a wallet stored by version X with version
  Y will return `LoadError`. Migrate or recreate.
- Blocking vs async crates: `bdk_esplora` exposes both `EsploraExt`
  and `EsploraAsyncExt`; pick one.
- `STOP_GAP` too low loses funds outside the gap window. BIP44 says 20;
  match what the user-side wallet used to fund the descriptor.

## References

- Wallet repo: https://github.com/bitcoindevkit/bdk_wallet
- Chain crates repo: https://github.com/bitcoindevkit/bdk
- Book: https://bookofbdk.com (examples updated to the 3.0 API)
- 3.0.0 release notes: https://github.com/bitcoindevkit/bdk_wallet/releases/tag/v3.0.0
- 3.1.0 release notes: https://github.com/bitcoindevkit/bdk_wallet/releases/tag/v3.1.0
- Companion: [rust-bitcoin/SKILL.md](../rust-bitcoin/SKILL.md), [bdk-python/SKILL.md](../bdk-python/SKILL.md)
