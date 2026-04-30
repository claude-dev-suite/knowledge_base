# Lightning Anchor Channel Fee Bumping - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/wallets/rbf-cpfp`.
> Canonical source: BOLT-3 update for anchor outputs ; BIP431 (TRUC v3)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/wallets/rbf-cpfp/SKILL.md

## Concept

A Lightning channel commitment transaction is **pre-signed** at a fee
rate decided when the channel was opened or last updated. Fees can move
dramatically between then and a force-close event months later. Without
a way to bump the commitment after broadcast, parties are stuck choosing
between paying massive premiums up front or risking timeout failures.

Anchor channels (BOLT-3, 2020) added two tiny outputs to commitment
transactions — one per channel party — that can be spent immediately to
bump the commitment via CPFP. TRUC (v3) channels (BIP431, 2024) refine
this further with package-aware sibling eviction.

## Walkthrough / mechanics

### Anchor commitment structure

A modern channel commitment has these outputs:

```
- to_local       (your balance, encumbered with revocation + delay)
- to_remote      (counterparty balance, immediately spendable)
- offered/received HTLCs (one each)
- anchor_local   (330 sats, spendable by you immediately, OR by anyone after 16 blocks)
- anchor_remote  (330 sats, spendable by them immediately, OR by anyone after 16 blocks)
```

The 330-sat anchors are deliberately just above the dust limit. The
"anyone after 16 blocks" branch ensures the outputs can never become
unspendable dust — important because both anchors must remain spendable
forever for cleanup.

### Bumping flow

When a node decides to broadcast a commitment with insufficient fee:

```
1. Build commitment tx (pre-signed) — fee = e.g., 1 sat/vB.
2. Build child tx:
   - Input 0: anchor_local (330 sats, you have witness)
   - Input 1..n: confirmed UTXOs from your on-chain wallet
   - Output 0: change to your wallet
3. Set child fee so package rate clears current mempool floor.
4. submitpackage([commitment, child]).
```

Effective package feerate:

```
rate_package = (fee_commitment + fee_child) / (vsize_commitment + vsize_child)
```

To target 35 sat/vB on a 720 vbyte commitment + 200 vbyte child:

```
required_total = 920 * 35 = 32_200 sats
fee_child = 32_200 - fee_commitment = 32_200 - 720 = 31_480 sats
```

### Pinning resistance via TRUC v3

The vanilla anchor design is vulnerable to pinning: a malicious
counterparty can broadcast their copy of the commitment with their own
high-feerate-low-amount child spending the *other* anchor (rule 5 of
BIP125: max 100 replaced) blocking honest fee bumps.

BIP431 ("v3 transactions") fixes this with:

- **Tx version = 3** is a magic value enforced by relay policy.
- **Topology limits**: a v3 tx may have at most 1 unconfirmed ancestor
  and at most 1 unconfirmed descendant.
- **Sibling eviction**: a v3 child can replace a v3 sibling without
  paying for the sibling's bandwidth.
- **Ephemeral anchor**: a 0-value output that *must* be spent in the
  same package. With ephemeral anchors, the channel commitment doesn't
  need to budget 330 sats per side.

The result: an honest fee bumper can always replace an attacker's pinning
child by paying just the sibling-eviction surcharge.

## Worked example: LND `bumpforceclose`

```bash
# After a force close, list pending channels
lncli pendingchannels

# LND already published the commitment but it's stuck.
# Trigger fee bump:
lncli wallet bumpfee <commitment_outpoint:vout> --conf_target 6
# OR
lncli wallet bumpforceclose <chan_point>
```

Internally LND's `bumpforceclose` builds the anchor-spending child and
submits a package. The new fee target is taken from the on-chain
estimator. LDK and CLN have analogous APIs (`force_close_with_target_feerate`,
`feebumper.bump`).

For LDK in raw code:

```rust
let bump = chain_monitor.bump_anchor_for_channel(
    &chan_id,
    target_feerate_sat_per_kw,
);
match bump {
    Ok(()) => log::info!("Anchor CPFP submitted"),
    Err(e) => log::warn!("Bump failed: {:?}", e),
}
```

## Watching descendant chains

Each force-close can spawn a *long* chain of resolution txs:

```
commitment ──> HTLC-success
            ─> HTLC-timeout
            ─> to_local (after CSV delay)
```

Each may need its own fee bump. The package-relay model lets you bump the
top of the chain; lower descendants may need RBF or their own CPFP child.
LDK's `OnchainTxHandler` orchestrates this state machine; CLN's
`onchaind` does similar.

## Common pitfalls

- **Pre-anchor channels** (legacy `option_static_remote_key` only) cannot
  CPFP — the commitment has no spendable anchor. Bumping requires RBF on
  the commitment itself, but the commitment is pre-signed. You're stuck.
  Migrate to anchor channels.
- **Insufficient on-chain wallet balance**: if your hot wallet has no
  spare UTXOs, you cannot pay a child's fee. Always reserve a UTXO sized
  for worst-case force-close scenarios.
- **Anchor reservation**: opening an anchor channel locks 330 sats per
  side that may sit unspent for years. Multiply by # of channels.
- **Forgetting the 16-block "anyone can spend" branch**: when cleaning
  up old anchors much later, a wallet helper may try the 1-sig path and
  fail because the remote key has been wiped from memory. Use the
  16-block fallback.
- **Pre-BIP331 nodes**: if your bitcoind is < v25, `submitpackage` is
  unavailable and you fall back to broadcasting child first (orphan)
  hoping the parent shows up. Modern LN daemons require v25+.
- **TRUC interaction with non-v3 mempool**: a v3 commitment can only be
  bumped with a v3 child. Misconfigured wallets that produce v2 children
  will see "v3-tx-nonstandard" rejection.

## References

- BOLT-3 anchor outputs: https://github.com/lightning/bolts/blob/master/03-transactions.md
- BIP431 (TRUC v3): https://github.com/bitcoin/bips/blob/master/bip-0431.mediawiki
- Pinning analysis (rgrant 2020): https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-April/002639.html
- See also: [package-cpfp-bip331.md](package-cpfp-bip331.md), [bip125-rules-walkthrough.md](bip125-rules-walkthrough.md)
