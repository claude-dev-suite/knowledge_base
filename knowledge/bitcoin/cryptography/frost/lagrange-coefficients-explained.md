# FROST Lagrange Coefficients Explained - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/cryptography/frost`.
> Canonical source: Shamir Secret Sharing (1979) + FROST (Komlo & Goldberg 2020)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/cryptography/frost/SKILL.md

## Concept

Shamir Secret Sharing distributes a secret `d` as evaluations `s_l = F(l)` of
a degree-(t-1) polynomial F with `F(0) = d`. To reconstruct `d` from any t
shares, we evaluate the Lagrange interpolation formula at `x = 0`. FROST does
not reconstruct `d` directly — that would defeat the threshold property — but
it uses the **Lagrange coefficient** `lambda_l` for each cooperating participant
so that `sum_l lambda_l * s_l = d` holds **inside** the partial-sig math. The
coefficients depend on which subset of participants is signing this round, so
they are recomputed every session.

## Walkthrough / mechanics

### Lagrange basis at x = 0

Given a cooperating subset `S` of size t containing participant identifiers
`{l_1, l_2, ..., l_t}` (small distinct nonzero integers mod q), the Lagrange
coefficient for participant `l_i` is:

```
lambda_{l_i} = product over l_j in S, j != i of  l_j / (l_j - l_i)   mod q
```

Two key properties:

1. **Sum-to-secret**: `sum_{l in S} lambda_l * F(l) = F(0) = d` for any subset
   S of size >= t.
2. **Public**: `lambda_l` depends only on the participant identifiers in S,
   not on any secret material. Anyone can compute it.

### How FROST uses it

In FROST signing, participant `l` holds share `s_l` and verification share
`S_l = s_l * G`. During Round 2 of signing, `l` computes:

```
c   = TaggedHash("BIP0340/challenge", R.x || P.x || m)
z_l = d_l + rho_l * e_l + lambda_l * s_l * c   mod q
```

The aggregator sums `z = sum_l z_l mod q` and the final sig is `(R.x, z)`.

Verification:
```
z * G ?= R + c * P
```

Why this works:
```
sum_l z_l * G
  = sum_l (d_l G + rho_l e_l G + lambda_l s_l c G)
  = sum_l (D_l + rho_l E_l) + c * sum_l (lambda_l * S_l)
  = R                       + c * P
```

The middle step uses the Sum-to-secret property: `sum_l lambda_l * S_l =
sum_l lambda_l * s_l * G = (sum_l lambda_l * s_l) * G = d * G = P`.

### Modular inversion details

`l_j - l_i` is computed in the scalar field GF(q) — for secp256k1, `q` is the
group order, a 256-bit prime. Inversion uses Fermat's little theorem
(`a^(q-2) mod q`) or the extended Euclidean algorithm. Constant-time
implementations are mandatory because participant IDs are public but timing
side channels could combine with other observations.

```rust
// rust-secp256k1-style pseudo
fn lagrange_coeff(my_id: Scalar, signer_ids: &[Scalar]) -> Scalar {
    let mut num = Scalar::ONE;
    let mut den = Scalar::ONE;
    for &id in signer_ids {
        if id == my_id { continue; }
        num = num * id;                     // mod q
        den = den * (id - my_id);           // mod q  (signed-difference style)
    }
    num * den.invert()                      // den^(q-2) mod q
}
```

### Why subsets matter

The lambda values for a 3-of-5 group depend on which 3 participants are
signing.

```
S = {1,2,3}: lambda_1 = (2*3)/((2-1)(3-1)) = 6/2 = 3
             lambda_2 = (1*3)/((1-2)(3-2)) = 3/(-1) = -3
             lambda_3 = (1*2)/((1-3)(2-3)) = 2/2 = 1

S = {1,2,4}: lambda_1 = (2*4)/((2-1)(4-1)) = 8/3
             lambda_2 = (1*4)/((1-2)(4-2)) = 4/(-2) = -2
             lambda_4 = (1*2)/((1-4)(2-4)) = 2/6 = 1/3
```

A signer in {1,2,3} computes a totally different `lambda` than the same signer
participating in a session with {1,2,4}. Hard-coding lambdas is a common bug.

## Worked example

Toy group with q = 23 (a small prime, for arithmetic clarity), t = 3, n = 5.

Master secret d = 13. Polynomial F(x) = 13 + 4x + 2x^2 mod 23.

Shares:
```
F(1) = 13 + 4 + 2  = 19
F(2) = 13 + 8 + 8  = 29 mod 23 = 6
F(3) = 13 + 12 + 18 = 43 mod 23 = 20
F(4) = 13 + 16 + 32 = 61 mod 23 = 15
F(5) = 13 + 20 + 50 = 83 mod 23 = 14
```

Reconstruct using S = {2, 3, 5}:

```
lambda_2 = (3*5)/((3-2)(5-2)) = 15/3 = 5         mod 23
lambda_3 = (2*5)/((2-3)(5-3)) = 10/(-2)
         = 10 * inverse(-2) = 10 * inverse(21)
         (21 * 11 = 231 = 10*23 + 1; so inverse(21) = 11)
         = 10 * 11 = 110 mod 23 = 18
lambda_5 = (2*3)/((2-5)(3-5)) = 6/(6)            // (-3)*(-2) = 6
         = 6 * inverse(6)
         (6 * 4 = 24 = 23 + 1; inverse(6) = 4)
         = 6 * 4 = 24 mod 23 = 1

sum = lambda_2*s_2 + lambda_3*s_3 + lambda_5*s_5
    = 5*6 + 18*20 + 1*14
    = 30 + 360 + 14
    = 404 mod 23
    = 404 - 17*23 = 404 - 391 = 13 = d   OK
```

Inside FROST, this same arithmetic happens with `q` being secp256k1's group
order, and the participants never publish `s_l` — only the partial sig
`z_l = d_l + rho_l*e_l + lambda_l * s_l * c`.

## Common pitfalls

- **Hard-coding lambdas.** Different sessions = different cooperating subsets
  = different lambdas. Always recompute.
- **Forgetting modular reduction.** `(l_j - l_i)` can be negative as an
  integer; in GF(q) it is `(l_j - l_i) mod q`. Implementations using signed
  i64 then converting to scalar mid-multiplication produce silent corruption.
- **Identifier collision.** Participant IDs must be **distinct nonzero**
  scalars mod q. Reusing 0 means `lambda` blows up; reusing the same ID twice
  causes a divide-by-zero in `(l_j - l_i)`.
- **Including non-cooperating participants in S.** S is the set of signers in
  *this* round, not the full group. A 3-of-5 session with only 3 signers must
  use a 3-element S.
- **Ignoring Pedersen DKG identifier convention.** Some impls use 1..n,
  others use random scalars. Mixing the two between DKG and signing breaks
  the polynomial.

## References

- Komlo & Goldberg, "FROST: Flexible Round-Optimized Schnorr Threshold
  Signatures" (2020): https://eprint.iacr.org/2020/852
- Shamir, "How to share a secret" (1979): https://dl.acm.org/doi/10.1145/359168.359176
- ZF FROST: https://github.com/ZcashFoundation/frost
- draft-irtf-cfrg-frost sec. "Polynomials":
  https://datatracker.ietf.org/doc/draft-irtf-cfrg-frost/
