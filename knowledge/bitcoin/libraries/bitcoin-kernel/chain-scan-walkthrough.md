# libbitcoinkernel Chain Scan - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/bitcoin-kernel`.
> Canonical source: https://github.com/sedited/rust-bitcoinkernel
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/bitcoin-kernel/SKILL.md

## Concept

`libbitcoinkernel` is Bitcoin Core's validation engine behind a C ABI.
The header is `src/kernel/bitcoinkernel.h` in the Core tree; the
`bitcoinkernel` CMake target builds it. `rust-bitcoinkernel` wraps that
ABI: `libbitcoinkernel-sys` holds the raw FFI plus the build script, and
`bitcoinkernel` holds the safe API.

Unlike the removed `libbitcoinconsensus` (deprecated in Core 27.0, gone
in 28.0), the kernel library is **stateful**. You hand it a data
directory and a blocks directory, it owns a `ChainstateManager`, and
from there it can validate a block in full chain context, walk the block
index, and read block and undo data back off disk.

## Status (as of September 2026)

Experimental at both layers.

- Core v31.1 (July 2026) ships the header but the Remarks section says
  it "is unversioned and not stable yet. Users should expect breaking
  changes. It is also not yet included in releases of Bitcoin Core."
  The CMake option `BUILD_KERNEL_LIB` ("Build experimental bitcoinkernel
  library") defaults to `BUILD_UTIL_CHAINSTATE`, which is `OFF`, so
  release builds do not produce it. Upstream tracking is
  bitcoin/bitcoin#27587 (open, last updated 31 July 2026).
- Rust side: `bitcoinkernel` 0.3.0 and `libbitcoinkernel-sys` 0.4.0,
  both published 26 August 2026, MSRV 1.71.0. The repo moved with its
  author's GitHub rename - `TheCharlatan/rust-bitcoinkernel` now
  redirects to `sedited/rust-bitcoinkernel`.

Every minor so far has broken source compatibility. 0.3.0 alone renamed
`ProcessBlockHeaderResult::Success`/`Failed` to `Valid`/`Invalid`, made
`ChainstateManager::process_block_header` return a `Result`, and
retyped `verify`'s `flags` parameter from `u32` to
`ScriptVerificationFlags`. Pin exact versions.

## Build model

The sys crate vendors Bitcoin Core as a `git subtree` under
`libbitcoinkernel-sys/bitcoin` and statically compiles the kernel
library during `cargo build`. Consequences:

- The host needs Core's build dependencies: `cmake`, a C and C++
  toolchain, and Boost.
- Build times are Core build times, not crate build times.
- **The subtree commit is part of your consensus surface.** Updating it
  changes what "valid" means:

```bash
git subtree pull --prefix libbitcoinkernel-sys/bitcoin \
    https://github.com/bitcoin/bitcoin master --squash
./contrib/check_subtree_kernel_commits.sh
```

Since `bitcoinkernel` 0.2.1 / `libbitcoinkernel-sys` 0.3.0 (both 20 May
2026) the sys crate ships checked-in bindings rather than running
`bindgen`; the changelog records the effect only as removing "some
build-time dependencies for this crate". Android cross-compilation
goes through the repo's Nix flake
(`nix build .#libbitcoinkernel-android-aarch64`), output targeting API
24+.

## Walkthrough: open a chainstate and iterate the chain

```rust
use bitcoinkernel::{ChainType, ChainstateManagerBuilder, ContextBuilder};

let context = ContextBuilder::new()
    .chain_type(ChainType::Regtest)
    .build()?;

let chainman = ChainstateManagerBuilder::new(&context, "./data", "./blocks")?
    .worker_threads(4)              // 0-15, clamped; 0 = no parallel verification
    .build()?;

chainman.import_blocks()?;          // load what is already on disk

let chain = chainman.active_chain();
println!("tip height: {}", chain.height());

for entry in chain.iter() {
    let block = chainman.read_block_data(&entry)?;
    let spent = chainman.read_spent_outputs(&entry)?;   // undo data
    // entry.height(), block.transaction(i), spent.iter() ...
}
```

`ChainstateManager::new(&context, data_dir, blocks_dir)` is the
shorthand - it is literally `ChainstateManagerBuilder::new(...)?.build()`.
Reach for the builder when you want `worker_threads`, `wipe_db`,
`block_tree_db_in_memory` or `chainstate_db_in_memory`.

The repo's `examples/silentpaymentscanner.rs` is this loop in anger: it
pairs `read_spent_outputs` with `read_block_data` so each input can be
matched against the coin it spends, which is exactly the shape an
indexer or a BIP352 scanner needs.

## Walkthrough: submit a block

```rust
use bitcoinkernel::{Block, ProcessBlockResult};

let block = Block::new(&block_bytes)?;
match chainman.process_block(&block) {
    ProcessBlockResult::NewBlock => {}      // validated, written to disk
    ProcessBlockResult::Duplicate => {}     // already known, still valid
    ProcessBlockResult::Rejected => {}      // consensus-invalid
}
```

There is no P2P layer, so nothing fetches blocks for you. Headers-first
flows use `process_block_header`, which since 0.3.0 returns
`Result<ProcessBlockHeaderResult, KernelError>` - `Err` is an internal
failure, `Ok(ProcessBlockHeaderResult::Invalid(state))` is a header that
failed validation. 0.3.0 also added context-free `Block::check` and
`Transaction::check` for cheap pre-screening before a full
`process_block`.

## Script verification

```rust
use bitcoinkernel::{prelude::*, verify, PrecomputedTransactionData, Transaction, VERIFY_ALL};

let spending_tx = Transaction::new(&spending_tx_bytes)?;
let prev_tx = Transaction::new(&prev_tx_bytes)?;
let prev_output = prev_tx.output(0)?;
let tx_data = PrecomputedTransactionData::new(&spending_tx, &[prev_output])?;

verify(
    &prev_output.script_pubkey(),
    Some(prev_output.value()),
    &spending_tx,
    0,                          // input index
    Some(VERIFY_ALL),           // or VERIFY_ALL_PRE_TAPROOT
    &tx_data,
)?;                             // Err(KernelError::ScriptVerify(..)) on failure
```

`PrecomputedTransactionData` can also be built without the spent
outputs, but then there is no taproot verification - BIP341 sighashes
need every spent output. This path is the direct replacement for what
`libbitcoinconsensus` used to do.

## Common pitfalls

- **Treating the crate version as the whole dependency.** Two builds of
  `bitcoinkernel` 0.3.0 against different vendored Core commits are
  different validators. Record the subtree commit alongside the crate
  version.
- **Expecting a node.** No P2P, no mempool acceptance, no wallet, no RPC
  server. The v31.1 header's stated scope is block and header
  validation, block-index iteration, block and undo reads, and script
  verification.
- **`get_block_tree_entry` before 0.3.0.** It passed the address of the
  `BlockHash` wrapper instead of the owned hash handle, so every lookup
  missed and it returned `None` for all inputs. Fixed in 0.3.0.
- **Assuming a system package exists.** Core release binaries do not
  ship the library; there is nothing to `apt install`.
- **Old rustc.** MSRV is 1.71.0, but anything below 1.77 needs
  `cargo build --locked` so dependency resolution stays on the pinned
  `Cargo-minimal.lock` / `Cargo-recent.lock` set.
- Context lifetime: keep the `Context` alive for as long as the
  `ChainstateManager` that was built from it.

## References

- Rust bindings: https://github.com/sedited/rust-bitcoinkernel
- Crate docs: https://docs.rs/bitcoinkernel
- Core tracking issue: https://github.com/bitcoin/bitcoin/issues/27587
- Core header: https://github.com/bitcoin/bitcoin/blob/v31.1/src/kernel/bitcoinkernel.h
- Companion: [corepc/SKILL.md](../corepc/SKILL.md), [rust-bitcoin/SKILL.md](../rust-bitcoin/SKILL.md)
