# BIP340 Verification - Deep Dive on Edge Cases

> Phase B article. Companion to dev-suite skill `bitcoin/cryptography/schnorr`.
> Canonical source: BIP340 sect. "Verification".
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/cryptography/schnorr/SKILL.md

## Concept

BIP340 verification is deceptively simple in pseudocode but its security
depends on a strict order of validation: range checks first, lift_x next,
then the algebraic check. Skipping or reordering steps opens specific
forgery surfaces. This article unpacks each step, what attack each blocks,
and how libsecp256k1 structures the implementation for speed and safety.

## Walkthrough / mechanics

### Inputs and the spec text

```
Input:
    pk    : 32 bytes (x-only public key)
    msg   : 32 bytes
    sig   : 64 bytes  (sig[0:32] = r,  sig[32:64] = s)

Output: TRUE or FALSE
```

The spec defines the verifier as:

```
1. Let P = lift_x(int(pk)); fail if P is None.
2. Let r = int(sig[0:32]); fail if r >= p.
3. Let s = int(sig[32:64]); fail if s >= n.
4. Let e = int(tagged_hash("BIP0340/challenge", sig[0:32] || pk || msg)) mod n.
5. Let R = s*G - e*P.
6. Fail if is_infinite(R).
7. Fail if not has_even_y(R).
8. Fail if x(R) != r.
9. Return TRUE.
```

The order matters: steps 2-3 are cheap range checks; lift_x in step 1 is
~1 field exponentiation. The expensive step is 5 (two scalar muls). Failing
fast on cheap checks is critical for DoS resistance during block validation.

### Why each check is necessary

**lift_x rejection**. About half of all 256-bit integers `x < p` are not
valid x-coordinates (the cubic `x^3 + 7 mod p` is non-quadratic-residue).
If a verifier accepts those by computing a phantom sqrt, an attacker can
construct fake "public keys" that pass all later checks for chosen
signatures.

**r < p**. `r` is the x-coordinate of `R`, which lives in `F_p`. If `r >= p`
the encoding is meaningless. Pre-BIP340 toy implementations sometimes
checked `r < n` instead - wrong, because `n < p`.

**s < n**. `s` is a scalar; `s = n` would mean the signer leaked `s = 0`
implicitly (since `s mod n = 0`). Spec is explicit: `s in [0, n)`. Some
verifiers check `s != 0` separately; not strictly required because step 6
(infinity check) catches the degenerate case.

**Even y of R**. Without this, `(R.x, even_y)` and `(R.x, odd_y)` both
satisfy `s*G - e*P` for two different `(s, e)` pairs - third-party
malleability. Forcing even y collapses this to a unique canonical form.

**x(R) == r**. The actual algebraic check. If lift_x produces a point with
x = r and even y, and the spec's `s, e` decompose correctly, `s*G - e*P`
must equal that exact point.

### Computational shortcut

The reference computes `R = s*G - e*P` but the verifier never needs `R.y`
explicitly except for the parity check. libsecp256k1 uses Jacobian
coordinates throughout:

```c
secp256k1_gej R_jac;
secp256k1_ecmult(&ctx->ecmult_ctx, &R_jac, &P_jac, &neg_e_scalar, &s_scalar);
/* R_jac = s*G + (-e)*P */

if (secp256k1_gej_is_infinity(&R_jac)) return 0;

secp256k1_fe rx, ry;
secp256k1_ge R;
secp256k1_ge_set_gej(&R, &R_jac);
secp256k1_fe_normalize_var(&R.y);
if (secp256k1_fe_is_odd(&R.y)) return 0;

secp256k1_fe_normalize_var(&R.x);
return secp256k1_fe_equal_var(&R.x, &r_target_fe);
```

The `_var` variants are used because by the verification stage we have
already accepted public input - timing leaks reveal nothing about secret
state (there is none).

### The "x(R) == r without computing y" trick

For batch verification, computing `R.y` for each verification is wasteful.
Actually computing only `R.x` in projective form lets the verifier compare
`R.X * R.Z^{-2} mod p == r`, with the inversion saved by cross-multiplying:

```
R.X == r * R.Z^2 mod p   AND   R.Z != 0
```

Plus a separate check for even y via the Jacobi symbol of `R.Y * R.Z`. This
avoids one inversion per signature.

## Worked example

Construct a verification-FALSE case to test rejection paths.

Take a valid signature `(r, s)` for a key `P` and message `m`. Now flip the
last bit of `s`:

```
s' = s ^ 1
```

The challenge `e` does not change (it depends on `r, P.x, m` only). So the
new claimed `R'` is:

```
R' = s'*G - e*P
   = (s + 1)*G - e*P
   = s*G - e*P + G
   = R + G
```

`R + G` has different x-coordinate from `R` with overwhelming probability,
so step 8 (`x(R') == r`) fails. The verifier rejects. Good.

Now flip the high bit of `r`:

```
r' = r XOR 0x80000000_..._00000000
```

If the flipped `r'` happens to be a valid x-coordinate of some point, the
verifier reaches step 8 and rejects (`r' != computed_x`). If `r'` is not a
valid x-coordinate, step 1's lift_x already returned None (but lift_x is
applied to `pk` not `r` in this protocol - actually only to `pk`). The
relevant check is step 5's `R = s*G - e*P` lookup against `r'`. If `r' >= p`,
step 2 rejects directly. If `r' < p`, the algebra fails at step 8.

This walkthrough is exactly what BIP340 vectors 7-11 codify.

## Common pitfalls

- **Doing lift_x on R before computing it**: redundant. R is computed by
  algebra; only its x and y parity need to be checked.
- **Using SHA256(SHA256(...))** instead of `SHA256(SHA256(tag) || SHA256(tag) || msg)`:
  this is a perennial confusion - the tagged-hash framework is NOT a
  double-SHA, it is `SHA(prefix || msg)` where the prefix is itself a
  precomputed double-tag.
- **Variable-time mul on the verifier**: not strictly a vulnerability since
  there's no secret, but allows DoS via crafted inputs that hit slow paths.
  Use `_var` only in clearly-public-input verifiers.
- **Skipping the infinity check (step 6)**: if `s = e * d` exactly, you get
  `s*G - e*P = 0`. Without the check the verifier proceeds to extract
  coordinates of point-at-infinity = undefined behavior.
- **Accepting empty pubkey or sig buffers**: length must be exactly 32 and
  64. Many language bindings have off-by-one in slice handling.
- **Caching `e`**: fine, but if you cache the message and the message is
  later mutated (mutable buffer in the caller), you produce stale `e`.

## References

- BIP340 sect. "Verification": https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki
- libsecp256k1 `src/modules/schnorrsig/main_impl.h` `secp256k1_schnorrsig_verify`.
- Bernstein, "Curve25519: new Diffie-Hellman speed records" - on x-only batch math.
- Tim Ruffing, "BIP340 verification edge cases" (delvingbitcoin post 2023).
