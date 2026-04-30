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
| Witness item size | 520 bytes (legacy) / no per-item limit (Tapscript) | BIP342 changed |

### Standardness limits (mempool default)

| Limit | Value | Source |
|-------|-------|--------|
| Max standard tx weight | 400,000 WU | `MAX_STANDARD_TX_WEIGHT` |
| Max standard tx sigops | 8000 | `MAX_STANDARD_TX_SIGOPS_COST` |
| Max standard scriptSig push | 1650 bytes | `MAX_STANDARD_SCRIPTSIG_SIZE` |
| Max P2SH redeem script | 520 bytes | implied by push limit |
| OP_RETURN data | 80 bytes / one per tx | `nMaxDatacarrierBytes` |
| Dust threshold | `3 * (output_size + 148) * feerate` | `GetDustThreshold` |
| Min relay feerate | 1 sat/vB (default) | `-minrelaytxfee` |

### Standard scriptPubKey types (`IsStandardTx`)

- `pubkey`, `pubkeyhash`, `scripthash`, `multisig` (with N <= 3, M <= 3)
- `witness_v0_keyhash`, `witness_v0_scripthash`
- `witness_v1_taproot`
- `nulldata` (OP_RETURN ...)
- `witness_unknown` (relay-only since v22, opt-in)

Anything else: rejected at relay. Notably, bare CHECKSIG with a pubkey that isn't compressed used to be non-standard.

### Package / chain limits

```
DEFAULT_ANCESTOR_LIMIT       = 25
DEFAULT_ANCESTOR_SIZE_LIMIT  = 101 kvB
DEFAULT_DESCENDANT_LIMIT     = 25
DEFAULT_DESCENDANT_SIZE_LIMIT= 101 kvB
```

Transactions whose acceptance would exceed these chain sizes are rejected by `MempoolAcceptResult`. TRUC (v3) tightens this to `<= 1` ancestor and `<= 1` descendant when both are unconfirmed.

### Replace-by-fee (BIP125)

A replacement transaction must:

1. The original signaled replaceability (sequence < 0xFFFFFFFE) **or** descend from a TRUC parent / use full-RBF (post v24.0 default true).
2. Pay a feerate strictly higher than the original.
3. Pay an absolute fee at least `min_relay_fee * size_of_replacement` more than the sum of replaced txs' fees.
4. Not introduce any new unconfirmed inputs.
5. Not evict more than 100 txs in total.

These are mempool-only; once mined, the rules are irrelevant.

### Recent additions

- **TRUC v3** (BIP431): policy-only contract for cleaner fee bumping.
- **Package relay** (BIP331): `submitpackage` admits a parent + child as a unit for dependent-CPFP.
- **Ephemeral anchors**: zero-value anchor outputs allowed iff spent in same package (policy carve-out, BIP431 companion).

## Worked example

Sending a 5-of-7 bare multisig to mainnet:

```
scriptPubKey = OP_5 <P1> <P2> <P3> <P4> <P5> <P6> <P7> OP_7 OP_CHECKMULTISIG
```

Consensus-valid (M and N up to 20). But standardness allows only `M, N <= 3` for bare multisig - the tx broadcasts but no default node relays it. Workarounds: wrap in P2SH (M, N up to 15) or P2WSH (M, N up to ~67, limited only by witness size 520-byte legacy or the Tapscript budget). To get the original mined, submit directly to a miner via private RPC (`mempool.space accelerator`, `viabtc transaction accelerator`).

Another: `OP_RETURN <100 bytes>` is consensus-valid but exceeds `nMaxDatacarrierBytes=80`. Rejected by every default-config node.

## Common bugs / anti-patterns

- Building a transaction that is consensus-valid but fails policy, then debugging "why won't my node accept it" - run `testmempoolaccept` first.
- Assuming RBF is consensus: it is purely mempool. A miner can choose to mine the original even if the replacement pays more.
- Using `IsStandard()` checks on historical chain data: standardness has changed many times; old txs may be non-standard today but were valid when mined.
- Building a v3 child that exceeds the 10 kvB single-descendant limit; the parent will accept but the child is rejected.
- Treating dust threshold as static: it scales with `dustrelayfee` (default 3000 sat/kB) - users with a higher setting create unspendable dust at a lower output.
- Mistaking `policyAsserts` failures for consensus failures in tests; Core's functional tests use both `assert_equal` on rejection reason and `prioritisetransaction` to bypass policy.

## References

- BIP125 (RBF): https://github.com/bitcoin/bips/blob/master/bip-0125.mediawiki
- BIP331 (package relay): https://github.com/bitcoin/bips/blob/master/bip-0331.mediawiki
- BIP431 (TRUC): https://github.com/bitcoin/bips/blob/master/bip-0431.mediawiki
- Bitcoin Core: `src/policy/policy.cpp`, `src/policy/rbf.cpp`, `src/policy/packages.cpp`
