# Strata Architecture Overview - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/strata`.
> Canonical source: https://docs.stratabtc.org/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/strata/SKILL.md

## Concept

Strata is a Bitcoin-anchored zk-rollup that combines an EVM execution
environment (built on Reth) with an SP1 zk-prover and a BitVM-based
two-way peg. Like Citrea, it uses Bitcoin for data availability via
Taproot envelope inscriptions and posts STARK validity proofs of L2
batches. Strata adds a "checkpoint" architecture: each anchor commits
multiple L2 epochs in one Bitcoin transaction.

## Walkthrough / mechanics

### Components

- **Strata Reth fork (alpen-reth)**: EVM execution client; tracks the
  L2 state.
- **Sequencer**: assembles L2 blocks, computes execution traces.
- **SP1 prover**: generates STARK proof of execution + state-transition
  correctness using Succinct Labs' SP1 zkVM (RISC-V proving).
- **DA layer**: Bitcoin via Taproot envelope inscriptions.
- **Bridge**: BitVM2-style fraud-proof bridge (in Strata called the
  "Alpen bridge"); operators front BTC and can be challenged.
- **Verifier nodes**: read inscriptions, run SP1 verifier off-chain.

### Epochs and checkpoints

L2 time is divided into **epochs** (e.g. 100 L2 blocks). At the end of
each epoch, the sequencer:

1. Commits the L2 state root to Bitcoin.
2. Posts an SP1 proof of correct execution for that epoch.
3. Posts compressed state diffs (or full DA blob if size permits).

A **checkpoint** is a Bitcoin transaction that:

1. Inscribes proof + state root + DA blob.
2. Optionally consumes a UTXO from the previous checkpoint (chained
   anchoring), so reorg detection is straightforward.

### Bridge: peg-in

1. User funds a peg-in deposit address derived from the operator-set
   MuSig2 key. The deposit script includes:
   - MuSig2 key-path spend (cooperative fast path).
   - Refund script-path: timelocked back to user if operators stall.
2. After confs, sequencer mints `wstBTC` on L2 via a state diff in the
   next checkpoint.

### Bridge: peg-out

1. L2 burn -> withdrawal event captured in checkpoint.
2. Operator fronts BTC to user.
3. Operator submits BitVM2 reimbursement claim spending peg UTXO.
4. Watcher network has fixed challenge window to disprove.
5. After window, operator's reimbursement Take tx confirms.

### Account abstraction

Strata embraces ERC-4337 native: every account is a smart contract
wallet. Standard `eth_sendTransaction` is replaced by `eth_sendUserOp`.
Pays for AA features on L2 with cheaper gas than L1 ETH.

### Performance targets

- ~100 ms L2 block time.
- ~5-10 TPS with current SP1 proving cadence (one checkpoint per
  Bitcoin block after maturity).
- Withdrawals: front-paid in seconds, operator reimbursement after
  ~1-2 weeks (BitVM challenge window).

## Worked example

Alice deposits 0.5 BTC on Strata.

```
1. Strata bridge UI gives address bc1p... derived from operator MuSig2.
2. Alice broadcasts BTC tx: 0.5 BTC -> bridge address.
3. After 100 confs, sequencer-included epoch K mints 0.5 wstBTC at
   Alice's L2 address.
4. Checkpoint inscribed at BTC block H, contains state root reflecting mint.
5. Alice uses 0.5 wstBTC in any EVM dApp on Strata.
```

Withdrawal:

```
1. Alice burns 0.5 wstBTC, emits withdrawal event.
2. Operator Op1 sends 0.499 BTC (after fee) to Alice on Bitcoin.
3. Op1 starts BitVM2 reimbursement workflow.
4. After ~14 days no challenge, Op1 takes 0.5 BTC from peg UTXO.
```

## Common pitfalls

- **Sequencer outage**: blocks paused; users with pending peg-out wait
  for next sequencer (force-inclusion path requires Bitcoin inscription
  by user).
- **Proof generation slowness**: SP1 STARKs on EVM execution take
  minutes-to-hours. Sequencer batches several epochs per checkpoint to
  amortise.
- **Operator capital lockup**: BitVM challenge window means operators
  hold significant capital; fees pricing must reflect time-value.
- **Inscription tail bloat**: DA blob sizes drop fast as compression
  improves; early checkpoints used full state diffs (~1 MB) — newer
  ones use Lagrange-style commitments + KZG (smaller but require
  more complex verifier).
- **Trust assumptions**: while the validity proof is trustless once
  finalised, operator front-payment is a counterparty trust within
  the challenge window if no operator picks up your withdrawal.

## References

- Strata documentation (docs.stratabtc.org).
- alpen-reth GitHub.
- SP1 zkVM (Succinct Labs).
- BitVM2 (Linus et al. 2024).
