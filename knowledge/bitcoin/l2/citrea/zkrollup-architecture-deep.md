# Citrea zk-Rollup Architecture - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/citrea`.
> Canonical source: https://docs.citrea.xyz/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/citrea/SKILL.md

## Concept

Citrea is a **zk-rollup** that uses Bitcoin as its data-availability and
settlement layer. It executes EVM-equivalent transactions off-chain,
generates a STARK proof of state-transition validity (RISC0 zkVM), and
posts the proof + state diffs to Bitcoin via Taproot inscriptions. Funds
move via a BitVM2-based two-way peg called **Clementine**.

## Walkthrough / mechanics

### Layer roles

- **Sequencer**: orders L2 transactions, builds blocks, posts data on
  Bitcoin.
- **Prover**: takes sequencer-published L2 blocks, generates a STARK
  proof using RISC0 circuits, publishes proof on Bitcoin.
- **Verifier**: anyone running `bitcoind`-aware logic that verifies the
  proof against published L2 state root.
- **Bridge operators (BitVM2)**: hold the peg UTXO; can be challenged on
  withdrawal validity and forced to publish a Citrea proof to settle a
  dispute.

### Data availability via inscriptions

Each rolled-up batch contains:

1. State diffs (compressed RLP-encoded touched accounts/storage).
2. STARK proof bytes (~few hundred KB depending on batch size).
3. Hashes of L2 block headers in the batch.

These are posted as **envelope inscriptions** under a magic tag (e.g.
`citrea`) using Taproot `OP_FALSE OP_IF "citrea" data OP_ENDIF`. Average
batch ≈ 200-400 KB on Bitcoin (chunk-witness costs).

### Proof verification on Bitcoin

Citrea's STARK is verified **off-chain** by light clients reading the
inscriptions; on-chain verification of the full STARK is currently too
expensive. The peg correctness is enforced by **BitVM2** (see
`bitvm2-bridge-walkthrough.md`): operators commit to claims, and
verifiers can challenge wrong claims with on-chain fraud proofs.

### EVM-equivalence

Citrea exposes a Geth-compatible JSON-RPC. Smart contracts compile with
unchanged Solidity. Precompiles match Ethereum (sha256, ecrecover, etc.)
plus Bitcoin-specific ones for SPV proofs.

### Sequencer batching

Sequencer batches ~1000 L2 transactions per Bitcoin block. Throughput:
~5-10 TPS sustained (limited by inscription bandwidth and proof
generation time, ~30 min per batch on a 64-core machine).

### State root anchoring

Each Bitcoin block carries (via inscription) the new Citrea state root.
A user wanting to verify their L2 balance scans Bitcoin for the latest
state root, then runs a Merkle-Patricia proof of their account against
the L2 client.

## Worked example

A user transfers 0.1 cBTC on Citrea L2:

```
L2 tx: from=0xAlice, to=0xBob, value=0.1 cBTC, gasPrice=1 gwei
```

Sequencer batches with 999 other txs into batch B_1234. Batch contains:
- Pre-state root: 0xa1...
- Post-state root: 0xb2...
- ~200 KB of compressed state diffs + STARK proof.

Batch is inscribed on Bitcoin tx `tx_seq` at block 850 100. Anyone
running a Citrea light client downloads the batch, runs RISC0 verifier
on the proof, accepts the new state root.

If a user wants to peg out, they trigger a Clementine BitVM2 disbursement
referencing this finalised state root.

## Common pitfalls

- **Inscription size limit**: Bitcoin standardness limits non-witness
  data to 100 kB; sequencer must split large batches across multiple
  txs and link them.
- **Reorg sensitivity**: a 1-block Bitcoin reorg invalidates the latest
  inscription. Citrea waits for ~6 BTC confs before considering a state
  root final.
- **Prover liveness**: if no prover finishes within timeout, sequencer
  cannot finalise; bridge withdrawals stall. Citrea has multiple
  permissionless provers.
- **Censorship resistance**: sequencer can censor users for up to a
  forced-inclusion window. Force-include path requires user to post
  their own L2 tx via Bitcoin inscription.
- **STARK soundness**: a bug in RISC0 -> proof of false claims accepted.
  Treat zk-roll-ups as economically secured until field-proven.

## References

- Citrea documentation (docs.citrea.xyz).
- RISC0 Bonsai prover docs.
- BitVM2 paper — Linus, Aumayr, Maffei 2023.
- Clementine bridge paper, Citrea team.
