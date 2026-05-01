# proptest Bitcoin Strategies - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/testing/property-based`.
> Canonical source: https://docs.rs/proptest/ and https://altsysrq.github.io/proptest-book/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/testing/property-based/SKILL.md

## Concept

A `proptest` strategy is a deterministic generator parameterised by an
RNG seed plus a shrinkability rule. For Bitcoin code, well-shaped
strategies are the difference between fuzzing-style noise (mostly
rejected by parsers) and useful coverage (valid-but-edgy txs, scripts,
descriptors). Strategies for Bitcoin types belong in a dedicated
module so they can be reused across crates.

The key strategies fall into three layers:

1. Primitives: `OutPoint`, `Amount`, sighash types, network type.
2. Wire types: `TxIn`, `TxOut`, `Transaction`, `Block`.
3. Domain types: descriptors, miniscripts, PSBTs, BIP-32 derivation
   paths, signet challenges.

Each layer should produce valid-by-construction values with controlled
distribution: small inputs frequent, edge cases (empty, max-size,
self-spending) sprinkled in, and invalid inputs only when the property
explicitly tests rejection paths.

## Walkthrough / mechanics

A strategy is anything that implements `proptest::strategy::Strategy`.
Composition happens via `prop_compose!` and combinators
(`.prop_map`, `.prop_filter`, `prop_oneof!`).

Typical pattern for `Transaction`:

```rust
use proptest::prelude::*;
use bitcoin::{Transaction, TxIn, TxOut, OutPoint, Amount, ScriptBuf, Witness};
use bitcoin::consensus::Encodable;

prop_compose! {
    fn arb_outpoint()(
        txid in any::<[u8; 32]>(),
        vout in 0u32..1024
    ) -> OutPoint {
        OutPoint::new(bitcoin::Txid::from_byte_array(txid), vout)
    }
}

prop_compose! {
    fn arb_txin()(
        prev in arb_outpoint(),
        seq  in 0u32..=u32::MAX,
        script in prop::collection::vec(any::<u8>(), 0..520),
        witness in prop::collection::vec(
            prop::collection::vec(any::<u8>(), 0..520),
            0..16,
        ),
    ) -> TxIn {
        TxIn {
            previous_output: prev,
            script_sig: ScriptBuf::from_bytes(script),
            sequence: bitcoin::Sequence(seq),
            witness: Witness::from_slice(&witness),
        }
    }
}

prop_compose! {
    fn arb_txout()(
        sats in 0u64..21_000_000 * 100_000_000,
        spk in prop::collection::vec(any::<u8>(), 0..10_000),
    ) -> TxOut {
        TxOut {
            value: Amount::from_sat(sats),
            script_pubkey: ScriptBuf::from_bytes(spk),
        }
    }
}

prop_compose! {
    pub fn arb_tx()(
        version in -2i32..=3,
        ins  in prop::collection::vec(arb_txin(), 1..16),
        outs in prop::collection::vec(arb_txout(), 1..16),
        locktime in 0u32..=u32::MAX,
    ) -> Transaction {
        Transaction {
            version: bitcoin::transaction::Version(version),
            lock_time: bitcoin::absolute::LockTime::from_consensus(locktime),
            input: ins,
            output: outs,
        }
    }
}
```

Use it:

```rust
proptest! {
    #![proptest_config(ProptestConfig::with_cases(2_000))]
    #[test]
    fn tx_consensus_roundtrip(tx in arb_tx()) {
        let mut bytes = Vec::new();
        tx.consensus_encode(&mut bytes).unwrap();
        let decoded = bitcoin::consensus::deserialize::<Transaction>(&bytes)
            .expect("must decode anything we just encoded");
        prop_assert_eq!(tx, decoded);
    }
}
```

## Worked example

Strategies and properties for descriptors and miniscripts:

