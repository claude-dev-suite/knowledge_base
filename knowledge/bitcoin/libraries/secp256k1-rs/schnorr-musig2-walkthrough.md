# rust-secp256k1 Schnorr and MuSig2 - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/secp256k1-rs`.
> Canonical source: https://github.com/rust-bitcoin/rust-secp256k1
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/secp256k1-rs/SKILL.md

## Concept

`secp256k1` (the Rust crate) wraps libsecp256k1 with a safe API: keys are
validated at construction, the destructor handles cleanup, and you get
`Result`-based error returns plus borrow-checker-enforced safety instead
of the bare C library's raw pointers. Up to 0.31.x the context was a
zero-cost type-state marker (`SignOnly`, `VerifyOnly`, `All`) that every
call took as an argument; 0.33.0 (August 2026) removed the context from
the public API and routes the free functions through an internal global
context instead. The context types and methods still exist but are
deprecated.

For modern Bitcoin signing you'll touch three groups: **ECDSA** (legacy
+ segwit-v0 inputs), **Schnorr / BIP340** (Taproot key-path and Tapscript
leaves), and **MuSig2 / BIP327** (n-of-n aggregated Schnorr). The
`schnorr` and `musig` modules are compiled unconditionally - there are no
features by those names - but key generation needs the `rand` feature.

This article targets 0.33.x; 0.33.1, published to crates.io on
2026-08-29, is the current release.

## API walkthrough

```rust
use secp256k1::{rand, schnorr, Keypair, PublicKey, SecretKey, XOnlyPublicKey};

let mut rng = rand::rng();          // rand 0.9 renamed `thread_rng()` to `rng()`

// BIP340 sign / verify
let sk = SecretKey::new(&mut rng);
let kp = Keypair::from_secret_key(&sk);
let (xonly, _parity) = kp.x_only_public_key();

let sighash = [0x11u8; 32];
let sig = schnorr::sign(&sighash, &kp);            // aux rand from the thread rng
schnorr::verify(&sig, &sighash, &xonly)?;

// ECDH (Taproot tweak inputs etc)
let other_pk = PublicKey::from_secret_key(&SecretKey::new(&mut rng));
let shared = secp256k1::ecdh::SharedSecret::new(&other_pk, &sk);
let bytes = shared.to_secret_bytes();              // `secret_bytes` deprecated in 0.33.0
```

## Worked example: 2-of-2 MuSig2 round trip

```rust
use secp256k1::musig::{
    new_nonce_pair, AggregatedNonce, KeyAggCache, Session, SessionSecretRand,
};
use secp256k1::{rand, Keypair, PublicKey, SecretKey};

fn musig2_demo() -> anyhow::Result<()> {
    let mut rng = rand::rng();
    let (sk_a, pk_a) = secp256k1::generate_keypair(&mut rng);
    let (sk_b, pk_b) = secp256k1::generate_keypair(&mut rng);

    // 1. Sort and aggregate pubkeys
    let mut pubkeys: Vec<&PublicKey> = vec![&pk_a, &pk_b];
    secp256k1::sort_pubkeys(&mut pubkeys);
    let cache = KeyAggCache::new(&pubkeys);
    let agg_xonly = cache.agg_pk();

    // 2. Each signer publishes a public nonce
    let msg: &[u8; 32] = b"This message is exactly 32 bytes";
    let sec_a = SessionSecretRand::from_rng(&mut rng);
    let sec_b = SessionSecretRand::from_rng(&mut rng);
    let (sec_nonce_a, pub_nonce_a) =
        new_nonce_pair(sec_a, Some(&cache), Some(sk_a), pk_a, Some(msg), None);
    let (sec_nonce_b, pub_nonce_b) =
        new_nonce_pair(sec_b, Some(&cache), Some(sk_b), pk_b, Some(msg), None);

    // 3. Aggregate nonces, build session
    let agg_nonce = AggregatedNonce::new(&[&pub_nonce_a, &pub_nonce_b]);
    let session = Session::new(&cache, agg_nonce, msg);

    // 4. Each signer produces a partial sig
    let kp_a = Keypair::from_secret_key(&sk_a);
    let kp_b = Keypair::from_secret_key(&sk_b);
    let part_a = session.partial_sign(sec_nonce_a, &kp_a, &cache);
    let part_b = session.partial_sign(sec_nonce_b, &kp_b, &cache);

    // optional: the aggregator can attribute a bad partial sig to a signer
    assert!(session.partial_verify(&cache, &part_a, &pub_nonce_a, pk_a));

    // 5. Aggregate to final BIP340 sig
    let final_sig = session.partial_sig_agg(&[&part_a, &part_b]);
    final_sig.verify(&agg_xonly, msg)?;
    Ok(())
}
```

The `MuSig2` flow above matches BIP327: round 1 publishes nonces, round 2
publishes partial sigs, an aggregator produces the BIP340-valid sig.
**Never reuse a `SecretNonce`** -- `partial_sign` takes it by value and
the type is neither `Copy` nor `Clone`, so single use is structural;
signing twice with the same nonce panics.

## Common pitfalls

- For ECDSA build the message with `Message::from_digest([u8; 32])`;
  `Message::from_digest_slice` has been deprecated since 0.31.0. The
  BIP340 and MuSig2 entry points skip `Message` entirely - `schnorr::sign`
  takes any `&[u8]` and `Session::new` takes a `&[u8; 32]`. Either way,
  pre-hash with the appropriate sighash flavour (`TapSighash` /
  `SighashCache::taproot_*`).
- There is no `schnorr` or `musig` Cargo feature (and, since 0.33.0, no
  `bitcoin-hashes` one either); `features = ["rand"]` is what key
  generation and `SessionSecretRand::from_rng` need.
- Reusing nonces across two signing sessions leaks the secret key
  (BIP340 + MuSig2 are vulnerable to nonce reuse). The crate enforces
  single-use structurally: `SecretNonce` is not `Copy`/`Clone` and
  `partial_sign` consumes it.
- `XOnlyPublicKey` lacks parity info; remember that Taproot output keys
  use only the x-coordinate, but signing requires tracking parity for
  tweaks (`Keypair::tap_tweak`).
- Context management is no longer your problem as of 0.33.0: the free
  functions share an internal, self-rerandomizing global context. On
  0.31.x and earlier, libsecp256k1 context creation was expensive enough
  that you created one `Secp256k1` and passed references around; that
  code still compiles but every context signing/verification method now
  carries a deprecation warning.

## References

- Repo: https://github.com/rust-bitcoin/rust-secp256k1
- Upstream forge (where development moved as of 0.33.1, August 2026):
  https://git.rust-bitcoin.org/rust-bitcoin/rust-secp256k1
- CHANGELOG: https://git.rust-bitcoin.org/rust-bitcoin/rust-secp256k1/raw/branch/master/CHANGELOG.md
- Reference MuSig2 example (0.33.1): https://docs.rs/crate/secp256k1/0.33.1/source/examples/musig.rs
- Docs: https://docs.rs/secp256k1
- BIP340: https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki
- BIP327 (MuSig2): https://github.com/bitcoin/bips/blob/master/bip-0327.mediawiki
- Companion: [secp256k1-c/SKILL.md](../secp256k1-c/SKILL.md), [../../cryptography/schnorr/SKILL.md](../../cryptography/schnorr/SKILL.md)
