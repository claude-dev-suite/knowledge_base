# corepc RPC Client and Node Harness - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/corepc`.
> Canonical source: https://github.com/rust-bitcoin/corepc
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/corepc/SKILL.md

## Concept

Bitcoin Core's JSON-RPC responses change shape between majors: fields
appear, disappear, and change type. `corepc` embraces that instead of
papering over it. `corepc-types` carries **one module per Core major**,
`v17` through `v31`, each holding the literal JSON shape that version
returns, plus a `model` module of version-agnostic types built from
`rust-bitcoin` types. Every version type has an `into_model()` that
converts up.

`corepc-client` is a thin blocking (or async) client whose modules are
also per-version, and whose real job upstream is to exercise those
types. `bitcoind` starts and stops real regtest nodes for integration
tests.

## Where the crates live (as of September 2026)

Repo is `github.com/rust-bitcoin/corepc`, but the GitHub README now
carries a "This repository has moved" banner pointing at the org's own
Forgejo instance, `git.rust-bitcoin.org/rust-bitcoin/corepc`, and says
issues and pull requests belong there. Crates still publish to
crates.io.

| Crate | Version | Released |
|-------|---------|----------|
| `corepc-types` | 0.15.0 | 18 June 2026 |
| `corepc-client` | 0.16.0 | 18 June 2026 |
| `bitcoind` | 0.41.0 | 18 June 2026 |
| `jsonrpc` | 0.20.1 | 27 May 2026 |
| `bitreq` | 0.3.7 | 28 May 2026 |
| `electrsd` | 0.41.0 | 18 June 2026 |

Workspace MSRV is Rust 1.75.0 and the crates build against
`bitcoin = "0.32.0"`. The April-to-June 2026 cadence was roughly
fortnightly (`corepc-client` 0.11.0 on 11 April through 0.16.0 on 18
June); no tag has landed since that June wave.

## The production-use caveat

Read this before designing around the client. The repo README states:

> If you require a JSON RPC client in production software it is expected
> you write your own and only use the `corepc-types` crate in your
> dependency graph.
>
> **Please do not use `corepc-client` in production and raise bugs,
> issues, or feature requests.**

Meanwhile `rust-bitcoin/rust-bitcoincore-rpc` was archived on
25 November 2025, and its README tells its users to switch to
`corepc-client`. Both are upstream statements and they pull in different
directions. The workable reading:

- Long-lived service talking to `bitcoind`: depend on `corepc-types`,
  put a small client of your own over `jsonrpc` (or any HTTP stack).
- Tests, tooling, one-off scripts: `corepc-client` is fine and is what
  it is maintained for.

`bitcoincore-rpc` itself is frozen at 0.19.0 (May 2024), so staying put
is not an option either.

## Walkthrough: a typed call

```toml
[dependencies]
corepc-client = { version = "0.16", features = ["client-sync"] }
```

```rust
use corepc_client::client_sync::v31::Client;
use corepc_client::client_sync::Auth;

let client = Client::new_with_auth(
    "http://127.0.0.1:8332",
    Auth::UserPass("user".into(), "pass".into()),
)?;

// corepc_types::v31 shape: `chain: String`, `blocks: i64`, ...
let json = client.get_blockchain_info()?;

// corepc_types::model shape: `chain: Network`, `blocks: u32`, ...
let info = json.into_model()?;
println!("{} at height {}", info.chain, info.blocks);
```

`Auth` is `None | UserPass(String, String) | CookieFile(PathBuf)`;
`Client::new(url)` skips auth entirely. Neither client feature is on by
default - pick `client-sync` or `client-async`.

`into_model()` is fallible on purpose. Core marks fields as "numeric"
and occasionally returns `-1`, so the version types use `i64` and the
conversion to `u32`/`u64` can fail with a `NumericError`; amounts have
to survive the JSON float round trip too. Handle the error rather than
unwrapping it in a service.

## Picking the right version module

Each version client asserts the server it was built for. The v31 module
expands `impl_client_check_expected_server_version!({ [310000] })`, so a
`v31::Client` pointed at a v29 node fails loudly instead of silently
mis-deserialising a changed field.

