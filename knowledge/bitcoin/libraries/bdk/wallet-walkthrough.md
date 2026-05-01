# BDK Wallet Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/bdk`.
> Canonical source: https://github.com/bitcoindevkit/bdk
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

## Common pitfalls

- 0.x to 1.0 migration: API renamed extensively; `Wallet::new` -> 
  `Wallet::create`/`Wallet::load`, `database` -> `connection`/persister.
- `apply_update` must be called or the wallet sees no UTXOs -- silent
  empty balance.
- Persister mismatch: opening a wallet stored by version X with version
  Y will return `LoadError`. Migrate or recreate.
- Blocking vs async crates: `bdk_esplora` exposes both `EsploraExt`
  and `EsploraAsyncExt`; pick one.
- `STOP_GAP` too low loses funds outside the gap window. BIP44 says 20;
  match what the user-side wallet used to fund the descriptor.

## References

- Repo: https://github.com/bitcoindevkit/bdk
- Book: https://bitcoindevkit.org/docs
- 1.0 release notes: https://github.com/bitcoindevkit/bdk/blob/master/CHANGELOG.md
- Companion: [rust-bitcoin/SKILL.md](../rust-bitcoin/SKILL.md), [bdk-python/SKILL.md](../bdk-python/SKILL.md)
