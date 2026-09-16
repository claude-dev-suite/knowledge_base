# Mempool Policy vs Consensus Rules - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/consensus`.
> Canonical source: https://github.com/bitcoin/bitcoin/blob/master/src/policy/policy.cpp
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/consensus/SKILL.md

## Concept

Bitcoin Core has two distinct rule sets: **consensus** (must hold for any block / tx accepted into the chain) and **policy** / **standardness** (node-local defaults for which transactions a node will relay or include in its own template). A non-standard transaction may be perfectly valid in a block but rejected by every default node's mempool. Understanding the boundary is essential for L2 designers, miner-direct submitters, and anyone chasing "my tx is valid but won't relay" bugs.

## Walkthrough / mechanics

### Consensus weight limits

| Limit | Value | Source |
|-------|-------|--------|
| Block weight | 4,000,000 WU | BIP141 |
| Block sigops | 80,000 | consensus |
| Tx weight | 4,000,000 WU (i.e. fits in a block) | implicit |
| Coinbase scriptSig length | [2, 100] | consensus |
| Witness stack item size | 520 bytes, segwit v0 and Tapscript alike (BIP342 lifted the 10,000-byte *script* size limit, not this one) | BIP141 / BIP342 |

### Standardness limits (mempool default)

| Limit | Value | Source |
|-------|-------|--------|
| Max standard tx weight | 400,000 WU | `MAX_STANDARD_TX_WEIGHT` |
| Max standard tx sigops (witness-scaled cost) | 16,000 | `MAX_STANDARD_TX_SIGOPS_COST` = `MAX_BLOCK_SIGOPS_COST`/5 |
| Max potentially executed legacy sigops per tx | 2500 (Core 30.0, October 2025; #32521) | `MAX_TX_LEGACY_SIGOPS` |
| Max standard scriptSig push | 1650 bytes | `MAX_STANDARD_SCRIPTSIG_SIZE` |
| Max P2SH redeem script | 520 bytes | implied by push limit |
| OP_RETURN data | 100,000 vB aggregate, multiple outputs allowed (Core 30.0, October 2025; was 80 bytes / one per tx) | `-datacarriersize` |
| Dust threshold | `(output_size + spend_input_size) * dustrelayfee`, spend input 148 B non-witness / 67 B witness - 546 / 294 sat at the 3000 sat/kvB default | `GetDustThreshold` |
| Min relay feerate | 0.1 sat/vB (Core 30.0, October 2025; was 1 sat/vB) | `-minrelaytxfee` |

The 2500 legacy-sigop cap counts signature operations in all prevout scriptPubKeys, all input scriptSigs and any P2SH redeem scripts, using BIP16 accounting; witness and tapscript sigops keep their own pre-existing limits. It is **policy only** - a miner can still include a transaction that exceeds it and the block is consensus-valid. Core 30.0 shipped it "to prepare for a possible BIP54 deployment in the future" (release notes, #32521); see [consensus-cleanup-bip54-deep.md](../proposals/consensus-cleanup-bip54-deep.md).

### Standard scriptPubKey types (`IsStandardTx`)

- `pubkey`, `pubkeyhash`, `scripthash`, `multisig` (with N <= 3, M <= 3)
- `witness_v0_keyhash`, `witness_v0_scripthash`
- `witness_v1_taproot`
- `nulldata` (OP_RETURN ...)
- `witness_unknown` (relay-only since v22, opt-in)

Anything else: rejected at relay. Notably, bare CHECKSIG with a pubkey that isn't compressed used to be non-standard.

### Package / cluster limits

As of Bitcoin Core 31.0 (April 2026) the mempool is a **cluster mempool** and no longer enforces ancestor or descendant size/count limits. Two cluster limits apply instead. A *cluster* is the set of mempool transactions connected to each other by any combination of parent/child relationships.

```
DEFAULT_CLUSTER_LIMIT          = 64 transactions   (-limitclustercount)
DEFAULT_CLUSTER_SIZE_LIMIT_KVB = 101 kvB           (-limitclustersize)
```

Transactions whose acceptance would push a cluster over either limit are rejected by `MempoolAcceptResult`. TRUC (v3) tightens this to `<= 1` ancestor and `<= 1` descendant when both are unconfirmed.

Pre-31.0 (Core 30.x and earlier) the limits were per-ancestor-chain and per-descendant-chain: `DEFAULT_ANCESTOR_LIMIT = 25`, `DEFAULT_ANCESTOR_SIZE_LIMIT = 101 kvB`, `DEFAULT_DESCENDANT_LIMIT = 25`, `DEFAULT_DESCENDANT_SIZE_LIMIT = 101 kvB`. In 31.0 the two *size* constants were deleted and `-limitancestorsize` / `-limitdescendantsize` became hidden no-ops that emit an init warning; the two *count* constants still exist in `policy.h` but are deprecated and used only by the wallet for coin selection.

Also removed in 31.0: the **CPFP carveout**, which had allowed one extra child (no larger than 10 kvB, with exactly one ancestor) past a package's descendant limit. Nothing bypasses the cluster count limit now - the replacement for carveout-style use cases is TRUC plus sibling eviction.

### Replace-by-fee

Core's RBF policy has drifted away from BIP125; `doc/policy/mempool-replacements.md` is the live specification. As of Bitcoin Core 31.1 (July 2026) a replacement transaction must:

1. Directly conflict with one or more in-mempool transactions (spend a shared input). No replaceability signal is required - full-RBF has been the default since v28.0 (October 2024), and BIP125's rule 1 (signalling) is marked "(Removed)".
2. Pay an absolute fee at least the sum paid by the original transactions (rule 3).
3. Pay for its own bandwidth at the node's incremental relay feerate: fee delta `>= incrementalrelayfee * vsize_of_replacement` (rule 4; `-incrementalrelayfee` default 0.1 sat/vB).
4. Conflict with at most 100 distinct clusters (rule 5; before 31.0 this counted the directly conflicting txs plus their descendants).
5. Strictly improve the mempool's feerate diagram (rule 6, added in 31.0 with cluster mempool). For a *singleton* - a transaction alone in its cluster - a higher fee **and** a higher feerate is sufficient.

BIP125's rule 2 ("no new unconfirmed inputs") was also dropped in 31.0 and is now marked "(Removed)".

These are mempool-only; once mined, the rules are irrelevant.

### Recent additions

- **TRUC v3** (BIP431): policy-only contract for cleaner fee bumping.
- **Package relay**: `submitpackage` (Core 26.0, December 2023) admits a parent + child as a unit for dependent-CPFP locally; Core 28.0 (October 2024) added *opportunistic 1p1c* relay, which pairs a below-min-feerate parent with one child over the **existing** tx-relay protocol. BIP331 ("Ancestor Package Relay") is still Status: **Draft** as of September 2026 and its `sendpackages` / `pkgtxns` P2P messages have never shipped - see [package-relay/overview.md](../package-relay/overview.md).
- **Ephemeral anchors**: zero-value anchor outputs allowed iff spent in same package (policy carve-out, BIP431 companion).

## Worked example

Sending a 5-of-7 bare multisig to mainnet:

```
scriptPubKey = OP_5 <P1> <P2> <P3> <P4> <P5> <P6> <P7> OP_7 OP_CHECKMULTISIG
```

Consensus-valid (M and N up to 20). But standardness allows only `M, N <= 3` for bare multisig - the tx broadcasts but no default node relays it. Workarounds: wrap in P2SH (M, N up to 15) or P2WSH (M, N up to ~67, limited only by witness size 520-byte legacy or the Tapscript budget). To get the original mined, submit directly to a miner via private RPC (`mempool.space accelerator`, `viabtc transaction accelerator`).

Another: `OP_RETURN <100 bytes>` was rejected by every default-config node under the old 80-byte datacarrier limit. Since Core 30.0 (October 2025) the default `-datacarriersize` is 100,000 vB, applied to the aggregate scriptPubKey size across all OP_RETURN outputs, so it relays today; only nodes that lower `-datacarriersize` below the payload size (e.g. the documented `-datacarriersize=83` revert to pre-30.0 behaviour) or set `-datacarrier=0` still reject it.

## Common bugs / anti-patterns

- Building a transaction that is consensus-valid but fails policy, then debugging "why won't my node accept it" - run `testmempoolaccept` first.
- Assuming RBF is consensus: it is purely mempool. A miner can choose to mine the original even if the replacement pays more.
- Using `IsStandard()` checks on historical chain data: standardness has changed many times; old txs may be non-standard today but were valid when mined.
- Building a v3 child larger than 1,000 vB; the parent will accept but the child is rejected. 10,000 vB is the size ceiling for a TRUC transaction generally (BIP431 rule 4) - the child of an unconfirmed TRUC is capped at 1,000 vB (rule 5, `TRUC_CHILD_MAX_VSIZE`).
- Treating dust threshold as static: it scales with `dustrelayfee` (default 3000 sat/kB) - users with a higher setting create unspendable dust at a lower output.
- Mistaking `policyAsserts` failures for consensus failures in tests; Core's functional tests use both `assert_equal` on rejection reason and `prioritisetransaction` to bypass policy.

## References

- BIP125 (RBF - historical; superseded in Core by `doc/policy/mempool-replacements.md`): https://github.com/bitcoin/bips/blob/master/bip-0125.mediawiki
- Cluster mempool terminology: https://github.com/bitcoin/bitcoin/blob/master/doc/policy/mempool-terminology.md
- BIP331 (ancestor package relay - Draft, never deployed as of September 2026): https://github.com/bitcoin/bips/blob/master/bip-0331.mediawiki
- BIP431 (TRUC): https://github.com/bitcoin/bips/blob/master/bip-0431.mediawiki
- Bitcoin Core: `src/policy/policy.cpp`, `src/policy/rbf.cpp`, `src/policy/packages.cpp`, `src/policy/truc_policy.h`
- Core 31.0 release notes (cluster mempool): https://bitcoincore.org/en/releases/31.0/
- Core 30.0 release notes (2500 legacy-sigop policy cap, #32521): https://bitcoincore.org/en/releases/30.0/
- Core 28.0 release notes (opportunistic 1p1c relay): https://bitcoincore.org/en/releases/28.0/
