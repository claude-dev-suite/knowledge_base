# TRUC v3 Rules Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/package-relay`.
> Canonical source: BIP431
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/package-relay/SKILL.md

## Concept

TRUC (Topologically Restricted Until Confirmation, transaction
version 3) is a mempool policy that imposes strict ancestor/descendant
limits on transactions opting in via nVersion=3. The skill summarizes
the rules; this article goes into the specific failure modes those
rules prevent (replacement cycling, sibling pinning, low-fee parent
pinning) and the implementation specifics like sibling eviction.

The transactions/truc-v3-rules article covers the Lightning use case;
this article focuses on the package-relay interaction and the
replacement semantics.

## Walkthrough / mechanics

**The seven TRUC rules (BIP431):**

1. **Version = 3.** Tx nVersion field equals 3 (otherwise standard
   policy applies).
2. **Single unconfirmed ancestor.** A v3 tx may have at most ONE
   unconfirmed parent, and that parent MUST also be v3.
3. **Single unconfirmed descendant.** A v3 parent may have at most ONE
   unconfirmed child.
4. **No mixing v3 with non-v3.** All ancestors and descendants must
   be v3 (or already confirmed).
5. **Size limit 10 kvB.** Each individual v3 tx ≤ 10_000 vbytes.
6. **Child size limit 1 kvB.** A v3 child of a v3 parent ≤ 1_000
   vbytes (child is small fee-bumper).
7. **Sibling eviction.** A new v3 tx that wants to replace an
   existing v3 sibling (same v3 parent) is evaluated as a replacement
   even though it doesn't spend the sibling's outputs.

**Why each rule:**

- Rule 2 prevents an attacker from extending a chain of low-fee v3
  txs that an honest party can't economically replace.
- Rule 3 limits descendant fanout; without it, an attacker spending
  multiple outputs of one v3 parent could pin via 25 children
  (default mempool descendant limit).
- Rule 6 keeps the fee-bump child cheap; large children waste
  bandwidth on every replacement.
- Rule 7 (sibling eviction) is the headline feature: in standard
  policy, a competing child only counts if it spends shared inputs
  (BIP125). With sibling eviction, a v3 child can replace a sibling
  that DOESN'T share inputs with it but shares the parent.

## Worked example

**Scenario A: standard CPFP fee bumping (no attack).**

```
v3 parent: commitment tx, fee = 0   # legal under v3
v3 child:  spends parent's anchor, fee = 5000 sat, vsize 150 vB
```

`submitpackage [parent, child]` -> accepted. Effective fee rate
sufficient.

**Scenario B: attacker tries to pin with another child.**

In old (non-v3) policy, an attacker spending a different output of
the parent (say, the remote_balance output, which they control after
HTLC resolution) can attach a low-fee descendant. Honest party tries
to replace via CPFP, but BIP125 rule 5 doesn't fire (no shared
inputs).

In v3 policy:

```
state of mempool:
  parent (v3, fee=0)
  attacker_child (v3, spends parent.out_attacker, fee=200 sat, vsize 200 vB)

Honest party submits:
  honest_child (v3, spends parent.out_anchor, fee=10000 sat, vsize 150 vB)

Mempool sees honest_child arriving and:
  - parent already has 1 descendant (attacker_child) - rule 3 violation
  - sibling eviction kicks in: evaluate honest_child as replacement
    of attacker_child even though they share no inputs.
  - if honest_child fee > attacker_child fee + min_relay_increment,
    attacker_child evicted, honest_child accepted.
```

**Scenario C: attacker tries replacement cycling.**

```
parent (v3, fee=0)
attacker submits: A_child (v3, fee=300, RBF replaces honest_child cycle 1)
honest submits:  H_child (v3, fee=600)
attacker submits: A_child' (v3, fee=900)
... repeat 100 times ...
```

In old policy with full RBF, attacker can cycle their own descendant
to repeatedly evict honest_child via BIP125, exhausting the honest
party's ability to track and respond before HTLC timeouts. v3 rule 6
caps child at 1 kvB - attacker pays the absolute fee for a 1 kvB tx
each cycle - bounding the attack cost.

The 1-descendant limit also means the attacker can't grow the
descendant chain long enough to make replacement uneconomical.

## Common bugs / pitfalls

1. **Mixing v3 with v2 ancestors.** Common mistake: build a v2 funding
   tx, then a v3 commitment that spends from it. v3 commitment fails
   admission - rule 4 forbids non-v3 ancestors except confirmed.
   Confirm the funding first.
2. **Child too large.** Rule 6 caps v3 child at 1 kvB. A child that
   sweeps multiple anchor outputs across channels can hit this limit.
   Split into multiple separate v3 packages.
3. **Sibling eviction expectations.** Sibling eviction only fires if
   the new tx is v3 AND has the same v3 parent. A non-v3 sibling
   competitor doesn't trigger.
4. **Two v3 children of same parent in same package.** Forbidden by
   rule 3. Submit as separate packages (each with the parent confirmed
   between them) or use one bigger child.
5. **Reorgs returning v3 chains to mempool.** When a v3 tx is
   re-added on reorg, all v3 rules re-apply. If multiple descendants
   were confirmed and now return, only one can stay; others evicted.
6. **Ephemeral anchor output not spent in package.** Ephemeral anchors
   (value=0) require atomic spending in the same package. Submitting
   commitment alone (no child spending the anchor) fails policy.

## References

- BIP431: https://github.com/bitcoin/bips/blob/master/bip-0431.mediawiki
- Bitcoin Core implementation: https://github.com/bitcoin/bitcoin/pull/28948
- Replacement cycling research: https://bitcoinops.org/en/newsletters/2023/10/25/
- BOLT3 anchor outputs: https://github.com/lightning/bolts/blob/master/03-transactions.md
