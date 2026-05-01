# submitpackage RPC Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/package-relay`.
> Canonical source: BIP331, Bitcoin Core RPC docs
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/package-relay/SKILL.md

## Concept

`submitpackage` is the user-facing entry point to Bitcoin Core's
package validation: a list of related raw transactions evaluated as
an atomic mempool admission unit. The skill mentions the RPC at a
high level; this article walks the validation pipeline, the result
format, the package types Core accepts (parent-child, parent-only,
child-only with confirmed parents), and the failure-mode reporting.

## Walkthrough / mechanics

**RPC signature (Bitcoin Core 28+):**

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
8. Apply ancestor/descendant policy limits to each tx.
9. If all pass, atomically add all txs to mempool.
   Emit "replaced-transactions" if RBF replaced any.
10. Relay each tx via P2P (BIP331 message types).
```

**Package shapes accepted:**

| Shape | Notes |
|-------|-------|
| 1-parent + 1-child (standard CPFP) | Most common |
| 1-parent + 2 children spending different parent outputs | Allowed |
| 1-child + 2 parents (child spends both parents) | Allowed |
| Standalone parent (no descendants in package) | Allowed but uncommon |
| Standalone child (parent already in mempool/chain) | Allowed |
| TRUC v3 packages | Strict 1-parent 1-child |
| Multi-level chains (grandparent -> parent -> child) | Limited; check ancestor count |

## Worked example

**Scenario:** Lightning commitment + anchor CPFP.

```
parent (commitment_tx):
  inputs:  channel_funding_outpoint (2-of-2 multisig, both sigs)
  outputs:
    out0: 100_000 sat to local_balance script
    out1: 100_000 sat to remote_balance script
    out2: 330 sat to ephemeral anchor (OP_TRUE-ish, spendable by either party)

child (anchor_cpfp):
  inputs:
    in0: parent.out2 (the 330-sat anchor)
    in1: my_other_utxo (50_000 sat)
  outputs:
    out0: 49_000 sat to my_change_addr   # 1330 sat fee
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
        "effective-feerate": 0.00006650,              // ~6.65 sat/vB after CPFP
        "effective-includes": ["<wtxid_parent>", "<wtxid_child>"]
      }
    },
    "<wtxid_child>": {
      "txid": "<txid_child>",
      "vsize": 200,
      "fees": {
        "base": 0.00001330,
        "effective-feerate": 0.00006650,
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

If the new package satisfies BIP125 rules at package level (total
fee >= old + incremental, etc.), it replaces. Old child appears in
`replaced-transactions`.

## Common bugs / pitfalls

1. **Parent already in mempool, child not.** Submit just the child
   via `sendrawtransaction`, not a package. Package format requires
   all txs to be NEW or all to share an ancestry within the package.
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
6. **Excess descendants.** Default policy: 25 unconfirmed descendants.
   A package that brings a parent's descendant count above this
   triggers eviction or rejection.
7. **Fee accounting confusion.** `fees.base` is the tx's own fee.
   `effective-feerate` is the package fee rate. UI tools that show
   only base fee mislead users about true confirmation incentive.

## References

- BIP331: https://github.com/bitcoin/bips/blob/master/bip-0331.mediawiki
- Bitcoin Core RPC docs: https://github.com/bitcoin/bitcoin/blob/master/src/rpc/mempool.cpp
- Mempool policy doc: https://github.com/bitcoin/bitcoin/blob/master/doc/policy/
- LDK package usage: https://github.com/lightningdevkit/rust-lightning
