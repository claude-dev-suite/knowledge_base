# submitpackage RPC Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/package-relay`.
> Canonical source: Bitcoin Core RPC docs and `doc/policy/packages.md`
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/package-relay/SKILL.md

## Concept

`submitpackage` is the user-facing entry point to Bitcoin Core's
package validation: a list of related raw transactions evaluated as
an atomic mempool admission unit. It was added in Bitcoin Core 26.0
(December 2023) and extended in 28.0 (`maxfeerate`, `maxburnamount`)
and 30.0. It is a **local** submission path only - there is no BIP331
package relay on the network, so a package that `submitpackage` accepts
still propagates only if it fits Core's opportunistic 1-parent-1-child
relay. The skill mentions the RPC at a high level; this article walks
the validation pipeline, the result format, the package shapes Core
accepts, and the failure-mode reporting.

## Walkthrough / mechanics

**RPC signature (Bitcoin Core 28.0+, unchanged through 31.1, July 2026):**

```
submitpackage [
  "rawtx_hex_1", "rawtx_hex_2", ..., "rawtx_hex_n"
] (maxfeerate, maxburnamount)
```

`maxfeerate` defaults to 0.10 BTC/kvB - rejected if package fee
rate exceeds this. `maxburnamount` defaults to 0 - rejected if any
output value exceeds this AND has unspendable script (OP_RETURN).

**Result format:**

```json
{
  "package_msg": "success" | "package-unknown-error" | ...,
  "tx-results": {
    "<wtxid>": {
      "txid": "...",
      "other-wtxid": "...",
      "vsize": ...,
      "fees": {
        "base": ...,
        "effective-feerate": ...,
        "effective-includes": ["<wtxid>", ...]
      }
    },
    ...
  },
  "replaced-transactions": ["<txid>", ...]
}
```

`effective-includes` lists which other package txs contributed to
this tx's effective fee rate (CPFP).

**Validation pipeline (simplified):**

```
1. Parse all raw txs; compute txid and wtxid for each.
2. Check none are already in mempool or block.
3. Topological sort by parent-child relationships within the package.
4. Pre-checks per tx:
   - no double-spend with mempool except via RBF
   - basic script validity (signatures, etc.)
   - no negative output amounts
5. Compute package effective fee rate:
   total_fee / total_vsize
6. Compare to mempool min fee rate.
7. Run script validation per tx.
8. Check the resulting mempool cluster against the cluster limits
   (64 transactions, 101 kvB) - Core 31.0, April 2026. Core 30.x and
   earlier applied per-tx ancestor/descendant limits here instead.
9. If all pass, atomically add all txs to mempool.
   Emit "replaced-transactions" if RBF replaced any.
10. Announce each tx via ordinary P2P transaction relay. A low-feerate
    parent only reaches peers if it is opportunistically paired with its
    child as a 1p1c package - there are no BIP331 package messages.
```

**Package shapes accepted:**