Nothing in the repo sniffs the peer's version for you - the README is
explicit that this is left to the application. The practical pattern is
to call `getnetworkinfo` once at startup, compare `version`, and refuse
to run on a mismatch.

Method coverage is deliberately complete: every documented Core RPC
since `corepc-types` 0.9.0 (September 2025), plus a batch of hidden ones
added in client 0.11.0 (April 2026).

## Walkthrough: integration test against a real bitcoind

```toml
[dev-dependencies]
bitcoind = { version = "0.41", features = ["latest", "download"] }
```

```rust
use bitcoind::BitcoinD;

#[test]
fn wallet_roundtrip() {
    let node = BitcoinD::from_downloaded().unwrap();
    let info = node.client.get_blockchain_info().unwrap();
    assert_eq!(info.chain, "regtest");
    // datadir is a tempfile::TempDir; the process is killed on drop
}
```

`BitcoinD::new(exe)` and `BitcoinD::with_conf(exe, &conf)` use a binary
already present on the host instead of downloading one. The crate
re-exports the version-specific types as `vtype` and the model types as
`mtype`.

Version features select which Core the harness expects and cascade
downwards:

- `default = ["0_17_2"]` - the default is **not** a modern node.
- `latest = ["31_0"]`.
- One pinned Core binary per supported major: `31_0` → 31.0, `30_2` →
  30.2, `29_0` → 29.0, then `28_2`, `27_2`, `26_2`, ... down to
  `0_17_2` (as of `bitcoind` 0.41.0, June 2026). There is deliberately
  **no `30_0` or `30_1`** feature, excluded upstream because of a
  wallet migration bug.
- No feature exists for Core 29.1-29.4, 30.3 or 31.1, although all of
  those tags shipped (as of September 2026). The `Cargo.toml` comment
  upstream about supporting every minor of the latest three majors is
  stale - `bitcoind/src/versions.rs` maps exactly one binary per
  feature.
- Enabling several version features is not an error; the highest wins.

`--no-default-features` does not build - a version feature is mandatory.

## Migrating from rust-bitcoincore-rpc

The node-harness crate has been renamed twice. It became `corepc-node`,
then was renamed **back** to `bitcoind` in 0.37.0 (16 April 2026, PR
#542), with the version number chosen to continue past the old
standalone `bitcoind` releases. `corepc-node` is frozen at 0.12.0
(April 2026) - move to `bitcoind` 0.37.0 or later.

Porting checklist:

- Replace the single universal response struct with a version module
  plus an `into_model()` hop (or read `v3x` fields directly and skip
  `model`).
- Add error handling around `into_model()`.
- Swap `bitcoincore_rpc::Auth` for `corepc_client::client_sync::Auth` -
  same three variants.
- Drop any hand-rolled `call()` workarounds for missing methods.

## Common pitfalls

- Shipping `corepc-client` in production. Upstream asks you not to and
  will not take bug reports for it.
- Mixing release waves. `corepc-client` 0.16.0 pins `corepc-types`
  0.15.0, and `bitcoind` 0.41.0 pins `corepc-client` 0.16.0. Take one
  wave or the types will not line up.
- Leaving `bitcoind` on its default `0_17_2` feature and wondering why
  modern RPCs are missing.
- The `download` feature fetches a Core binary over the network at build
  time; sealed CI images will block it. Pre-stage the binary and use
  `BitcoinD::new(exe)`.
- Assuming a version module exists for a Core point release. Support is
  per-feature and gapped (no `30_0`/`30_1`), not continuous.

## References

- Repo (old GitHub mirror): https://github.com/rust-bitcoin/corepc
- Repo (active, Forgejo): https://git.rust-bitcoin.org/rust-bitcoin/corepc
- Docs: https://docs.rs/corepc-types, https://docs.rs/corepc-client
- Archived predecessor: https://github.com/rust-bitcoin/rust-bitcoincore-rpc
- Companion: [rust-bitcoin/SKILL.md](../rust-bitcoin/SKILL.md), [bdk/SKILL.md](../bdk/SKILL.md)
