# rust-bitcoin API Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/rust-bitcoin`.
> Canonical source: https://github.com/rust-bitcoin/rust-bitcoin
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/rust-bitcoin/SKILL.md

## Concept

`rust-bitcoin` is the foundation crate for Bitcoin in Rust. It provides
strongly-typed primitives (`Transaction`, `Script`, `Address`, `Network`),
consensus encoding/decoding, BIP32 derivation, PSBT (BIP174), and Taproot
support. Most higher-level crates (BDK, LDK, miniscript-rs, electrs)
re-export it. The API leans on the type system to prevent class-of-bugs
like passing the wrong network or mixing pubkey parities.

Two key design choices: borrowed `Script` vs owned `ScriptBuf` (most APIs
take `&Script`), and `Amount`/`SignedAmount` newtypes that block raw
satoshi math errors.

## API walkthrough

```rust
use bitcoin::{
    Address, Amount, Network, OutPoint, ScriptBuf, Sequence, Transaction,
    TxIn, TxOut, Witness,
    absolute::LockTime,
    transaction::Version,
};
use bitcoin::secp256k1::{Secp256k1, SecretKey};
use bitcoin::key::{CompressedPublicKey, PrivateKey};

let secp = Secp256k1::new();

// Keys
let sk = SecretKey::from_slice(&[0x42; 32])?;
let priv_key = PrivateKey::new(sk, Network::Bitcoin);
let pub_key = priv_key.public_key(&secp);

// Address
let comp = CompressedPublicKey::try_from(pub_key)?;
let addr = Address::p2wpkh(&comp, Network::Bitcoin);

// Transaction
let tx = Transaction {
    version: Version::TWO,
    lock_time: LockTime::ZERO,
    input: vec![TxIn {
        previous_output: OutPoint::null(),
        script_sig: ScriptBuf::new(),
        sequence: Sequence::MAX,
        witness: Witness::new(),
    }],
    output: vec![TxOut {
        value: Amount::from_sat(50_000),
        script_pubkey: addr.script_pubkey(),
    }],
};
println!("txid: {}", tx.compute_txid());
```

`Address::p2wpkh(pk: &CompressedPublicKey, hrp: impl Into<KnownHrp>)`
returns `Address` directly, so there is no `?` to apply. It takes a
`CompressedPublicKey`, and a `bitcoin::PublicKey` may be uncompressed,
so the narrowing conversion is `TryFrom`, not `From`.

## Worked example: derive an address from xpub at index 5

```rust
use bitcoin::bip32::{DerivationPath, Xpub};
use bitcoin::secp256k1::Secp256k1;
use bitcoin::{Address, CompressedPublicKey, Network};
use std::str::FromStr;

fn address_at(xpub_str: &str, index: u32) -> anyhow::Result<Address> {
    let secp = Secp256k1::verification_only();
    let xpub = Xpub::from_str(xpub_str)?;
    let path = DerivationPath::from_str(&format!("m/0/{index}"))?;
    let child = xpub.derive_pub(&secp, &path)?;
    let comp = CompressedPublicKey(child.public_key);
    Ok(Address::p2wpkh(&comp, Network::Bitcoin))
}
```

`compute_txid()` returns the BIP141 txid (witness-stripped); for the
wtxid (witness-included) call `compute_wtxid()`. Always serialize via
`bitcoin::consensus::encode::serialize` rather than hand-rolling bytes.

## Common pitfalls

- API churn between 0.x minor versions (0.30 -> 0.31 -> 0.32 each
  reshuffled module paths). Pin the patch version and read CHANGELOG
  before bumping. `0.33.0-beta` (2026-02-23) continues the pattern:
  `Amount` now enforces a `MAX_MONEY` invariant, so `Amount::MAX` is
  21 million BTC rather than `u64::MAX` and `Amount::from_sat` returns
  `Result<Amount, OutOfRangeError>`; `Psbt`'s serde representation
  changed; and the MSRV moved 1.63.0 -> 1.74.0. Upstream says it will
  never publish a plain `0.33.0` -- the next tag is `0.34.0-beta`, and
  the suffix drops once `bitcoin-units`, `bitcoin-primitives` and
  `bitcoin-consensus-encoding` reach `1.0.0`. As of September 2026 only
  `bitcoin-consensus-encoding` has (1.2.0, 2026-08-13).
- Mixing `Script` and `ScriptBuf`: pass `&script_buf` (deref to `&Script`)
  rather than cloning. `script_pubkey()` already returns `ScriptBuf`.
- `Amount::from_btc` does not panic -- it returns
  `Result<Amount, ParseAmountError>`, so handle the error rather than
  assuming infallibility. It still takes an `f64`, so prefer it only
  for trusted constants and use `Amount::from_str` for user input.
- Network mismatch: `Address<NetworkUnchecked>` vs `Address<NetworkChecked>`
  type states catch this at compile time -- always call `.require_network`.

## References

- Repo: https://github.com/rust-bitcoin/rust-bitcoin
- Docs: https://docs.rs/bitcoin
- Examples directory: https://github.com/rust-bitcoin/rust-bitcoin/tree/master/bitcoin/examples
- Companion: [bdk/SKILL.md](../bdk/SKILL.md), [secp256k1-rs/SKILL.md](../secp256k1-rs/SKILL.md)
