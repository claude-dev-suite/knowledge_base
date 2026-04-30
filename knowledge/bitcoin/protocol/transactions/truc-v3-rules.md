# TRUC (v3) Transactions - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/transactions`.
> Canonical source: https://github.com/bitcoin/bips/blob/master/bip-0431.mediawiki
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/transactions/SKILL.md

## Concept

TRUC ("Topologically Restricted Until Confirmation") is a mempool policy contract that opts a transaction in to a tighter topology in exchange for cleaner, pinning-resistant fee bumping. Activated as policy in Bitcoin Core 28.0, TRUC pairs with BIP331 (package relay) and ephemeral anchors to give Lightning and other contract protocols a reliable way to bump unconfirmed parents. Crucially, TRUC is **policy** only, not consensus.

## Walkthrough / mechanics

### Triggering TRUC

Transaction has `nVersion = 3`. Once accepted, the mempool enforces the rules below for its package. After confirmation, all rules drop away (it is a normal v3 tx in a block).

### The five rules

1. **Topological cap**: a v3 tx may have **at most one unconfirmed ancestor**, and **at most one unconfirmed descendant**. So clusters max out at parent + child.
2. **Mixed-version forbidden**: any unconfirmed ancestor or descendant of a v3 tx must also be v3. (And conversely a non-v3 cannot have a v3 unconfirmed parent.)
3. **Size cap on the parent**: the v3 parent must be `<= 10_000` virtual bytes.
4. **Size cap on the child**: the v3 child must be `<= 1_000` virtual bytes (small CPFP).
5. **Sibling eviction**: when a new v3 child enters and its parent already has another unconfirmed v3 child, the new child can replace the old one even without satisfying full BIP125 RBF (specifically, no need to pay for the old child's bandwidth) - provided it pays a higher feerate.

### Zero-fee parents

Because the child is mandatory to ship the parent (single descendant), a TRUC parent can pay **zero fee** as long as its child pays for the package. Mempool's package validation (`AcceptMultipleTransactions`) treats them as one feerate unit. This is what enables BOLT-3 commitment txs to be unsigned-fee anchors with no value carved off the channel.

### Ephemeral anchor outputs

A v3 parent can include an output of `value = 0` and `scriptPubKey = <anyonecanspend>` (often `OP_TRUE`). Mempool requires:

- Such anchors must be spent in the **same package** as the parent.
- Only one ephemeral anchor per tx.
- After confirmation, normal UTXO rules resume - if anyone can spend it, anyone might.

### Why the constraints

Pinning attack prevention: in pre-TRUC mempool, an attacker could attach a large, low-fee, hard-to-evict child to your unconfirmed funding tx, making it impossible to fee-bump (BIP125 rule 5: a replacement may evict at most 100 txs). Capping descendants to 1 means at most one child to evict, capping its size means low absolute-fee buy-in. Sibling eviction means the legitimate party can always swap in their own bumping child.

### State machine pseudocode

```
def accept_v3(tx, mempool):
    if tx.version != 3:
        return  # not our concern
    ancestors = mempool.get_unconfirmed_ancestors(tx)
    if any(a.version != 3 for a in ancestors):
        reject("v3 cannot have non-v3 unconfirmed ancestor")
    if len(ancestors) > 1:
        reject("v3 max-1-ancestor")
    if any(d.version != 3 for d in mempool.get_unconfirmed_descendants(tx)):
        reject("ancestor cannot have non-v3 unconfirmed descendant")
    if ancestors:
        # tx is the child
        if tx.vsize > 1000: reject("v3 child > 1kvB")
        siblings = [c for c in ancestors[0].children if c.txid != tx.txid]
        if siblings:
            sib = siblings[0]
            if tx.feerate <= sib.feerate: reject("sibling eviction needs higher feerate")
            mempool.remove(sib)
    else:
        # tx is the parent
        if tx.vsize > 10_000: reject("v3 parent > 10kvB")
    mempool.insert(tx)
```

## Worked example

Lightning commitment tx (post BOLT-3 + TRUC update):

```
parent: v3, 1 input from funding, 2 outputs (to_local, to_remote), 1 ephemeral anchor (0 sat OP_TRUE)
        feerate = 0 sat/vB, fee = 0
child:  v3, 1 input spending parent's ephemeral anchor, 1 output back to fee-payer
        size 200 vB, fee = 5000 sat -> child feerate 25 sat/vB

Package feerate (combined):
  vsize_parent = 700, vsize_child = 200 -> total = 900 vB
  total_fee   = 5000 sat
  package feerate = 5.55 sat/vB
```

If a sibling child was already in the mempool paying 3 sat/vB feerate, the new child evicts it via sibling eviction (rule 5) without having to pay for its bandwidth.

## Common bugs / anti-patterns

- Building a v3 child with `nVersion = 2`: ancestor mismatch, rejected.
- A wallet that accidentally creates a v3 tx (libraries with `nVersion = 3` in test config) hits the topological cap on a normal payment chain.
- Trying to RBF a v3 parent without using the child: rule 5 sibling eviction requires the new tx to be a *child*, not a replacement of the parent.
- Including 2 ephemeral anchors hoping for redundancy: only one allowed.
- Ephemeral anchor present but no spending child in the same submitpackage call: parent-only is rejected.
- Counting on TRUC rules in a block validator: TRUC is policy. A miner can include a v3 tx with a giant non-v3 descendant if no policy enforces relay - the resulting block is consensus-valid.
- Assuming TRUC implies wtxid relay or signaling: TRUC has no header signaling; it is on by version.

## References

- BIP431 (TRUC): https://github.com/bitcoin/bips/blob/master/bip-0431.mediawiki
- BIP331 (package relay): https://github.com/bitcoin/bips/blob/master/bip-0331.mediawiki
- Bitcoin Core 28.0 release notes: https://github.com/bitcoin/bitcoin/blob/v28.x/doc/release-notes/release-notes-28.0.md
- Original design: https://delvingbitcoin.org/t/v3-transaction-policy-for-anti-pinning/340
