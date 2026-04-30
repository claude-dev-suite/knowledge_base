# BIP340 Signing - Implementation Walkthrough

> Phase B article. Companion to dev-suite skill `bitcoin/cryptography/schnorr`.
> Canonical source: BIP340 reference.py + libsecp256k1 schnorrsig module.
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/cryptography/schnorr/SKILL.md

## Concept

This article documents the moving parts of a production-quality BIP340
signer with emphasis on (a) constant-time discipline, (b) memory hygiene
for secret-bearing buffers, and (c) integration points where signing
deviates between a stand-alone Schnorr signature and a Taproot key-path
spend (which adds a tweak). The skill quick-ref shows the math; this
article covers the *implementation* concerns the math does not.

## Walkthrough / mechanics

### State variables

A signer at any moment in time owns:

- `d0`: original 32-byte secret. Loaded from KDF/HD path.
- `d`:  parity-adjusted secret (32 bytes). `d = n - d0` if `P.y` is odd.
- `P`:  affine pubkey `(P.x, P.y)`, even-y by construction.
- `aux`: 32 bytes of optional randomness; pad with zero if absent.
- `m`: 32-byte message hash.
- `t`: derived nonce mask (transient).
- `k0, k`: nonce scalar (transient).
- `R`: nonce point.
- `e`: challenge scalar.
- `s`: final scalar.

Of these, `d0`, `d`, `t`, `k0`, `k` MUST be wiped on exit. `aux` is not
secret. Use `secp256k1_memczero` / `explicit_bzero` / `OPENSSL_cleanse`.

### Step-by-step with constant-time annotations

```c
int schnorr_sign(const u8 sk[32], const u8 msg32[32],
                 const u8 aux32[32], u8 sig64[restrict 32+32])
{
    secp256k1_scalar d0, d, k0, k, e, s;
    secp256k1_ge   P_ge, R_ge;
    secp256k1_gej  P_gej, R_gej;
    u8 t[32], rand[32];

    /* (1) Parse and validate d0. CT compare: 1 <= d0 < n. */
    int overflow;
    secp256k1_scalar_set_b32(&d0, sk, &overflow);
    if (overflow || secp256k1_scalar_is_zero(&d0)) goto cleanup_fail;

    /* (2) Compute P = d0 * G (CT). */
    secp256k1_ecmult_gen(&ctx->ecmult_gen_ctx, &P_gej, &d0);
    secp256k1_ge_set_gej(&P_ge, &P_gej);
    secp256k1_fe_normalize_var(&P_ge.x);
    secp256k1_fe_normalize_var(&P_ge.y);

    /* (3) Force even-y on P; conditionally negate d0. */
    int p_y_odd = secp256k1_fe_is_odd(&P_ge.y);
    secp256k1_scalar_cond_negate(&d0, p_y_odd);  /* d <- d if even, n-d if odd */
    d = d0;                                       /* keep readable name */

    /* (4) t = d XOR tagged_hash("BIP0340/aux", aux). */
    tagged_hash("BIP0340/aux", aux32, 32, /*out*/ rand);
    for (int i = 0; i < 32; i++) t[i] = ((u8*)&d)[i] ^ rand[i];

    /* (5) rand = tagged_hash("BIP0340/nonce", t || P.x || msg). */
    tagged_hash3("BIP0340/nonce", t, P.x_ser32, msg32, /*out*/ rand);
    secp256k1_scalar_set_b32(&k0, rand, &overflow);
    if (overflow || secp256k1_scalar_is_zero(&k0)) goto cleanup_fail;

    /* (6) R = k0 * G. */
    secp256k1_ecmult_gen(&ctx->ecmult_gen_ctx, &R_gej, &k0);
    secp256k1_ge_set_gej(&R_ge, &R_gej);
    secp256k1_fe_normalize_var(&R_ge.y);

    /* (7) Force even-y on R; conditionally negate k0. */
    int r_y_odd = secp256k1_fe_is_odd(&R_ge.y);
    secp256k1_scalar_cond_negate(&k0, r_y_odd);
    k = k0;

    /* (8) e = tagged_hash("BIP0340/challenge", R.x || P.x || msg) mod n. */
    tagged_hash3("BIP0340/challenge", R.x_ser32, P.x_ser32, msg32, rand);
    secp256k1_scalar_set_b32(&e, rand, NULL);

    /* (9) s = k + e*d  mod n. */
    secp256k1_scalar_mul(&s, &e, &d);
    secp256k1_scalar_add(&s, &s, &k);

    /* (10) Serialize: sig = R.x || s. */
    secp256k1_fe_get_b32(sig64, &R_ge.x);
    secp256k1_scalar_get_b32(sig64 + 32, &s);

    /* (11) Optional self-check: verify(sig) under P; abort on failure
            (defends against fault attacks). */

cleanup_ok:
    explicit_bzero(&d0, sizeof d0); explicit_bzero(&d, sizeof d);
    explicit_bzero(&k0, sizeof k0); explicit_bzero(&k, sizeof k);
    explicit_bzero(t, sizeof t);
    return 1;
cleanup_fail:
    explicit_bzero(...);
    return 0;
}
```