Packages submitted to the mempool must be **child-with-parents**:
exactly one child plus some (not necessarily all) of its unconfirmed
parents, and nothing else; none of the parents may depend on each other
(`IsChildWithParentsTree` in `src/rpc/mempool.cpp`, otherwise "package
topology disallowed"). Count is capped at 25 transactions and total
weight at 404,000 WU (`doc/policy/packages.md`, as of Bitcoin Core 31.1,
July 2026).

| Shape | Notes |
|-------|-------|
| 1-parent + 1-child (standard CPFP) | Most common; the only shape that also relays over P2P |
| 1-child + N parents (child spends several unconfirmed parents) | Allowed; since 30.0 not every unconfirmed parent need be listed |
| 1-parent + 2 children spending different parent outputs | Rejected - not child-with-parents |
| Standalone parent (no child in package) | Allowed - it is just a 1-tx package; `sendrawtransaction` is simpler |
| Standalone child (parents already in mempool/chain) | Allowed - a package of one |
| TRUC v3 packages | Strict 1-parent 1-child |
| Multi-level chains (grandparent -> parent -> child) | Rejected - no parent in the package may spend another parent in the package |

## Worked example

**Scenario:** Lightning commitment + anchor CPFP.

```
parent (commitment_tx):
  inputs:  channel_funding_outpoint (2-of-2 multisig, both sigs)
  outputs:
    out0: 100_000 sat to local_balance script
    out1: 100_000 sat to remote_balance script
    out2: 240 sat to P2A anchor (OP_1 <0x4e73>, spendable by anyone)

child (anchor_cpfp):
  inputs:
    in0: parent.out2 (the 240-sat P2A anchor, empty witness)
    in1: my_other_utxo (50_000 sat)
  outputs:
    out0: 49_000 sat to my_change_addr   # 1240 sat fee
```

```bash
parent_hex=$(bitcoin-cli createrawtransaction ...)
child_hex=$(bitcoin-cli createrawtransaction ...)

bitcoin-cli submitpackage "[\"$parent_hex\",\"$child_hex\"]"
```

**Successful response:**

```json
{
  "package_msg": "success",
  "tx-results": {
    "<wtxid_parent>": {
      "txid": "<txid_parent>",
      "vsize": 200,
      "fees": {
        "base": 0.00000200,                           // parent has only 200 sat fee
        "effective-feerate": 0.00003600,              // 3.6 sat/vB after CPFP
        "effective-includes": ["<wtxid_parent>", "<wtxid_child>"]
      }
    },
    "<wtxid_child>": {
      "txid": "<txid_child>",
      "vsize": 200,
      "fees": {
        "base": 0.00001240,
        "effective-feerate": 0.00003600,
        "effective-includes": ["<wtxid_parent>", "<wtxid_child>"]
      }
    }
  }
}
```

Both txs share the same effective fee rate because they were
evaluated as a package.

**Failure case: parent below mempool min, child not paying enough.**

```json
{
  "package_msg": "package-unknown-error",
  "tx-results": {
    "<wtxid_parent>": {
      "error": "min relay fee not met, 1 < 1000"
    },
    "<wtxid_child>": {
      "error": "package-mempool-limits"
    }
  }
}
```

When this happens, increase child fee, rebuild PSBT, resubmit.

**RBF replacement via package:**

```bash
# Originally submitted package had child fee = 5000 sat.
# We want to bump to 10000 sat and replace.
new_child_hex=...  # same parent, same outputs, different fee/inputs
bitcoin-cli submitpackage "[\"$parent_hex\",\"$new_child_hex\"]"
```

Package RBF is deliberately limited: the package must be
1-parent-1-child with no in-mempool ancestors, the parent feerate must
be below the package feerate, total fees must cover the replacement at
the incremental relay feerate, and - since Core 31.0 (April 2026) - the
replacement must strictly improve the mempool's feerate diagram
(`doc/policy/packages.md`, as of Bitcoin Core 31.1, July 2026). Core
28.0 through 30.x used the BIP125 rule set and additionally required
every conflicting cluster to be of size <= 2. Old child appears in
`replaced-transactions`.

## Common bugs / pitfalls

1. **Parent already in mempool, child not.** Submit just the child
   via `sendrawtransaction`. Package txs that are already in the
   mempool by txid are deduplicated out before submission and excluded
   from the package feerate.
2. **Wrong order in array.** Must be topologically sorted parent
   before child. Reversed order fails with "package-not-sorted".
3. **Missing inputs documented in package.** Every input of every
   package tx must spend either:
   - A confirmed UTXO, OR
   - An output of an earlier tx in the package, OR
   - An output of a tx already in mempool.
   Reference to a non-existent UTXO causes "missing inputs".
4. **maxfeerate too low.** A high-fee CPFP child can exceed the
   default 0.10 BTC/kvB limit. Increase via second arg.
5. **TRUC violations within package.** A v0 parent + v3 child is
   rejected because v3 requires v3 ancestors.
6. **Excess mempool topology.** Since the cluster mempool in Core 31.0
   (April 2026) there are no ancestor/descendant limits; instead the
   resulting cluster must stay within 64 transactions and 101 kvB.
   Core 30.x and earlier enforced 25 unconfirmed ancestors/descendants
   and 101 kvB instead.
7. **Fee accounting confusion.** `fees.base` is the tx's own fee.
   `effective-feerate` is the package fee rate. UI tools that show
   only base fee mislead users about true confirmation incentive.

## References

- BIP331 (Status: Draft as of September 2026, never deployed): https://github.com/bitcoin/bips/blob/master/bip-0331.mediawiki
- Bitcoin Core RPC docs: https://github.com/bitcoin/bitcoin/blob/master/src/rpc/mempool.cpp
- Package mempool accept policy: https://github.com/bitcoin/bitcoin/blob/master/doc/policy/packages.md
- Mempool design and cluster limits: https://github.com/bitcoin/bitcoin/blob/master/doc/policy/mempool-design.md
- LDK package usage: https://github.com/lightningdevkit/rust-lightning
