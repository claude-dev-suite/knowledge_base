# CET Tree for Numerical Outcomes - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/cryptography/dlcs`.
> Canonical source: dlcspecs NumericOutcome.md + Crypto Garage research notes
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/cryptography/dlcs/SKILL.md

## Concept

A naive DLC over a numeric range with K-bit precision needs `2^K` CETs — one
per possible outcome — each requiring an adaptor pre-signature exchange
between the parties. For a 20-bit price oracle (~1M resolution) that is over
a million CETs per contract, prohibitive. The CET tree compresses this by
exploiting the **multi-nonce attestation**: the oracle commits to one nonce
per digit, and a CET pre-signed under a *prefix* of bits is satisfied by any
outcome consistent with that prefix. A binary tree of CETs reduces storage
from `O(2^K)` to `O(K * #payout_intervals)` typical. This article shows the
construction and the adaptor-point algebra that makes it work.

## Walkthrough / mechanics

### Multi-nonce setup recap

```
Oracle commits per digit i in 0..K-1:
  k_i <- random
  R_i = k_i * G
Announcement = (P_oracle, [R_0, ..., R_{K-1}])
```

Outcome `V = b_{K-1} ... b_1 b_0` (in binary). Oracle publishes one Schnorr
sig per digit:

```
e_{i, b}  = H(R_i || P_oracle || str(b))    for digit i and bit b in {0,1}
s_i        = (k_i + e_{i, b_i} * d_oracle) mod n      for the actual b_i
```

`T_{i, b} = lift_x(R_i) + e_{i, b} * P_oracle` is the adaptor point that
matches "digit i is b". An attestation publishes only `s_i` for the actual
`b_i`; the *other* `T_{i, 1-b_i}` remains an unmatched commitment.

### CET semantics and prefix-CETs

A "prefix-CET" is a CET pre-signed under an adaptor point that combines
several digit-points:

```
For prefix P = (b_{K-1}, b_{K-2}, ..., b_j) covering the top K-j bits
(low j bits are "don't care"):

  T_P = sum over i in {j, j+1, ..., K-1} of T_{i, b_i}

  Pre-sign CET_P with adaptor point T_P.
```

The adaptor secret needed to finalize CET_P is:

```
t_P = sum over i in {j, ..., K-1} of s_i      // sum of attested scalars
                                              // for the fixed bits
```

Any actual outcome `V` whose top bits match `P` will produce the right
`s_i` values for `i >= j`, so `t_P = sum_{i>=j} s_i` matches exactly.

### Binary tree layout

Build a tree over the outcome range:

```
Root       = whole range [0, 2^K)
Internal node at depth d = range covering 2^{K-d} outcomes,
                           fixes top d bits
Leaf node  = single outcome (depth K)
```

For a payout function with `r` distinct payout intervals, we cover the
`[0, 2^K)` range with the **smallest set of tree nodes** whose ranges are
exactly the payout intervals. Each chosen node becomes one prefix-CET.

```
def cover_intervals(intervals, K):
    # intervals: list of [lo, hi] payout-rate-constant ranges
    # returns: list of (depth, prefix_bits, payout_a, payout_b) tuples
    cets = []
    for (lo, hi) in intervals:
        # decompose [lo, hi] into a minimal set of binary-tree nodes
        nodes = max_aligned_subranges(lo, hi, K)
        for n in nodes: cets.append((n.depth, n.prefix, n.payout))
    return cets
```

A piecewise-constant payout function with `r` segments in a `K`-bit range
yields at most `O(r * K)` tree nodes total — orders of magnitude less than
`2^K`.

### Adaptor combinator algebra

The CET tree relies on the adaptor sig being **homomorphic** under point
addition for the adaptor offset:

```
PreSign(d, m, T_1) gives s'_1 = k + e * d - log(T_1) ... (informally)
Actually: s'_1 = k + e * d, with verifier checking against R + T_1.

For combined offset T_combined = sum T_i:
  PreSign(d, m, T_combined) gives s' = k + e * d
  Verifier checks against R + T_combined.

Adapt with t_combined = sum t_i:
  s = s' + t_combined = k + e*d + sum t_i
```

The adaptor *point* combination is just point addition; the adaptor *secret*
combination is just scalar addition. Both are abelian, so any subset of
digit-attestations can be summed in any order.

### Storage and verification cost

```
| Resolution | Naive CETs | Tree CETs (3 payout segments) |
|-----------|-----------|-------------------------------|
| 8 bits    | 256       | ~24 (3 * 8)                   |
| 16 bits   | 65536     | ~48                           |
| 20 bits   | 1048576   | ~60                           |
```

