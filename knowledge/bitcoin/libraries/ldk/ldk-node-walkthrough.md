# LDK-Node Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/ldk`.
> Canonical source: https://github.com/lightningdevkit/ldk-node
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/ldk/SKILL.md

## Concept

LDK is a modular Lightning library: `lightning` is the protocol core,
`lightning-net-tokio` the transport, `lightning-persister` the file IO,
`lightning-background-processor` the task driver, `lightning-block-sync`
or `lightning-transaction-sync` the chain feeder. Wiring them yourself
gives total control but is hundreds of lines.

`ldk-node` is the opinionated batteries-included crate: it picks
sensible defaults (Esplora chain source, sqlite persister, BDK wallet
underneath, Tokio runtime), exposes a high-level `Node` API, and is
the basis of the Swift / Kotlin / JS bindings. Use it for mobile
wallets and apps; drop down to raw `lightning` only when ldk-node's
choices don't fit.

## API walkthrough

```rust
use ldk_node::{Builder, Network};
use ldk_node::bitcoin::secp256k1::PublicKey;
use ldk_node::lightning::ln::msgs::SocketAddress;
use ldk_node::lightning_invoice::Bolt11Invoice;
use std::str::FromStr;

fn main() -> anyhow::Result<()> {
    let node = Builder::new()
        .set_network(Network::Testnet)
        .set_chain_source_esplora(
            "https://mempool.space/testnet/api".to_string(), None)
        .set_storage_dir_path("./ldk-data".to_string())
        .build()?;

    node.start()?;
    println!("node id: {}", node.node_id());

    // Connect to a peer
    let peer_pubkey = PublicKey::from_str("02abc...")?;
    let peer_addr = SocketAddress::from_str("1.2.3.4:9735")?;
    node.connect(peer_pubkey, peer_addr, true)?;

    // Receive
    let invoice = node.bolt11_payment().receive(
        100_000,                     // amount msat
        &"coffee".into(), 3600)?;
    println!("invoice: {}", invoice);

    // Pay
    let to_pay = Bolt11Invoice::from_str("lnbc...")?;
    let payment_id = node.bolt11_payment().send(&to_pay, None)?;
    println!("payment id: {:?}", payment_id);

    node.stop()?;
    Ok(())
}
```

## Worked example: open a channel and wait for funding

```rust
use ldk_node::UserChannelId;

let user_id: UserChannelId = node.open_channel(
    peer_pubkey,
    peer_addr,
    1_000_000,            // channel value sat
    Some(0),              // push msat
    None,                 // channel config: defaults
    false,                // announce: private channel
)?;

// Poll events until ChannelReady
loop {
    match node.next_event() {
        Some(ldk_node::Event::ChannelReady { user_channel_id, .. })
            if user_channel_id == user_id =>
        {
            println!("channel ready");
            break;
        }
        Some(ev) => { println!("event: {:?}", ev); node.event_handled(); }
        None => std::thread::sleep(std::time::Duration::from_secs(1)),
    }
}
```

`next_event()` is a blocking poll; `event_handled()` must be called
after each event so LDK can mark it persisted.

## Common pitfalls

- Forgetting `event_handled()` -- events replay on restart, breaking
  idempotency.
- Backing up the storage dir mid-channel-update can corrupt state.
  Take cold backups only when stopped, or use `force_close_all_channels`
  before migrating storage.
- Async signers (used for HW wallets) must complete signing within a
  few seconds or peer disconnect cycles begin.
- ldk-node's Esplora source polls; for low latency use a private
  Esplora or run `lightning-block-sync` against bitcoind directly via
  the lower-level crates.
- Version migrations: keep the previous binary available to drain
  funds before upgrading state on major releases.

## References

- Repo: https://github.com/lightningdevkit/ldk-node
- Sample app: https://github.com/lightningdevkit/ldk-node/tree/main/sample
- LDK book: https://lightningdevkit.org/tutorials
- Companion: [../../lightning/ldk/SKILL.md](../../lightning/ldk/SKILL.md), [bdk/SKILL.md](../bdk/SKILL.md)
