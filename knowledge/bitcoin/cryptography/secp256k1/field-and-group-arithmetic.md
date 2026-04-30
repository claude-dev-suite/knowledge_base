# secp256k1 Field and Group Arithmetic - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/cryptography/secp256k1`.
> Canonical source: SEC 2 v2 (https://www.secg.org/sec2-v2.pdf), libsecp256k1 source.
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/cryptography/secp256k1/SKILL.md

## Concept

secp256k1 has two distinct algebraic structures: the prime field `F_p` where
`p = 2^256 - 2^32 - 977` (used for x and y coordinates), and the cyclic group
of points on `y^2 = x^3 + 7 (mod p)` of order `n` (used for scalars and
signatures). Confusing them is the most common bug in EC code: an x-coordinate
is reduced mod `p`, a scalar is reduced mod `n`, and `p > n` by exactly
`0x14551231950B75FC4402DA1722FC9BAEE` (~2^128), so reductions silently differ.

## Walkthrough / mechanics

### Field reduction with the secp256k1 form

The prime is `p = 2^256 - c` with `c = 2^32 + 977 = 0x1000003D1`. Any 512-bit
product `x = x_hi * 2^256 + x_lo` reduces as:

```
x mod p = x_lo + x_hi * c   (then one more conditional subtraction of p)
```

libsecp256k1 represents field elements as five 52-bit limbs (so `n*x` keeps a
26-bit headroom before overflow). Reduction touches only the top limb plus a
multiplication by the 33-bit constant `c`. No general 256-bit modular
reduction is needed.

### Point addition (affine) - chord and tangent

For `P = (x1, y1)`, `Q = (x2, y2)`, `P + Q = (x3, y3)` with `P != +/-Q`:

```
lambda = (y2 - y1) * (x2 - x1)^(-1)   mod p
x3 = lambda^2 - x1 - x2               mod p
y3 = lambda * (x1 - x3) - y1          mod p
```

For doubling (`P == Q`, `y != 0`):

```
lambda = (3*x1^2) * (2*y1)^(-1)        mod p   (a = 0 simplifies this)
x3 = lambda^2 - 2*x1                   mod p
y3 = lambda * (x1 - x3) - y1           mod p
```

The `a = 0` choice in secp256k1 (Koblitz curve) eliminates the `+ a` term
present in generic short Weierstrass formulas, saving one field addition per
doubling. This compounds across 256 doublings in scalar multiplication.

### Jacobian coordinates

Affine arithmetic costs one field inversion per addition (~80x a multiply).
Jacobian projective coordinates `(X, Y, Z)` represent the affine point
`(X/Z^2, Y/Z^3)`. Addition becomes inversion-free; one inversion is needed
only at the end to convert back to affine. This is what libsecp256k1 actually
uses internally.

## Worked example

Compute `2 * G` with concrete numbers (truncated for legibility):

```
G.x = 0x79BE667E F9DCBBAC 55A06295 CE870B07 029BFCDB 2DCE28D9 59F2815B 16F81798
G.y = 0x483ADA77 26A3C465 5DA4FBFC 0E1108A8 FD17B448 A6855419 9C47D08F FB10D4B8

3 * G.x^2 mod p =
  0x9D17... (compute)
2 * G.y mod p =
  0x9075B4EE 4D478D8A BB49F7F8 1C2211...

lambda = num * den^{-1} mod p
       = 0xC6047F94 41ED7D6D 3045406E 95C07CD8 5C778E4B 8CEF3CA7 ABAC09B9 5C709EE5

(2G).x = lambda^2 - 2*G.x  mod p
       = 0xC6047F94 41ED7D6D 3045406E 95C07CD8 5C778E4B 8CEF3CA7 ABAC09B9 5C709EE5
(2G).y = lambda * (G.x - (2G).x) - G.y mod p
       = 0x1AE168FE A63DC339 A3C58419 466CEAEE F7F63265 3266D0E1 236431A9 50CFE52A
```

These match `secp256k1_ec_pubkey_create` for the scalar `2`. Bookmark them as
quick sanity checks.

## Common pitfalls

- **Mixing `mod p` and `mod n`**: `r = R.x mod n` for ECDSA - never `mod p`.
  The two differ in the high `~2^128` range; bugs surface only on rare inputs.
- **Skipping the curve-equation check**: accepting `(x, y)` without verifying
  `y^2 == x^3 + 7 mod p` enables invalid-curve attacks against any code that
  later does scalar multiplication.
- **Using `y == 0` doubling**: there is no point with `y == 0` on secp256k1
  (it would require `x^3 = -7 mod p`, which has no solution since `-7` is a
  cubic non-residue mod p). But generic library code must still guard.
- **Variable-time inversion**: `pow(a, p-2, p)` via square-and-multiply leaks
  `a` via timing. Use the constant-time addition-chain inversion in
  libsecp256k1 (`secp256k1_fe_inv`).
- **Endianness in serialization**: SEC1 encodes coordinates big-endian; many
  test harnesses (especially Python `int.from_bytes`) silently default to the
  wrong endianness.

## References

- SEC 2 v2: https://www.secg.org/sec2-v2.pdf
- libsecp256k1 source: https://github.com/bitcoin-core/secp256k1
- "Guide to Elliptic Curve Cryptography" (Hankerson, Menezes, Vanstone) ch. 3.
- Pieter Wuille, "The secp256k1 endomorphism is fast" notes.
