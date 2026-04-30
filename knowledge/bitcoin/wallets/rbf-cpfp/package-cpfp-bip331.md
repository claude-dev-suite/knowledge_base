# Package CPFP and BIP331 - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/wallets/rbf-cpfp`.
> Canonical sources: BIP331 https://github.com/bitcoin/bips/blob/master/bip-0331.mediawiki ; package-relay design notes
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/wallets/rbf-cpfp/SKILL.md

## Concept

Pre-BIP331, mempool admission was per-transaction. A child transaction
paying high fees couldn't rescue a parent below the mempool floor: the
parent was rejected, and the child became orphan-of-no-known-parent.

BIP331 ("Ancestor Package Relay") introduces atomic *package* admission.
A package is a parent + child (or several parents + one child) submitted
together, evaluated as one mining unit by aggregate feerate. If the
package's effective rate clears the floor, both go in even if the parent
alone wouldn't.

Core exposes this via the `submitpackage` RPC and a new P2P message
flow (`sendpackages`, `getpkgtxns`, `pkgtxns`).

## Walkthrough / mechanics

A package is currently restricted to 1 parent + 1 child (Core 26+) or
multi-parent + 1 child (Core 28+). The validation steps:

1. **Topological check** — packages must be a DAG with the child as the
   sink. No cycles, no siblings.
2. **Per-tx pre-check** — each tx must individually be valid scripts and
   format; only fee/feerate failures are deferred.
3. **Package feerate** — `total_fees / total_vsize` is computed. If the
   package's modified feerate clears `mempoolminfee`, package admission
   proceeds.
4. **Per-tx feerate** — each tx's individual feerate must clear at least
   1 sat/vB (the relay floor). This stops a 0-fee parent from ever
   relying entirely on the child.
5. **Mempool effects** — admission is atomic. If any later check fails,
   the entire package is rejected.

The RPC accepts an array of hex transactions in topological order:

```bash
bitcoin-cli submitpackage '["<parent_hex>", "<child_hex>"]'
```

Returns per-tx results:

```json
{
  "package_msg": "success",
  "tx-results": {
    "abc...parent_wtxid": {"txid":"...","vsize":141,"fees":{"base":0.00000050}},
    "def...child_wtxid":  {"txid":"...","vsize":141,"fees":{"base":0.00010000}}
  },
  "replaced-transactions": []
}
```

## Worked example

Lightning anchor channel scenario: commitment tx is pre-signed with a
fixed (low) fee. To get it confirmed during a fee spike, the wallet
spends one of the anchor outputs in a child tx with the desired fee.

```python
# Pseudo-code
parent_hex = signed_commitment_tx       # 200 vsize, fee = 200 sats (1 sat/vB)
                                        # mempool min currently 30 sat/vB

# Child spends anchor (1 input ~58 vB) + fee-bump UTXO (1 input ~68 vB)
# + change output. ~166 vsize. We want package rate >= 35 sat/vB.
# Total target package fee = (200 + 166) * 35 = 12_810 sats
# Parent contributes 200, child needs 12_610.
child = build_anchor_spend_with_fee(12_610)
child_hex = sign(child)

result = rpc.submitpackage([parent_hex, child_hex])
assert result["package_msg"] == "success"
```

If the wallet had broadcast `parent` alone, mempool would reject:
```
testmempoolaccept ... "min relay fee not met"
```

With package submission, both go in atomically.

## Package feerate vs ancestor feerate

There are two related concepts:

- **Package feerate**: the rate computed from the literal txs being
  submitted right now.
- **Ancestor feerate**: the rate any one of those txs would mine at,
  considering all unconfirmed ancestors already in mempool.

Bitcoin Core mining always sorts by ancestor-aware feerate. Submission
uses package feerate. They are equivalent when all txs in the package
are themselves ancestor-free; once you start chaining packages, the math
gets subtle and Core picks the better of the two.

## P2P relay (`sendpackages`)

The P2P side of BIP331 is currently behind a flag (`-blockreconstructionextratxn`
related infrastructure) and rolling out gradually. The wire flow:

```
Node A: sendpackages(version=0)        # advertise capability
Node A: pkgtxns(parents..., child)     # submit a package
Node B: getpkgtxns(pkg_hash)           # request via gossip if missing parts
```

For now, most package activity happens via direct `submitpackage` RPC
calls from local fee-bumpers — Lightning daemons, watchtowers, and so on.
Cross-node gossip of packages is operational on >50% of the network as of
late 2025 (Core 28 default).

## Common pitfalls

- Submitting txs out of topological order → `package_msg: "package-not-sorted"`.
  Parent first, then child.
- Submitting a package where the child has feerate < 1 sat/vB while
  expecting the parent's high feerate to lift it → rejected. The
  per-tx-min check is *floor*, not ratio.
- Treating package admission as automatic broadcast: it is admission to
  *your* mempool. If your peers don't speak BIP331 yet, the package
  may not propagate beyond you. As of 2025, this is mostly mitigated.
- Sibling-eviction confusion: BIP331 does not allow siblings — only
  parents and one child. If you need to evict a sibling, that's TRUC
  (BIP431) territory.
- Forgetting that `submitpackage` is **wallet-agnostic**; the txs do not
  have to come from your wallet. This makes it useful for relayers.
- Anchor-output Lightning channels rely on `submitpackage` working
  end-to-end. If your bitcoind is below v25 (which has only partial
  package support), CPFP fee bumps may silently fail.

## References

- BIP331: https://github.com/bitcoin/bips/blob/master/bip-0331.mediawiki
- Bitcoin Core PR 27932 (multi-parent package support): https://github.com/bitcoin/bitcoin/pull/27932
- See also: [bip125-rules-walkthrough.md](bip125-rules-walkthrough.md), [lightning-anchor-bumping.md](lightning-anchor-bumping.md)
