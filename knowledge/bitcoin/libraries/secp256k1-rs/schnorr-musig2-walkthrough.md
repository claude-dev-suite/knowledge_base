# rust-secp256k1 Schnorr and MuSig2 - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/secp256k1-rs`.
> Canonical source: https://github.com/rust-bitcoin/rust-secp256k1
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/secp256k1-rs/SKILL.md

## Concept

`secp256k1` (the Rust crate) wraps libsecp256k1 with a safe API: contexts
are zero-cost type-state markers (`SignOnly`, `VerifyOnly`, `All`), keys
are validated at construction, and the destructor handles cleanup.
Compared to the bare C library, you get `Result`-based error returns and
borrow-checker-enforced safety.

For modern Bitcoin signing you'll touch three groups: **ECDSA** (legacy
+ segwit-v0 inputs), **Schnorr / BIP340** (Taproot key-path and Tapscript
leaves), and **MuSig2 / BIP327** (n-of-n aggregated Schnorr). The
`schnorr` and `musig` features must be enabled in `Cargo.toml`.

## API walkthrough

```rust
use secp256k1::{
    rand, Keypair, Message, Secp256k1, SecretKey, XOnlyPublicKey,
};

let secp = Secp256k1::new();

// BIP340 sign / verify
let sk = SecretKey::new(&mut rand::thread_rng());
let kp = Keypair::from_secret_key(&secp, &sk);
let (xonly, _parity) = kp.x_only_public_key();

let msg = Message::from_digest_slice(&[0x11; 32])?;
let sig = secp.sign_schnorr(&msg, &kp);
secp.verify_schnorr(&sig, msg.as_ref(), &xonly)?;

// ECDH (Taproot tweak inputs etc)
let other_pk = secp256k1::PublicKey::from_secret_key(&secp,
    &SecretKey::new(&mut rand::thread_rng()));
let shared = secp256k1::ecdh::SharedSecret::new(&other_pk, &sk);
```

## Worked example: 2-of-2 MuSig2 round trip

```rust
use secp256k1::musig::{
    new_nonce_pair, AggregatedNonce, KeyAggCache, Session, SessionSecretRand,
};
use secp256k1::{Keypair, Message, Secp256k1, SecretKey};

fn musig2_demo() -> anyhow::Result<()> {
    let secp = Secp256k1::new();
    let kp_a = Keypair::new(&secp, &mut secp256k1::rand::thread_rng());
    let kp_b = Keypair::new(&secp, &mut secp256k1::rand::thread_rng());

    // 1. Aggregate pubkeys
    let mut cache = KeyAggCache::new(&secp,
        &[&kp_a.public_key(), &kp_b.public_key()]);
    let agg_xonly = cache.agg_pk();

    // 2. Each signer publishes a public nonce
    let msg = Message::from_digest_slice(&[0x42; 32])?;
    let sec_a = SessionSecretRand::from_rng(&mut secp256k1::rand::thread_rng());
    let sec_b = SessionSecretRand::from_rng(&mut secp256k1::rand::thread_rng());
    let (sec_nonce_a, pub_nonce_a) =
        new_nonce_pair(&secp, sec_a, Some(&cache), Some(kp_a.secret_key()),
            kp_a.public_key(), Some(msg), None);
    let (sec_nonce_b, pub_nonce_b) =
        new_nonce_pair(&secp, sec_b, Some(&cache), Some(kp_b.secret_key()),
            kp_b.public_key(), Some(msg), None);

    // 3. Aggregate nonces, build session
    let agg_nonce = AggregatedNonce::new(&secp, &[&pub_nonce_a, &pub_nonce_b]);
    let session = Session::new(&secp, &cache, agg_nonce, msg);

    // 4. Each signer produces a partial sig
    let part_a = session.partial_sign(&secp, sec_nonce_a, &kp_a, &cache)?;
    let part_b = session.partial_sign(&secp, sec_nonce_b, &kp_b, &cache)?;

    // 5. Aggregate to final BIP340 sig
    let final_sig = session.partial_sig_agg(&[&part_a, &part_b]);
    secp.verify_schnorr(&final_sig, msg.as_ref(), &agg_xonly)?;
    Ok(())
}
```

The `MuSig2` flow above matches BIP327: round 1 publishes nonces, round 2
publishes partial sigs, an aggregator produces the BIP340-valid sig.
**Never reuse a `SecNonce`** -- the API consumes it by value to enforce
single-use.

## Common pitfalls

- `Message::from_digest_slice` requires exactly 32 bytes; pre-hash with
  the appropriate sighash flavour (`TapSighash` / `SighashCache::taproot_*`).
- The `musig` Cargo feature is not on by default; add 
  `features = ["rand", "schnorr", "musig"]`.
- Reusing nonces across two signing sessions leaks the secret key
  (BIP340 + MuSig2 are vulnerable to nonce reuse). The crate enforces
  single-use via type system; don't `clone` `SecNonce`.
- `XOnlyPublicKey` lacks parity info; remember that Taproot output keys
  use only the x-coordinate, but signing requires tracking parity for
  tweaks (`Keypair::tap_tweak`).
- libsecp256k1 context creation is expensive; create one `Secp256k1` and
  pass references rather than instantiating per call.

## References

- Repo: https://github.com/rust-bitcoin/rust-secp256k1
- Docs: https://docs.rs/secp256k1
- BIP340: https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki
- BIP327 (MuSig2): https://github.com/bitcoin/bips/blob/master/bip-0327.mediawiki
- Companion: [secp256k1-c/SKILL.md](../secp256k1-c/SKILL.md), [../../cryptography/schnorr/SKILL.md](../../cryptography/schnorr/SKILL.md)
