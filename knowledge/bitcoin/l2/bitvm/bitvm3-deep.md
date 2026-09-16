# BitVM3 Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/bitvm`.
> Canonical source: https://bitvm.org/ (BitVM3 working drafts 2025)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/bitvm/SKILL.md

## Concept

BitVM3 is the third-generation BitVM family of designs. It targets two
limitations of BitVM2: (a) the enormous setup ceremony (~10 GB
pre-signed txs per claim) and (b) the per-claim Lamport overhead. BitVM3
introduces **succinct disprove proofs** based on SNARK-style verifiers
with fewer revealed bits, and **N-party stateful vaults** suitable for
staking and bridges with thousands of participants.

As of September 2026, BitVM3 is still a research item; multiple teams
(Babylon, Citrea) are prototyping variants, and no BitVM3 deployment is
confirmed on Bitcoin mainnet. Babylon's Trustless Vaults, announced on
BitVM3 in August 2025, are public testnet as of September 2026.

**Alpen Labs / Strata left that list in July 2025.** Alpen's post
"Glock: A new standard for verification on Bitcoin" (15 July 2025)
states that Liam Eagen and Fairgate Labs "recently reported separate
core flaws in its RSA-based construction, rendering it unsafe to build
upon", and that the "company's direction now fully orients towards
Glock in the development of the Alpen ZK rollup and the Strata bridge".
Glock is written up as "Glock: Garbled Locks for Bitcoin" (Liam Eagen
and Alpen Labs, IACR eprint 2025/1485, received August 2025). Treat
older sources listing Strata as a BitVM3 adopter as historical.

**Design note (September 2026).** The academic write-up - "BitVM3:
Efficient Bitcoin Bridges via Garbled Circuits" (Robin Linus Woll,
Alexopoulos, Aumayr, Avarikioti, Maffei, Tse; IACR eprint 2026/933,
received May 2026, revised June 2026) - settles on a different
primitive from the SNARK/KZG sketch described below: in BitVM3-CORE the
challenger evaluates a **garbled circuit** entirely off-chain, which
the paper reports as a nearly 1000x on-chain saving (total on-chain
cost ~$9, with the challenge transaction itself ~$0.20, against a
~$16,000 worst-case dispute cost under BitVM2). Read the KZG-flavoured
mechanics below as one line from the 2025 working drafts rather than as
the design in the 2026 paper.

## Walkthrough / mechanics

### Improvements vs BitVM2

| Aspect | BitVM2 | BitVM3 |
|--------|--------|--------|
| Setup size | ~10 GB pre-signed tx graph | ~100 MB or less |
| Disprove tx | Single but per-gate | Single, succinct (SNARK-based) |
| Multi-party | 2-party | N-party native |
| Challenge cost | per-gate | per-claim O(log) |
| State transitions | static computation | stateful vaults |

### Succinct disprove proofs

Instead of revealing bits at every wire, BitVM3 uses a **zk-SNARK
verifier circuit** in Bitcoin script. Operator commits to the entire
proof using a polynomial commitment (KZG-style); challenger picks a
single evaluation point. Operator reveals the evaluation; challenger
provides the expected value. Mismatch -> disprove via single Bitcoin tx.

This requires SNARK-friendly hash functions (Poseidon variants) compiled
to Bitcoin script. Non-trivial but feasible (10s of KB witness).

### N-party stateful vaults

BitVM3 supports vaults shared by N parties (e.g. 1000 stakers in a
Babylon-style staking pool). Each party has independent slashing
conditions; operator manages aggregate UTXO.

State transitions (e.g. "staker P withdraws V") are encoded as state
commitments in a Merkle tree; operator updates the root and provides
SNARK proof of valid update.

### Compute primitives

BitVM3 uses **MATT (Merkleize All The Things)** approach: every state
is a Merkle root, every transition is a SNARK proving Merkle update.
This enables generic stateful computation without explicit script
support for state.

### Use cases

- **Decentralized staking pools**: N stakers share a vault; slashing
  per FP misbehavior.
- **Optimistic rollups with massive operator sets**: rollups where
  operators rotate, each with their own collateral.
- **Lightning-style payment channels with N participants**: not just
  bilateral.

## Worked example

Babylon vault with 1000 stakers, total 1000 BTC. Staker #437 unstakes.

```
1. Operator updates state Merkle: leaf #437 amount 0, others unchanged.
   Computes new state root R'.
2. Operator pre-signs SNARK proof: "given old state root R and unstake
   request for #437, new state root is R'".
3. Operator broadcasts vault state-update tx with commitment to (R, R',
   SNARK proof).
4. Challenge window 14 days.
5. If challenger Bob suspects fraud (e.g. operator stole leaf #437's BTC
   to themselves):
   - Bob requests SNARK evaluation at random point;
   - Operator reveals;
   - Bob computes expected value off-chain;
   - If mismatch: Bob broadcasts disprove. Operator slashed.
6. If honest: state root R' becomes new vault state. Staker #437
   receives BTC via separate exit tx tied to R'.
```

## Common pitfalls

- **SNARK-friendly hashes**: Poseidon-family hashes are not Bitcoin
  primitives. They must be implemented in Bitcoin script (slow, large
  witnesses) or simulated via specialized commitment schemes.
- **Trusted setup**: KZG-style polynomial commitments require Powers
  of Tau ceremony; if compromised, soundness breaks.
- **Operator collusion** with covenant signers (in setup phase) can
  fake state updates. Pre-signing ceremony participants must be
  diverse and incentivized to publish challenges.
- **Liveness**: stalled state machines lock funds; protocol must have
  fallback timeouts for individual stakers to force exit.
- **Composability**: combining multiple BitVM3 vaults (e.g. staking
  + bridge) requires careful nesting; bug-prone.

## References

- Robin Linus Woll, Alexopoulos, Aumayr, Avarikioti, Maffei, Tse.
  "BitVM3: Efficient Bitcoin Bridges via Garbled Circuits", IACR eprint
  2026/933 (received May 2026, revised June 2026).
- bitvm.org research drafts (BitVM3 work-in-progress), including the
  line whose RSA-based construction Liam Eagen and Fairgate Labs had
  reported broken as of July 2025.
- Liam Eagen and Alpen Labs. "Glock: Garbled Locks for Bitcoin", IACR
  eprint 2025/1485 (received August 2025).
- Alpen Labs. "Glock: A new standard for verification on Bitcoin",
  15 July 2025. https://www.alpen.org/blog/glock-verification-on-bitcoin
- "MATT - Merkleize All The Things" (Salvatore Ingala 2023).
- Poseidon hash function (Grassi et al. 2019).
- KZG polynomial commitments (Kate, Zaverucha, Goldberg 2010).