```rust
use miniscript::{Descriptor, DescriptorPublicKey};
use std::str::FromStr;
use proptest::prelude::*;

prop_compose! {
    fn arb_xonly_pk()(seed in any::<[u8; 32]>()) -> DescriptorPublicKey {
        // Deterministic: derive a tweaked secp pubkey from seed
        let secp = bitcoin::secp256k1::Secp256k1::new();
        let sk = bitcoin::secp256k1::SecretKey::from_slice(&seed).unwrap();
        let pk = bitcoin::PublicKey::new(sk.public_key(&secp));
        DescriptorPublicKey::Single(miniscript::descriptor::SinglePub {
            origin: None,
            key: miniscript::descriptor::SinglePubKey::FullKey(pk),
        })
    }
}

fn arb_descriptor() -> impl Strategy<Value = Descriptor<DescriptorPublicKey>> {
    prop_oneof![
        arb_xonly_pk().prop_map(|k| Descriptor::new_pkh(k).unwrap()),
        arb_xonly_pk().prop_map(|k| Descriptor::new_wpkh(k).unwrap()),
        prop::collection::vec(arb_xonly_pk(), 1..5)
            .prop_filter("dedup", |v| {
                let mut s: Vec<_> = v.iter().collect();
                s.sort_by_key(|k| k.to_string());
                s.dedup();
                s.len() == v.len()
            })
            .prop_map(|keys| {
                let n = keys.len();
                Descriptor::new_sh_sortedmulti(n.min(15) as usize, keys).unwrap()
            }),
    ]
}

proptest! {
    #[test]
    fn descriptor_roundtrip_with_checksum(d in arb_descriptor()) {
        let s = d.to_string();
        let parsed = Descriptor::<DescriptorPublicKey>::from_str(&s)
            .expect("must reparse");
        prop_assert_eq!(d.to_string(), parsed.to_string());
    }

    #[test]
    fn address_for_descriptor_is_valid(d in arb_descriptor()) {
        if let Ok(addr) = d.address(bitcoin::Network::Regtest) {
            // Round-trip the address
            let s = addr.to_string();
            let parsed = bitcoin::Address::from_str(&s)
                .unwrap()
                .require_network(bitcoin::Network::Regtest)
                .unwrap();
            prop_assert_eq!(addr, parsed);
        }
    }
}
```

PSBT round-trip with shrinking-friendly counterexamples:

```rust
use bitcoin::psbt::Psbt;

proptest! {
    #[test]
    fn psbt_base64_roundtrip(tx in arb_tx().prop_filter(
        "psbt requires non-coinbase",
        |t| !t.input.is_empty() && !t.is_coinbase(),
    )) {
        let psbt = Psbt::from_unsigned_tx(tx).unwrap();
        let s = psbt.to_string();
        let parsed: Psbt = s.parse().unwrap();
        prop_assert_eq!(psbt, parsed);
    }
}
```

Tune the runner for slow strategies:

```rust
proptest! {
    #![proptest_config(ProptestConfig {
        cases: 500,
        max_shrink_iters: 20_000,
        timeout: 30_000,
        .. ProptestConfig::default()
    })]
    #[test]
    fn miniscript_static_analysis_does_not_panic(
        d in arb_descriptor()
    ) {
        if let Descriptor::Wsh(wsh) = &d {
            let ms = wsh.as_inner();
            let _ = ms.script_size();
        }
    }
}
```

Run:

```bash
cargo test --release tx_consensus_roundtrip -- --nocapture
# When a counterexample is found, proptest persists it under
# proptest-regressions/<test_name>.txt
```

Re-run a regression deterministically:

```bash
PROPTEST_CASES=1 cargo test tx_consensus_roundtrip
```

## Common pitfalls

- Filter-heavy strategies (`prop_filter`) starve when the predicate
  rejects most inputs. Prefer constructing valid values rather than
  generating then rejecting.
- Forgetting bounds on collection sizes (`0..usize::MAX`) eats memory
  during shrinking. Always cap.
- Assertions inside the strategy break shrinking. Keep validation
  outside the generator.
- `proptest-regressions/` files must be committed; otherwise CI cannot
  reproduce a previously seen counterexample.
- Using `Hash` derive on types containing `f64` etc. yields
  non-shrinkable equivalence classes - keep generated types pure.

## References

- proptest book: `https://altsysrq.github.io/proptest-book/`
- bitcoin crate: `https://docs.rs/bitcoin/`
- rust-miniscript: `https://docs.rs/miniscript/`
- BDK proptest examples:
  `https://github.com/bitcoindevkit/bdk/tree/master/crates/wallet/tests`
