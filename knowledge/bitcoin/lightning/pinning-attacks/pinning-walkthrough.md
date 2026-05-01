# HTLC Pinning Attack - Walkthrough Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/pinning-attacks`.
> Canonical source: https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-April/002639.html
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/pinning-attacks/SKILL.md

## Concept

A **pinning attack** abuses Bitcoin mempool policy to prevent the victim
from broadcasting / confirming a Lightning HTLC enforcement transaction
in time. The attacker submits a low-feerate large transaction that is
descended from the conflicting output, "pinning" it in mempool: the
victim cannot replace the parent without also replacing all descendants,
and the descendants combined exceed the victim's affordable replacement
fee.

Pinning is a precursor to the replacement-cycling attack (October 2023)
but predates it conceptually.

## Walkthrough / mechanics

### Setup

- Channel: Alice — Bob — Carol.
- Alice routed an HTLC to Carol via Bob (100,000 sat).
- Bob is the victim; Alice and Carol collude.
- Channel is non-anchor (legacy "BOLT-03 v1").

### Pinning sequence

1. **Force close**: Carol broadcasts the Bob-Carol commitment tx
   (commitment_carol).
2. Bob plans to broadcast HTLC-success once commitment confirms.
3. Before commitment confirms, Carol broadcasts a **descendant
   transaction** spending another output of commitment_carol (e.g.
   her own balance output). Critically, the descendant has:
   - Very low feerate (e.g. 1 sat/vB).
   - Very large size (e.g. 100 KB, padded with junk if needed).
4. Mempool now contains:
   `commitment_carol -> descendant_carol`.
5. Carol's descendant has a low fee but high size; replacement
   policy (BIP-125 rule 3) requires Bob's replacement to pay strictly
   more total fee than the descendant + parent combined.
6. Bob tries to replace the HTLC output spend. Per RBF rules, his
   replacement must also spend output #X of commitment_carol. The
   replacement must outbid the descendant (which is large but low
   fee per byte) on **total fee**, not feerate.
7. Bob's replacement fee budget = some reasonable fee for a normal
   HTLC tx (~5,000 sat). Pinned descendant has fee ~10,000 sat (high
   total because of size). Bob can't outbid economically.

### Why the descendant matters

BIP-125 rule 3: replacement transaction must pay an absolute fee
exceeding the sum of fees of the conflicting transactions and their
descendants. By inflating the descendant's size + fee, Carol forces
Bob to pay a prohibitive amount.

### Timing

The attack works only until Bob's CLTV timeout. If Bob can outwait the
attacker (preimage-based path requires Bob to claim before CLTV expires
or Carol can reverse-claim), attacker wins.

### Carol's funds at risk?

Carol's pinning descendant is a real spend; she pays the descendant's
fee. But she gets back the HTLC value if Bob fails to claim — net
profitable.

### Anchor channels mitigation

In **anchor channels** (BOLT-03 v2), commitment_carol has small anchor
outputs (330 sat) that Bob can use as a CPFP fee-bump source. He can
attach a child transaction with high feerate to anchor, bumping the
commitment without needing to RBF. However, before TRUC v3 (BIP-431),
this can still be cycled.

## Worked example

```
Block H: Carol broadcasts commitment_carol (size 250 vB, fee 1000 sat).
Block H: Carol broadcasts descendant_carol:
   Inputs: commitment_carol output 0 (Carol's main balance).
   Outputs: padded with OP_RETURN to size 100 KB.
   Fee: 10,000 sat (effective rate: 0.1 sat/vB; total fee high).

Mempool:
   commitment_carol (fee 1000)
   descendant_carol (fee 10,000)

Bob wants to broadcast HTLC-success(Bob):
   Inputs: commitment_carol output 1 (HTLC).
   Fee budget: 5000 sat.

Per RBF: Bob's tx replaces commitment_carol's mempool position only if
Bob's tx pays > 1000 + 10,000 = 11,000 sat total.

Bob can't economically pay 11,001 sat for a 110-vB tx (~100 sat/vB).
He's stuck.

Time passes. CLTV timeout passes. Carol broadcasts HTLC-timeout to
claim the HTLC value back.
Bob loses 100,000 sat.
```

## Common pitfalls

- **Underestimating mempool inheritance**: any descendant of the
  conflicting output forces Bob's replacement to pay for the entire
  package.
- **Big descendants**: legitimate users sometimes have multi-input
  descendant txs they cannot easily rebuild; attacker uses this to
  craft pinning.
- **Time-pressure mistake**: if Bob's monitoring loop polls infrequently,
  attacker has more time to land pinning.
- **Anchor channels alone insufficient**: cycling-resistant designs
  (TRUC v3) needed for full mitigation.

## References

- Matt Corallo. "Pinning attacks" lightning-dev list, April 2020.
- BIP-125 (RBF) replacement rules.
- BOLT-03 anchor channels.
- BIP-431 (TRUC) — full mitigation.