Each prefix-CET is one tx + two adaptor sigs (one per party) ≈ a few hundred
bytes. Total storage is megabytes for naive, kilobytes for tree.

### Numerical pitfalls: parity and signs

Each `T_{i, b}` may have any parity. When summing:

```
T_P = sum T_{i, b_i}
```

the result may have odd-y. BIP340 verification requires even-y, so the
adaptor sig protocol applies a parity flip on the *combined* point. Each
party computes the parity of `T_P` and adjusts their pre-sig and the eventual
adapt step accordingly.

For multi-attestation `s_i`, parties must verify each `s_i * G == lift_x(R_i)
+ e_i * P_oracle` (i.e., the per-digit Schnorr sig is valid) before summing
into `t_P`; otherwise a malicious oracle could publish bogus `s_i` values
that don't correspond to any actual outcome.

## Worked example

3-bit precision (`K = 3`), outcomes `[0..7]`. Payout function:

```
0 -> Alice 100, Bob 0
1, 2, 3 -> Alice 70, Bob 30
4, 5, 6, 7 -> Alice 0, Bob 100
```

Three payout segments. Tree decomposition:

```
[0]:        leaf, prefix = 000             (depth 3)
[1, 3]:     covered by ranges [1] (001), [2,3] (01x)
            -> three nodes if exact, but [1, 3] is not power-of-2 aligned
            actually [1] = 001 and [2, 3] = 01x is only [2,3] minus exclusion
            For [1, 2, 3]: nodes are 001 (leaf), 01x (depth 2)
[4, 7]:     covered by 1xx (depth 1)

Total CETs: 4 (one for [0], two for [1..3], one for [4..7])
```

Adaptor points:

```
For node 000 (depth 3):
  T = T_{2,0} + T_{1,0} + T_{0,0}    (all three digits = 0)
For node 001 (depth 3):
  T = T_{2,0} + T_{1,0} + T_{0,1}
For node 01x (depth 2):
  T = T_{2,0} + T_{1,1}              (depth 2 fixes top 2 bits, digit 0 don't-care)
For node 1xx (depth 1):
  T = T_{2,1}                        (fixes top bit only)
```

Each party pre-signs four CETs with these four adaptor points. When the
oracle attests outcome = 5 = binary 101:

```
s_2 (for b_2 = 1)
s_1 (for b_1 = 0)
s_0 (for b_0 = 1)
```

The applicable CET is node 1xx (since 5 starts with bit 1). Adaptor secret:

```
t_{1xx} = s_2     (only digit 2 is fixed in this prefix)
```

The party adapts the CET[1xx] pre-sig with `t_{1xx} = s_2` and broadcasts.

Notice the bettor only needs `s_2` from the attestation, not all three digit
sigs — the lower-depth CET requires fewer attestation values, which makes
the math more robust to partial oracle outages.

## Common pitfalls

- **Tree-cover algorithm bugs.** Decomposing arbitrary `[lo, hi]` ranges
  into minimal aligned subranges is non-trivial; off-by-one creates a CET
  for an empty range or fails to cover an outcome at all. dlcspecs has a
  reference algorithm and test vectors — use it.
- **Partial attestation.** If the oracle publishes only some `s_i` values
  (e.g., truncates), the bettor may not have enough to form `t_P` for the
  needed depth. Mitigation: use the lowest-depth (most ambiguous) CET that
  still resolves the payout; only top-bit attestation may be needed.
- **Adaptor parity skipped on combined `T_P`.** Each summand has its own
  parity; the **sum** has an independent parity. Pre-signing must use the
  parity of the sum, not the parity of any summand.
- **CET selection ambiguity.** When two non-overlapping CETs both match
  (e.g., a leaf and an ancestor with the same fixed prefix), broadcast the
  one most favorable / most likely to confirm — the duplicate is structural,
  not a bug.
- **Oracle commits but doesn't sign all digits.** If the oracle publishes
  some `s_i` and not others, only CETs depending on the published digits
  are spendable. Refund must remain reachable to recover when this happens.

## References

- dlcspecs NumericOutcome.md:
  https://github.com/discreetlogcontracts/dlcspecs/blob/master/NumericOutcome.md
- Crypto Garage research, "DLC Numerical Outcomes":
  https://medium.com/cryptogarage/dlcs-numerical-outcomes-cdc97e7d12a3
- rust-dlc numerical contract examples:
  https://github.com/p2pderivatives/rust-dlc/tree/master/dlc-trie
- Atomic Finance, "Practical DLC Range Contracts":
  https://atomic.finance/blog
