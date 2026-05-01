# cargo-fuzz Walkthrough (rust-bitcoin / BDK) - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/testing/fuzz`.
> Canonical source: https://rust-fuzz.github.io/book/cargo-fuzz.html and https://github.com/rust-bitcoin/rust-bitcoin/tree/master/fuzz
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/testing/fuzz/SKILL.md

## Concept

`cargo-fuzz` wraps libFuzzer for Rust crates. Every fuzz target is a
single `.rs` file in a sibling `fuzz/` cargo project, with its own
`Cargo.toml` declaring the parent crate as a dependency. Targets use
`fuzz_target!(|data: &[u8]| { ... })` and run under nightly Rust with
`-Zsanitizer=address,undefined` plus the libFuzzer driver.

For Bitcoin Rust libraries (`rust-bitcoin`, `rust-miniscript`,
`rust-secp256k1`, `bdk`), cargo-fuzz is the standard hardening tool
for parser code: consensus deserialisation, miniscript parsing,
descriptor parsing, PSBT round-trips, BIP-32 derivation. Unlike
property-based tests, fuzzing finds crashes inside `unsafe` paths,
panics, OOM patterns, and infinite loops.

## Walkthrough / mechanics

Set up a fuzz workspace next to the crate under test:

```
my-crate/
├── Cargo.toml
├── src/
└── fuzz/
    ├── Cargo.toml
    ├── fuzz_targets/
    │   └── tx_decode.rs
    └── corpus/
        └── tx_decode/
```

`fuzz/Cargo.toml`:

```toml
[package]
name = "my-crate-fuzz"
version = "0.0.0"
publish = false
edition = "2021"

[package.metadata]
cargo-fuzz = true

[dependencies]
libfuzzer-sys = "0.4"
my-crate      = { path = ".." }

[[bin]]
name = "tx_decode"
path = "fuzz_targets/tx_decode.rs"
test = false
doc  = false
```

A target:

```rust
#![no_main]
use libfuzzer_sys::fuzz_target;
use bitcoin::{consensus, Transaction};

fuzz_target!(|data: &[u8]| {
    if let Ok(tx) = consensus::deserialize::<Transaction>(data) {
        // Round-trip property
        let bytes = consensus::serialize(&tx);
        let tx2: Transaction = consensus::deserialize(&bytes).unwrap();
        assert_eq!(tx, tx2);
    }
});
```

Run:

```bash
cargo install cargo-fuzz
cd my-crate
cargo +nightly fuzz run tx_decode \
    -- -max_total_time=600 -jobs=4
```

`cargo-fuzz` auto-creates `corpus/tx_decode/` and persists interesting
inputs there. Crashes drop to `fuzz/artifacts/tx_decode/`.

Reproduce a crash deterministically:

```bash
cargo +nightly fuzz run tx_decode fuzz/artifacts/tx_decode/crash-1234
```

Coverage report:

```bash
cargo +nightly fuzz coverage tx_decode
cargo +nightly cov -- show \
    target/x86_64-unknown-linux-gnu/coverage/.../tx_decode \
    --instr-profile=fuzz/coverage/tx_decode/coverage.profdata \
    --format=html > coverage.html
```

## Worked example

Fuzz miniscript parsing in `rust-miniscript`:

```rust
// fuzz/fuzz_targets/miniscript_parse.rs
#![no_main]
use libfuzzer_sys::fuzz_target;
use miniscript::{Miniscript, Segwitv0};
use std::str::FromStr;

fuzz_target!(|data: &[u8]| {
    if let Ok(s) = std::str::from_utf8(data) {
        if let Ok(ms) = Miniscript::<bitcoin::PublicKey, Segwitv0>::from_str(s) {
            // Round-trip
            let s2 = ms.to_string();
            let ms2 = Miniscript::<bitcoin::PublicKey, Segwitv0>::from_str(&s2)
                .expect("printed form must reparse");
            assert_eq!(ms, ms2);

            // Static analysis must not panic
            let _ = ms.script_size();
            let _ = ms.max_satisfaction_size();
        }
    }
});
```

Bootstrap the corpus with valid miniscripts:

```bash
mkdir -p fuzz/corpus/miniscript_parse
echo -n 'pk(02...)' > fuzz/corpus/miniscript_parse/pk
echo -n 'and_v(v:pk(02...),pk(03...))' > fuzz/corpus/miniscript_parse/and
echo -n 'thresh(2,pk(02...),pk(03...),pk(04...))' \
    > fuzz/corpus/miniscript_parse/thresh

cargo +nightly fuzz run miniscript_parse \
    -- -max_total_time=900 \
       -dict=fuzz/dicts/miniscript.txt
```

Dictionary file `fuzz/dicts/miniscript.txt` (a few entries):

```
"pk("
"and_v("
"or_d("
"thresh("
"hash160("
"older("
"sha256("
")"
","
```

CI integration (GitHub Actions, run on PRs touching the parser):

```yaml
- uses: actions-rs/toolchain@v1
  with: { toolchain: nightly, override: true }
- run: cargo install cargo-fuzz
- run: |
    cd fuzz
    cargo +nightly fuzz run miniscript_parse \
      -- -max_total_time=300 -runs=200000
```

When a crash is found, minimise:

```bash
cargo +nightly fuzz tmin miniscript_parse \
    fuzz/artifacts/miniscript_parse/crash-deadbeef
# Outputs a smaller crash-XXXX file
```

Convert the minimal repro into a regression unit test:

```rust
#[test]
fn regression_miniscript_panic_42() {
    let bytes = include_bytes!("crashes/miniscript_panic_42.bin");
    if let Ok(s) = std::str::from_utf8(bytes) {
        let _ = Miniscript::<bitcoin::PublicKey, Segwitv0>::from_str(s);
    }
}
```

## Common pitfalls

- Forgetting `#![no_main]` and `fuzz_target!`: the binary builds but
  libFuzzer cannot find the entry point.
- Running on stable Rust: cargo-fuzz requires nightly for sanitizer
  flags. `cargo +nightly` everywhere.
- Asserting properties (e.g. round-trip) inside the target: if the
  property is wrong, every input is a "crash" and you drown in
  artifacts. First validate properties only when the parse succeeds.
- Forgetting to commit the corpus: a shared corpus is the fuzzer's
  memory. Lose it and you re-find the same shallow bugs every time.
- ASLR + sanitizers can spike memory. `-rss_limit_mb=2048` keeps
  individual workers bounded.

## References

- cargo-fuzz book: `https://rust-fuzz.github.io/book/`
- rust-bitcoin fuzz dir:
  `https://github.com/rust-bitcoin/rust-bitcoin/tree/master/fuzz`
- rust-miniscript fuzz dir:
  `https://github.com/rust-bitcoin/rust-miniscript/tree/master/fuzz`
- libFuzzer flag reference: `https://llvm.org/docs/LibFuzzer.html`