### Tagged hash helper

```c
void tagged_hash3(const char* tag,
                  const u8* a, const u8* b, const u8* c,
                  u8 out[32])
{
    static u8 cache[64];                          /* SHA(tag)||SHA(tag) */
    static int cached_for = -1;
    if (cached_for != hash_id(tag)) {
        u8 t32[32];
        sha256(tag, strlen(tag), t32);
        memcpy(cache, t32, 32); memcpy(cache+32, t32, 32);
        cached_for = hash_id(tag);
    }
    sha256_ctx s;
    sha256_init(&s);
    sha256_update(&s, cache, 64);
    sha256_update(&s, a, 32);
    sha256_update(&s, b, 32);
    sha256_update(&s, c, 32);
    sha256_final(&s, out);
}
```

The cache trick saves two SHA256 compressions per signature (the tag prefix
is constant). libsecp256k1 hardcodes the precomputed midstate for the three
BIP340 tags.

### Taproot key-path integration

For Taproot, the signer holds the internal key `d_int` and a Merkle root
`mr` of the script tree (or null for key-path-only). The output secret is:

```
t = int(tagged_hash("TapTweak", P_int.x || mr)) mod n
d_out = (d_int + t) mod n     (with parity adjustment)
P_out = P_int + t*G           (with even-y forcing)
```

Then sign with `(d_out, P_out)` as inputs to the standard BIP340 routine.
Critically, if `P_int.y` is odd, you negate `d_int` before adding `t`. If
`P_out.y` is odd after the addition, you negate `d_out`. Two parity flips
that must compose correctly.

## Worked example

Mini-trace for `d = 1`, `aux = zeros`, `m = zeros`:

```
P = G                                  (G has even y, no flip)
d after parity = 1

aux_hash = SHA256d-tagged("BIP0340/aux", zeros) = 0x16E16D8B9...
t        = 1 XOR aux_hash              -> t = aux_hash with low bit flipped

nonce_hash = SHA256d-tagged("BIP0340/nonce", t || G.x || zeros)
k0 = int(nonce_hash) mod n             -> some 256-bit value

R = k0 * G                             -> compute, check parity, possibly negate k0

e_hash = SHA256d-tagged("BIP0340/challenge", R.x || G.x || zeros)
e      = int(e_hash) mod n

s = (k + e * 1) mod n = (k + e) mod n
sig = R.x || s
```

Compare against BIP340 test vector with `d = 1`, `aux = 0` if available;
otherwise re-run reference.py and compare hex byte-for-byte.

## Common pitfalls

- **Variable-time `point_mul`**: if your scalar mul is not CT, an attacker
  with a sidechannel observes `k`'s Hamming weight and recovers `d`.
- **Wiping after error path only**: a fault that returns success early
  may leak `d`. Wipe in both `ok` and `fail` branches.
- **Caching `R` across signatures**: the parity flip on `k` produces a
  different `R` than `k0 * G`; never cache the pre-flip `R`.
- **Self-verify after sign**: an extra verify is a 50% cost increase but
  cheap insurance against fault attacks. Hardware wallets always do this.
- **Mixing little-endian and big-endian**: secp256k1 scalars and field
  elements are big-endian on the wire, but internal libsecp256k1 limbs are
  little-endian arrays. Use the library accessors (`set_b32`, `get_b32`).
- **Mutating the input `sk` buffer**: even if you wipe afterward, callers
  may not expect their buffer to change between calls.

## References

- BIP340 reference.py: https://github.com/bitcoin/bips/blob/master/bip-0340/reference.py
- libsecp256k1 `src/modules/schnorrsig/main_impl.h`.
- Bernstein, Lange, "Failures in NIST's ECC standards" (constant-time discussion).
- "Verify-after-sign: defense against fault attacks", Hardware Wallet Best Practices, Coldcard docs.
