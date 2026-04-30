# Spark Leaf Architecture - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/spark`.
> Canonical source: https://www.spark.money/ and https://docs.spark.money/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/spark/SKILL.md

## Concept

A Spark **leaf** is the atomic off-chain ownership unit. Where Ark uses a per-round
binary tree of VTXOs and Mercury uses a single 2-of-2 UTXO per position, Spark uses
*hierarchical leaves under a continuously-maintained tree*. Each leaf is an off-chain
UTXO under a FROST k-of-n threshold key held by Spark Operators (SOs), with a
relative-timelock unilateral-exit transaction baked in. Leaves can be split or merged
at any time without waiting for a round, which is what gives Spark its "instant
continuous transfer" property.

## Walkthrough / mechanics

### Tree topology

```
              [On-chain root UTXO held by SO collective]
                              |
                  [FROST-signed off-chain root]
                  /            |             \
              [Branch]      [Branch]        [Branch]
              /     \         |              /    \
           [Leaf] [Leaf]    [Leaf]        [Leaf] [Leaf]
           Alice  Bob        Carol         Dave   Eve
```

- The root has a single on-chain anchor.
- Internal branches are off-chain pre-signed splits, refreshed cooperatively as users
  transfer.
- Leaves are owned by individual users. Each leaf carries:
  - A FROST-aggregated public key authorising cooperative spends.
  - A *unilateral exit script*: relative-timelock path spendable by the user alone.

### FROST signing

Spark uses **FROST (Flexible Round-Optimised Schnorr Threshold)** rather than MuSig2
because FROST allows k-of-n with k < n, supports robust signing (continuing without
non-responsive parties), and produces standard Schnorr signatures indistinguishable
from single-sig on chain. A typical Spark configuration is 2-of-2 (Lightspark + Flashnet)
during beta; the roadmap expands this to 5-of-7 or 7-of-11 across jurisdictions.

### Continuous transfer

To transfer 5,000 sat from Alice's leaf to Bob:

1. Alice signs a "transfer authorisation" with her key share to the SOs.
2. SOs run a FROST signing ceremony to:
   - Spend Alice's old leaf.
   - Create two new leaves: one for Alice (change), one for Bob (5,000 sat).
3. The new leaves replace the old in the tree. No on-chain tx; only an internal
   re-pre-signing of the branch.

### Unilateral exit

Each leaf has a CSV-locked exit branch. If the SOs vanish:

1. The user broadcasts the chain of pre-signed branch txs from root anchor down to their
   leaf.
2. After CSV expiry on the leaf, the user spends the leaf to a normal address.
3. Cost: O(tree depth) txs, typically 4-6 in beta.

The exit tree is recomputed every time a transfer happens; old branches become invalid
once the FROST collective signs the replacement. This is the property that makes
"instant" transfer compatible with self-custody.

## Worked example

```
Initial state:
  Root UTXO: 0.5 BTC on-chain at FROST(2-of-2)
  Tree:
    Root
      L: Branch1 -> [ Alice 0.1 BTC, Bob 0.05 BTC ]
      R: Branch2 -> [ Carol 0.2 BTC, Dave 0.15 BTC ]

Alice pays Bob 0.02 BTC on Spark:
  - Alice signs "transfer 0.02 from leaf_alice_0.1 to leaf_bob_0.05"
  - SOs FROST-sign new Branch1' = [ Alice 0.08, Bob 0.07 ]
  - Pre-signed exit txs for old leaves are invalidated by the SOs deleting
    their old shares (FROST key refresh).
  - Total time: < 1 second. No Bitcoin tx broadcast.

Alice pays a Lightning invoice for 0.01 BTC:
  - Alice's wallet asks SOs to debit leaf_alice_0.08 by 0.01.
  - SOs hold the debit, pay the LN invoice from Lightspark's LN routing
    node, on success commit the debit (FROST resign of Branch1').
  - On HTLC timeout, the SOs revert: no leaf change.
```

## Trade-offs and security

- **k-of-n trust on SO collective**: secure as long as at least n-k+1 honest SOs refuse
  to collude. Beta with 2-of-2 means *both* must be honest -- weaker than a 5-of-7
  setup but stronger than statechain 1-of-1.
- **FROST liveness**: requires at least k operators online. Spark targets >99.99% by
  geo-distributing SOs.
- **Exit-tree refresh costs**: every transfer requires re-pre-signing of the relevant
  subtree branches. This is bounded by FROST cost (~tens of ms per signature) and is the
  main throughput limit.
- **State synchronisation**: clients must hold the latest pre-signed exit branch chain.
  Loss = inability to unilaterally exit (cooperative still works). SOs help with state
  recovery in beta.
- **Compared to Ark**: Spark trades the round-batching efficiency for instant transfer.
  No "round wait", but on-chain footprint per leaf does not amortise the way it does in
  an Ark pool tx. Spark relies on SOs charging fees rather than amortising chain costs.

## References

- Lightspark, "Spark Architecture" - https://docs.spark.money/architecture
- Spark whitepaper, April 2025
- David Marcus interview, Bitcoin Magazine 2025
- FROST RFC - https://datatracker.ietf.org/doc/draft-irtf-cfrg-frost/
