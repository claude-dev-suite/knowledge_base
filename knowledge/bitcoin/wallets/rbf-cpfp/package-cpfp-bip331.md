# Package CPFP, `submitpackage` and BIP331 - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/wallets/rbf-cpfp`.
> Canonical sources: Core `doc/policy/packages.md` https://github.com/bitcoin/bitcoin/blob/v31.1/doc/policy/packages.md ; BIP331 https://github.com/bitcoin/bips/blob/master/bip-0331.mediawiki
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/wallets/rbf-cpfp/SKILL.md

## Concept

Before package validation, mempool admission was per-transaction. A child
transaction paying high fees couldn't rescue a parent below the mempool
floor: the parent was rejected, and the child became
orphan-of-no-known-parent.

*Package* admission fixes that. A package is a parent + child (or several
parents + one child) submitted together and evaluated as one mining unit
by aggregate feerate. If the package's effective rate clears the floor,
both go in even if the parent alone wouldn't.

**Two different things carry the name "package relay" and only one of
them exists in Bitcoin Core.** Keep them apart:

| Mechanism | Status as of September 2026 |
|---|---|
| Core's `submitpackage` RPC | Shipped in Core **26.0 (December 2023)**. Local submission only: it admits a package to *your* mempool. It is Core's own interface, not a BIP. |
| Core's opportunistic 1-parent-1-child (1p1c) relay | Shipped in Core **28.0 (October 2024)** (#28970). Carried over the *existing* transaction relay protocol, not new P2P messages. |
| BIP331 "Ancestor Package Relay" | Still **Status: Draft**. Its `sendpackages` / `ancpkginfo` / `getpkgtxns` / `pkgtxns` P2P messages do not exist in Core 31.1 (July 2026) — `src/protocol.h` defines no such message types. |

The rest of this article describes what Core actually implements; the
BIP331 wire protocol is covered at the end as an unshipped design.

## Walkthrough / mechanics

A package submitted through `submitpackage` must be *child-with-parents*:
exactly one child plus some of its unconfirmed parents, nothing else
(`doc/policy/packages.md`, as of Core 31.1, July 2026). The narrower
1-parent-1-child shape is what the P2P side and package RBF are limited
to. The validation steps:

1. **Topological check** — packages must be a DAG with the child as the
   sink. No cycles, no siblings.
2. **Per-tx pre-check** — each tx must individually be valid scripts and
   format; only fee/feerate failures are deferred.
3. **Package feerate** — `total_fees / total_vsize` is computed. If the
   package's modified feerate clears `mempoolminfee`, package admission
   proceeds.
4. **Per-tx feerate** — each tx's individual feerate must normally clear
   the node's `-minrelaytxfee` floor, whose default dropped from
   1 sat/vB to 0.1 sat/vB in Core 30.0 (October 2025, PR #33106). The
   exception is the 1-parent-1-child case: the parent may be below
   `-minrelaytxfee`, even 0 fee — allowed for TRUC parents since 28.0,
   and extended to non-TRUC packages in 31.0 (April 2026, #33892).
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

## P2P: what actually relays

Core has no package-specific P2P messages. What it has, since 28.0
(October 2024), is *opportunistic 1p1c relay*: a transaction whose
feerate is too low to be accepted on its own is held and opportunistically
paired with its child, and the two are submitted as a package — downloaded
over the ordinary transaction relay protocol rather than a package-specific
one (#28970). Core 28.0's own release notes call this "limited" and "not
yet reliable under adversarial conditions", and note that a child with
multiple unconfirmed parents is not supported on the P2P path.

Two later releases widened it:

- **30.0 (October 2025)** — the 1p1c child may already have unconfirmed
  parents in the mempool, so 1p1c packages attached to broader topologies
  (multi-parent-1-child, grandparent-parent-child) also propagate
  (#31385). `submitpackage` likewise stopped requiring every unconfirmed
  parent to be present.
- **31.0 (April 2026)** — the 1p1c parent may pay below `-minrelaytxfee`,
  even 0 fee, for non-TRUC packages too (#33892).

Because this rides the existing relay protocol, there is no capability
handshake to negotiate and no separate deployment to track: a peer either
runs 28.0+ and does the pairing, or it does not and simply drops the
low-feerate parent as it always did. Treat propagation as best-effort,
and keep the direct `submitpackage` path (Lightning daemons, watchtowers,
rebroadcasters) as the reliable one.

### BIP331, the design that has not shipped

BIP331 specifies a real negotiated package-relay protocol — `sendpackages`
to advertise the capability during the version handshake, `ancpkginfo` to
describe a transaction's unconfirmed ancestors, and `getpkgtxns` /
`pkgtxns` to fetch them as an all-or-nothing batch:

```
A <-> B: sendpackages(versions)          # negotiate at handshake
B  ->  A: getdata(MSG_ANCPKGINFO, wtxid) # ask for the ancestor package
A  ->  B: ancpkginfo(wtxids...)          # list the ancestor package
B  ->  A: getpkgtxns(wtxids...)          # request all of them or none
A  ->  B: pkgtxns(txns...)               # deliver them together
```

None of these message types exist in Core 31.1; the BIP has been Status:
Draft since it was assigned in 2022 and remains so as of September 2026.
Do not design a wallet around it.

## Common pitfalls

- Submitting txs out of topological order → `package_msg: "package-not-sorted"`.
  Parent first, then child.
- Submitting a package where the child is below the node's
  `-minrelaytxfee` floor (0.1 sat/vB by default since Core 30.0,
  October 2025) while expecting the parent's high feerate to lift it
  → rejected. The per-tx-min check is a *floor*, not a ratio, and the
  below-floor exception applies to the 1p1c *parent*, not the child.
- Treating package admission as automatic broadcast: `submitpackage` is
  admission to *your* mempool. Propagation beyond you depends on peers
  running Core 28.0+ (October 2024) and on the package fitting the
  1-parent-1-child shape that opportunistic relay handles. Anything
  wider — multi-parent packages — stays local.
- Sibling-eviction confusion: a package is parents plus one child, never
  siblings. If you need to evict a sibling, that's TRUC (BIP431)
  territory.
- Forgetting that `submitpackage` is **wallet-agnostic**; the txs do not
  have to come from your wallet. This makes it useful for relayers.
- Anchor-output Lightning channels rely on `submitpackage` working
  end-to-end. The RPC landed in Core **26.0 (December 2023)**; on an
  earlier backend it does not exist at all and CPFP fee bumps fall back
  to broadcasting the child on its own. For the child to pull in a
  below-floor parent over P2P you additionally want peers on 28.0+.

## References

- BIP331 (Status: Draft): https://github.com/bitcoin/bips/blob/master/bip-0331.mediawiki
- Core package policy: https://github.com/bitcoin/bitcoin/blob/v31.1/doc/policy/packages.md
- Bitcoin Core 26.0 release notes (`submitpackage` RPC added): https://github.com/bitcoin/bitcoin/blob/v26.0/doc/release-notes.md
- Bitcoin Core 28.0 release notes (opportunistic 1p1c relay, #28970): https://github.com/bitcoin/bitcoin/blob/v28.0/doc/release-notes.md
- Bitcoin Core 31.0 release notes (non-TRUC 1p1c parent below minrelaytxfee, #33892): https://github.com/bitcoin/bitcoin/blob/v31.0/doc/release-notes.md
- Bitcoin Core PR 27932 (multi-parent package support): https://github.com/bitcoin/bitcoin/pull/27932
- See also: [bip125-rules-walkthrough.md](bip125-rules-walkthrough.md), [lightning-anchor-bumping.md](lightning-anchor-bumping.md)
