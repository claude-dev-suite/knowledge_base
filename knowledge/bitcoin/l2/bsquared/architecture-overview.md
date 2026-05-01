# B^2 Network (BSquared) Architecture Overview - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/bsquared`.
> Canonical source: https://docs.bsquared.network/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/bsquared/SKILL.md

## Concept

B^2 Network is an EVM-compatible Bitcoin L2 with a **two-tier validity
architecture**: a zk-rollup execution layer plus a "Validity-DA" layer
that records state commitments and zk proofs onto Bitcoin. Where Citrea
posts everything inline, B^2 delegates DA to Celestia/EigenDA-style data
sources and posts only commitments + ZK proofs to Bitcoin.

## Walkthrough / mechanics

### Layer roles

- **Rollup layer**: EVM execution (forked geth), processes user txs.
- **DA layer**: external (Celestia at launch), holds full data blobs.
- **Bitcoin layer**: stores commitments to DA blobs + zk-SNARK proofs of
  state validity.
- **Bridge**: hybrid optimistic + zk; zk path settles in days, optimistic
  in 7 days as fallback during prover outages.

### Block lifecycle

1. Sequencer produces L2 block. Forwards block data to DA layer.
2. Prover (Risc0/Halo2 hybrid) generates zk proof for batch of L2 blocks.
3. **Commit transaction** on Bitcoin: Taproot output committing to
   `(da_blob_root, state_root, zk_proof_hash)` via P2TR Tapscript.
4. **Reveal transaction** later, posting full proof inline via
   inscription.
5. After Bitcoin confirmation, B^2 nodes verify the ZK proof off-chain
   and accept the new state root.

### Bitcoin commit format

A B^2 commit is a Taproot reveal:

```
Taproot witness:
  OP_FALSE OP_IF
    "b2nt"           magic 4 bytes
    <da_root>        32 bytes
    <state_root>     32 bytes
    <proof>          ~256 KB Halo2 proof
  OP_ENDIF
```

The whole inscription costs ~250 KB witness data; at 5 sat/vB cost is
manageable but not cheap.

### Bridge architecture

B^2 uses a "BitVM2-style" bridge for trust-minimisation in production
phases. Operators front BTC for peg-outs; their claim is challengeable
via on-chain disprove transactions during a window.

In earlier mainnet, operators ran a multi-sig with social-trust assumption
(akin to Liquid). Roadmap migrates to BitVM2 fraud proofs.

### Account model

EVM-equivalent with extras:

- Native gas token: B^2 (B^2 Network token).
- Bridged BTC token: WBTC-style ERC-20 from peg.
- Smart-contract precompile for Bitcoin SPV proofs (validate Bitcoin
  block headers + Merkle inclusions).

### Modular ZK proof system

B^2 uses **Halo2 + KZG** (PLONK family) plus a "ZK aggregation" stage
(Plonky2-like) to compress multiple batch proofs into one final
SNARK suitable for Bitcoin inscription.

## Worked example

L2 throughput: 100 user txs in batch B_4567.

```
1. Sequencer block 4567 added to L2 chain, da_blob posted to Celestia.
2. Prover generates Halo2 proof π for the batch (~2 minutes proving on GPU).
3. Final aggregated proof size: ~250 KB.
4. Bitcoin commit tx at BTC block 850 100 (Taproot reveal).
5. After ~6 confs, B^2 nodes verify proof and accept state root S_4567.
```

For peg-out: user burns wBTC on B^2, operator front-pays user, operator
claim path is in development (BitVM2 disprove logic).

## Common pitfalls

- **DA layer dependency**: if Celestia fails, B^2 cannot reconstruct
  state from chain alone. A "pessimistic DA" inscription path can fall
  back to Bitcoin but at much higher cost.
- **Prover bottleneck**: Halo2 GPU proving is fast but not free; long
  blocks cause batch delays.
- **Inscription standardness**: Bitcoin Core's standardness rules cap
  witness data ~400 KB; large proofs require splitting + chained reveal.
- **Force-inclusion gap**: until force-include via Bitcoin inscription
  is implemented, sequencer censorship is mitigated only socially.
- **Halo2 trusted setup**: Powers-of-Tau ceremony required for KZG;
  inherits the security of that ceremony.

## References

- B^2 Network technical paper (docs.bsquared.network).
- Halo2 / KZG reference (Bowe et al., Zcash docs).
- Celestia DA spec.
