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
the basis of the Swift / Kotlin / Python bindings. Use it for mobile
wallets and apps; drop down to raw `lightning` only when ldk-node's
choices don't fit.

## API walkthrough

Written against ldk-node 0.7.0 (released 2025-12-03, still the latest
release as of September 2026), which depends on the `lightning` 0.2
crate line.

```rust
use ldk_node::Builder;
use ldk_node::bitcoin::Network;
use ldk_node::bitcoin::secp256k1::PublicKey;
use ldk_node::lightning::ln::msgs::SocketAddress;
use ldk_node::lightning_invoice::{
    Bolt11Invoice, Bolt11InvoiceDescription, Description};
use std::str::FromStr;

fn main() -> anyhow::Result<()> {
    let mut builder = Builder::new();
    builder.set_network(Network::Testnet);
    builder.set_chain_source_esplora(
        "https://mempool.space/testnet/api".to_string(), None);
    builder.set_storage_dir_path("./ldk-data".to_string());
    let node = builder.build()?;

    node.start()?;
    println!("node id: {}", node.node_id());

    // Connect to a peer
    let peer_pubkey = PublicKey::from_str("02abc...")?;
    let peer_addr = SocketAddress::from_str("1.2.3.4:9735")?;
    node.connect(peer_pubkey, peer_addr, true)?;

    // Receive
    let description = Bolt11InvoiceDescription::Direct(
        Description::new("coffee".to_string())?);
    let invoice = node.bolt11_payment().receive(
        100_000,                     // amount msat
        &description, 3600)?;
    println!("invoice: {}", invoice);

    // Pay
    let to_pay = Bolt11Invoice::from_str("lnbc...")?;
    let payment_id = node.bolt11_payment().send(&to_pay, None)?;
    println!("payment id: {:?}", payment_id);

    node.stop()?;
    Ok(())
}
```

The `Builder` setters take `&mut self` and return `&mut Self`, so they
cannot be chained off a `Builder::new()` temporary in a `let` binding;
use a `let mut` binding. `Network` is re-exported at
`ldk_node::bitcoin::Network`, not `ldk_node::Network`. `receive` takes
a `&Bolt11InvoiceDescription`, not a string.

## Worked example: open a channel and wait for funding

```rust
use ldk_node::UserChannelId;

let user_id: UserChannelId = node.open_channel(
    peer_pubkey,
    peer_addr,
    1_000_000,            // channel value sat
    Some(0),              // push msat
    None,                 // channel config: defaults
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

`next_event()` is a non-blocking peek that returns `Option<Event>`;
`wait_next_event()` blocks. `event_handled()` must be called after
each event so LDK can mark it persisted.

`open_channel` opens an unannounced (private) channel and takes no
announce flag in 0.7 — announced channels use the separate
`open_announced_channel`, which errors unless `listening_addresses`
and `node_alias` are configured.

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

## Version floor and security history (as of September 2026)

LDK backports security fixes as patch releases, so pinning a minor
line alone is not a floor. Check `Cargo.lock`, not the manifest.

- **`lightning` 0.2.6 (2026-09-09)** -- current floor on the 0.2 line.
  Fixes a DoS that could leave a `ChannelManager` unreadable on
  deserialization after a bogus duplicate-`payment_hash` HTLC, and a
  splice fee-inflation bug letting a malicious counterparty make us
  over-allocate fee into their output.
- **`lightning` 0.2.5 / 0.1.12 (both 2026-08-05)** -- on-chain
  funds-theft fix for forwarding nodes accepting channels from
  untrusted peers (`ChannelMonitor` mis-resolved two HTLCs with equal
  `payment_hash` and amount), plus several remote-panic DoS vectors.
- **`lightning` 0.2.3 (2026-06-18)** -- superseded by the 0.2.6 floor.
  Fixed anchor-reserve underestimates that could leave a node unable
  to properly force-close, `possiblyrandom` not generating random data
  unless explicitly configured (HashDoS), and several
  remote-triggerable panics.
- **`lightning` 0.3.0-rc1 (2026-08-31)** -- next-major release
  candidate. No stable 0.3 yet.

The 0.1 line got no backport of the 0.2.6 fixes, so 0.1.12 is not a
current floor.

ldk-node 0.7.0 declares `lightning = "0.2.0"`, a caret requirement: a
fresh resolve lands on 0.2.6, but an existing lockfile needs
`cargo update -p lightning`.

## Migrating to `lightning` 0.3

0.3.0-rc1 changes defaults and invalidates some persisted state:

- `ChannelHandshakeConfig::negotiate_anchors_zero_fee_htlc_tx`
  defaults to `true`.
- `UserConfig::manually_accept_inbound_channels` is removed and is
  now always on: inbound channels must be accepted explicitly from
  event handling. `ChannelHandshakeLimits::max_funding_satoshis` is
  gone with it.
- Payment metadata is committed to in the payment secret, so BOLT11
  invoices already issued with payment metadata are invalidated on
  upgrade (and newly issued ones on downgrade). Long-lived invoices
  are the hazard here.
- Blinded-path receives are authenticated with a `ReceiveAuthKey`;
  0.3+ rejects payments over blinded paths built by earlier versions.
  Existing BOLT12 offers stay valid; issued-but-unclaimed BOLT12
  refunds do not.
- Splices gain RBF and can add and remove funds in the same splice. A
  splice negotiated pre-0.3 cannot be RBF'd, and downgrade after an
  RBF is unsupported.
- MSRV rises to rustc 1.75.

Primary `rust-lightning` development moved to git.rust-bitcoin.org
during the 0.3 cycle; GitHub is kept as a synchronised mirror.

## References

- Repo: https://github.com/lightningdevkit/ldk-node
- Sample app: https://github.com/lightningdevkit/ldk-node/tree/main/sample
- LDK book: https://lightningdevkit.org/tutorials
- Companion: [../../lightning/ldk/SKILL.md](../../lightning/ldk/SKILL.md), [bdk/SKILL.md](../bdk/SKILL.md)
