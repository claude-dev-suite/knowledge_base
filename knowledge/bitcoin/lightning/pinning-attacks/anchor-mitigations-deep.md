# Anchor Output Pinning Mitigations - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/pinning-attacks`.
> Canonical source: https://github.com/lightning/bolts/blob/master/03-transactions.md (anchor outputs)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/pinning-attacks/SKILL.md

## Concept

**Anchor outputs** were introduced to BOLT-03 (channel transactions
v2) specifically to mitigate pinning attacks and enable reliable
fee-bumping. Each commitment transaction has two small **anchor
outputs** (one per peer) that any party can spend with their key, but
only via CPFP after a 1-block CSV. This gives each peer a guaranteed
way to attach a child transaction bumping the commitment's fee, even
if the commitment itself is pinned by an opponent.

## Walkthrough / mechanics

### Anchor output script

Each peer's anchor output uses:

```
<peer_pk> CHECKSIG
IFDUP NOTIF
   16 OP_CHECKSEQUENCEVERIFY
ENDIF
```

Equivalent: spendable by peer's key OR (after 16 blocks) by anyone.
Value: 330 sat (dust threshold).

The 16-block clause ensures the anchor can be claimed by anyone after
2.5 hours, preventing dust loss on stale commitments.

### CPFP via anchor

When a peer broadcasts the commitment, they (or any helper) can attach
a child:

```
Input: anchor output (peer's signature).
Output: peer's wallet, value = 330 - cpfp_fee.
```

This child transaction's fee bumps the parent's effective feerate per
package-relay rules.

### Why this mitigates pinning

Without anchors, attacker could pin via descendants on the
opponent's side. With anchors:

- Each peer has independent fee-bump source.
- Attacker pinning Carol's outputs doesn't affect Bob's anchor.
- Bob can attach a high-fee CPFP through his anchor regardless of
  what attacker does on chain.

### Limitations and replacement-cycling

Anchors enable fee bumping but don't prevent replacement cycling
(October 2023 attack). The cycling attack works because:

1. Bob attaches CPFP child to his anchor.
2. Carol replaces the commitment_carol -> conflict creates eviction
   of Bob's anchor + child.
3. Bob retries; Carol cycles again.

So anchors solve PINNING (a static attack) but not CYCLING (a dynamic
attack). TRUC v3 / BIP-431 was the next iteration to solve cycling.

### Practical fee-bump strategy

Honest node detects commitment in mempool, immediately:

1. Computes target feerate (e.g. 100 sat/vB current).
2. Commitment fee + child fee = target feerate * package size.
3. Constructs anchor-spending child with appropriate fee.
4. Broadcasts package (commitment + child) via package-relay.

Watchtowers running BOLT-03 v2 do this on operator's behalf.

### Dust limit concerns

Anchors at 330 sat are above standard dust threshold for P2WSH outputs.
They can be safely created without violating policy. Some implementations
use Taproot-based anchors (lower min) for further savings.

### Witnessed by both peers

Both peers' anchors exist on every commitment. Peer A's anchor is
signed by A's key; B's by B's. Commitment also lets either peer or any
third party (after 16 CSV) sweep the anchor, preventing dust accumulation
in mempool over time.

## Worked example

Bob has 100,000 sat outbound to Carol via channel. Carol force-closes.

```
commitment_carol (v2 with anchors):
  Output 0: Bob's main balance: 50,000 sat
  Output 1: Carol's main balance: 50,000 sat (minus fee)
  Output 2: HTLC output: 100,000 sat (pending)
  Output 3: Bob's anchor: 330 sat (Bob's key + 16 CSV anyone)
  Output 4: Carol's anchor: 330 sat (Carol's key + 16 CSV anyone)
  Total fee: 600 sat (low, for static commitment fee)

Bob detects commitment_carol. Mempool feerate is 100 sat/vB.

Bob constructs child_bob:
  Input: Output 3 (Bob's anchor)
  Output: 330 - cpfp_fee -> Bob's wallet
  cpfp_fee = (commitment_size + child_size) * 100 - existing_fee
           = (250 + 110) * 100 - 600 = 35,400 sat
  But Bob's anchor only has 330 sat; he must also include external
  funding input or accept low effective bump.

Real-world: Bob includes a fresh wallet UTXO as a second input,
contributing the bulk of the fee:

  Inputs: anchor 330 + Bob_wallet_UTXO 50,000
  Outputs: Bob_change 14,600 (after 35,400 fee)

Package broadcast: commitment_carol + child_bob (total fee 36,000 sat,
package feerate 100 sat/vB).
```

## Common pitfalls

- **Wallet UTXO availability**: anchor CPFP requires Bob have a UTXO
  for fee. If Bob has no on-chain wallet, anchor is useless.
- **Carol's symmetric ability**: Carol's anchor is also CPFP-able,
  giving her equal bumping power. Doesn't help attacker but doesn't
  hurt.
- **CSV 16 anyone-can-spend**: after 16 blocks, anyone can sweep the
  anchor. Stale commitments don't dust the chain.
- **Commitment fee static**: legacy channels use a fixed fee schedule;
  anchor channels effectively have flexible feerate via CPFP.
- **Mempool acceptance**: package relay must be enabled on all
  intermediate nodes for packages to propagate; older nodes may reject.

## References

- BOLT-03 anchor output spec.
- BIP-329 — Package relay.
- "Anchor outputs proposal" — Conner Fromknecht, 2019 lightning-dev.
- BIP-431 (TRUC) — fixes cycling that anchors alone don't.
