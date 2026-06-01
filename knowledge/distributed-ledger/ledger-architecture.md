# Distributed-Ledger Architecture (general)

## Overview

A distributed ledger is an append-only, replicated, tamper-evident log whose
state is agreed by mutually-distrusting parties. This article generalizes the
Bitcoin-specific material into engine-agnostic design guidance; for Bitcoin and
Lightning specifics, see the dedicated bitcoin documentation.

## Do you actually need a ledger?

The most important architectural decision is whether a ledger is warranted at
all. It earns its considerable cost only when you have all of: multiple parties
that **do not trust each other**, a need for **shared, auditable state**, and no
acceptable trusted intermediary. If a single organization controls the data, an
ordinary database — append-only and audited if needed — is simpler, faster, and
cheaper. Most "blockchain" projects are better served by a database.

## Consensus families

Consensus provides agreement and Sybil resistance.

| Family | Sybil resistance | Finality | Trade-off |
|--------|------------------|----------|-----------|
| Proof of Work | Burn energy | Probabilistic | Robust and simple, but slow and energy-intensive |
| Proof of Stake | Stake capital | Fast | Efficient, but validator-set and weak-subjectivity concerns |
| BFT (PBFT, Tendermint) | Known validator set | Instant, deterministic | Fast, but bounded validators → more permissioned |

Permissionless (open) networks favor PoW or PoS; permissioned consortia favor
BFT.

## Layering: L1 vs L2

- **L1** is the base chain providing security and data availability. Scaling it
  directly runs into the trilemma.
- **L2** moves execution off the base chain while inheriting its security:
  - **Rollups** post compressed transaction data and proofs to L1. **ZK
    rollups** use validity proofs (fast finality, harder to build); **optimistic
    rollups** use fraud proofs (a challenge window adds withdrawal latency).
  - **State/payment channels** settle bilaterally off-chain with on-chain
    open/close — ideal for high-frequency payments.
  - **Sidechains** run their own security and bridge to L1 (weaker guarantees).

## Other decisions

- **Permissioned vs permissionless** follows the trust model and regulation.
- **State model:** UTXO (parallelizable, better privacy, harder smart contracts)
  versus account-based (stateful contracts, simpler, sequential nonces).
- **The scalability trilemma:** decentralization, security, and scalability —
  optimize two; L2s are the usual escape valve.
- **Finality:** probabilistic (PoW) versus deterministic/instant (BFT) affects
  user experience and the safety of bridges.

## Guidance

A single trust domain → a database. Multi-party and open → a permissionless L1
with L2s for scale. A known consortium → a permissioned BFT chain.
High-frequency payments → channels. Always justify the ledger over a database
first.
