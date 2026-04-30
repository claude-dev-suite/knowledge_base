# secp256k1 GLV Endomorphism Optimization - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/cryptography/secp256k1`.
> Canonical source: Gallant-Lambert-Vanstone (2001), libsecp256k1 ecmult_const.c.
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/cryptography/secp256k1/SKILL.md

## Concept

secp256k1 admits an efficiently computable group endomorphism `phi(P) = lambda * P`
for a known scalar `lambda`. Computing `phi(P)` only takes one field
multiplication (multiply x by a constant `beta`), not a full scalar
multiplication. This lets us split a scalar `k` into two ~128-bit halves
`k1, k2` such that `k*P = k1*P + k2*phi(P)`, halving the doubling cost in
scalar mul. This is the GLV (Gallant-Lambert-Vanstone) method, and is the
primary reason libsecp256k1 outperforms generic ECC libraries.

## Walkthrough / mechanics

### The endomorphism constants

```
beta   = 0x7AE96A2B 657C0710 6E64479E AC3434E9 9CF04975 12F58995 C1396C28 719501EE
lambda = 0x5363AD4C C05C30E0 A5261C02 8812645A 122E22EA 20816678 DF02967C 1B23BD72
```

For any curve point `P = (x, y)`, `lambda * P = (beta * x, y)` where `beta`
is a primitive cube root of unity in `F_p` (so `beta^3 = 1 mod p`) and
`lambda` is a primitive cube root of unity in the scalar ring `Z_n` (so
`lambda^3 = 1 mod n`). Verification: `beta^3 mod p == 1`, `lambda^3 mod n == 1`.

### The split

Given `k`, find small `k1, k2` with `k = k1 + k2 * lambda mod n`. The key
insight: there exists a basis of `Z_n` lattice vectors whose components are
~`sqrt(n) ≈ 2^128` in magnitude. Pre-computed for secp256k1:

```
a1 =  0x3086D221A7D46BCDE86C90E49284EB15
b1 = -0xE4437ED6010E88286F547FA90ABFE4C3
a2 =  0x114CA50F7A8E2F3F657C1108D9D44CFD8
b2 =  0x3086D221A7D46BCDE86C90E49284EB15
```

Algorithm:

```
c1 = round(b2 * k / n)
c2 = round(-b1 * k / n)
k1 = k - c1*a1 - c2*a2
k2 =   - c1*b1 - c2*b2
```

Both `k1, k2` end up in `[-2^128, 2^128]`. Either may be negative; if so,
negate it and negate the corresponding point (free for affine: just flip y).

### Combined scalar multiplication

Now `k * P = k1 * P + k2 * phi(P)`. Use a simultaneous (Shamir-trick)
double-and-add over both 128-bit scalars:

```
acc = O                                       (identity)
for i = 127 downto 0:
    acc = 2 * acc
    if bit_i(k1): acc = acc + P
    if bit_i(k2): acc = acc + phi_P           (where phi_P.x = beta * P.x)
return acc
```

Cost: 128 doublings (instead of 256) and ~128 additions. With wNAF (window
size 5), this drops further to ~25 additions plus 128 doublings - roughly
1.4x faster than naive 256-bit double-and-add even ignoring the doubling
savings.

## Worked example

Take `k = 0x1` (trivial). Split:

```
c1 = round(b2 / n) = 0
c2 = round(-b1 / n) = 0
k1 = 1 - 0 - 0 = 1
k2 = 0
```

So `1 * G = 1 * G + 0 * phi(G)` - degenerates to a normal scalar mul, as
expected.

Take `k = lambda` (so `k*G == phi(G)`):

```
k1 should be 0,  k2 should be 1
phi(G) = (beta * G.x mod p, G.y)
       = (0x7AE96A2B657C07106E64479EAC3434E99CF0497512F58995C1396C28719501EE *
          0x79BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798 mod p,
          0x483ADA7726A3C4655DA4FBFC0E1108A8FD17B448A685541999C47D08FFB10D4B8)
       = (0xFD17B4488C432F4D9050BBA1FE1D5717E1B6...   ,   G.y)
```

Verify against `lambda * G` computed by naive scalar mul - they must match
exactly. libsecp256k1's `bench_internal` exercises this constantly.

## Common pitfalls

- **Wrong sign of `lambda`**: there are two primitive cube roots; using the
  wrong one gives `phi(P) = -lambda * P` and the split produces wrong answers
  for half of inputs. Always pin `lambda` against the canonical SEC value
  above.
- **Constant-time conditional**: `if k1 < 0: k1 = -k1; P = -P` must be
  branchless in production code. libsecp256k1 uses bit-tricks (XOR with a
  mask derived from sign).
- **Using endomorphism for ECDSA verification only**: it's safe everywhere -
  there is no patent issue (the original GLV patent expired in 2020), and
  Bitcoin Core enabled `--with-ecmult-gen-precision=ecmult-gen` by default.
- **Skipping the lattice round**: naive `round(b2*k/n)` with floats loses
  precision around 2^256; must use exact integer arithmetic with bias
  correction (libsecp256k1 uses Lemire-style high-low split).
- **Recomputing phi on every multiply**: cache `beta` as a precomputed
  constant; the optimization is `1 mul` per call, not 1 inversion.

## References

- Gallant, Lambert, Vanstone, "Faster point multiplication on elliptic curves
  with efficient endomorphisms" (CRYPTO 2001).
- libsecp256k1 `src/scalar_impl.h`, `src/ecmult_impl.h`.
- Pieter Wuille, "The secp256k1 endomorphism" (bitcoin-dev 2014).
- Bos, Costello, Hisil, Lauter "Fast cryptography in genus 2" sect. on GLV.
