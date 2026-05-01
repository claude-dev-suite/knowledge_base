# Citrea Clementine BitVM2 Bridge Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/citrea`.
> Canonical source: https://github.com/chainwayxyz/clementine
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/citrea/SKILL.md

## Concept

Clementine is Citrea's BitVM2-based two-way peg. Pegging in is
permissionless and trustless (anyone can deposit and mint cBTC on L2).
Pegging out uses a **operator-relayed claim with on-chain fraud proofs**
via BitVM2: operators front the BTC for L2 withdrawals, then claim
reimbursement from a peg UTXO; their claim can be challenged for up to
N blocks via a BitVM2 disprove transaction.

## Walkthrough / mechanics

### Roles

- **Verifiers (N=N_v)**: hold the peg-in multisig key share. Operator a
  subset of these.
- **Operators**: front BTC liquidity for peg-outs. Earn fees on
  successful claims.
- **Watchers**: anyone running an L2 verifier; can submit fraud proofs.

### Peg-in (deposit)

1. User picks an operator pool. Operator pool publishes peg-in script:
   `MuSig2(verifier_set)` or threshold script with explicit script-path
   alternatives (refund timelock to user).
2. User funds output `O_in` paying the peg script.
3. Operators co-sign the L2 mint transaction crediting user's L2 address
   with `cBTC = value(O_in)`.
4. Citrea state root reflects the mint after sequencer batches it.

### Peg-out (withdrawal)

1. User L2 transaction burns `cBTC` and emits a withdrawal event with
   target Bitcoin scriptPubKey.
2. Sequencer includes the burn in a batch and posts the batch on
   Bitcoin (with state root + STARK).
3. **Operator front-payment**: any operator scans batches, then sends
   `value` BTC from its own wallet to the user. Operator now has a
   reimbursement claim.
4. **Reimbursement claim** (after a delay window): operator constructs a
   BitVM2 chained transaction structure:
   - **Kickoff** tx anchored to the operator's claim.
   - **Assert** chain: operator publishes commitments to the L2 state
     root, the burn tx hash, and intermediate STARK verifier outputs.
   - **Take** tx: spends the peg UTXO to operator after `T` blocks if
     no challenge.
5. **Challenge window**: any watcher who finds the operator's claim
   inconsistent (wrong state root, wrong amount, fake burn) submits a
   **disprove tx** spending operator's collateral. Disprove tx contains
   a single bit-commitment opening that reveals operator's lie.

### BitVM2 mechanics in two paragraphs

BitVM2 expresses computation in Bitcoin script via **bit commitments**
and **Lamport-style hash equivocation**. Each gate of the verifier
circuit is broken into per-bit commitments; the operator must publish
opening for each bit on chain. A challenger only needs to publish ONE
contradiction (e.g. operator committed to bit 0 in tx A and bit 1 in
tx B for the same wire); the on-chain disprove tx requires only one
hash equivocation, ~few KB.

In Clementine, the verifier circuit is a STARK-verifier reduction so
that proving "Citrea state root R is valid" reduces to a single boolean
output of a deterministic computation.

### Operator collateral

Operator pre-stakes `K * value` BTC into a separate UTXO with timelocks.
A successful disprove takes operator's collateral and refunds the user
(or compensates challenger, depending on protocol parameter).

## Worked example

User Alice peg-outs 1 cBTC.

```
1. L2 tx: Alice burns 1 cBTC -> withdrawal event (script=bc1q...alice, amount=1)
2. Sequencer batches; Bitcoin block N includes the batch inscription.
3. Operator Op1 sends 0.999 BTC (1 - fee) to Alice on Bitcoin from its
   liquidity wallet.
4. Op1 pre-publishes Kickoff at block N+50.
5. Op1 publishes Assert chain at blocks N+60..N+90.
6. Challenge window N+90..N+10000:
   - Watcher checks each commitment vs replay of L2 state.
   - If all valid -> Op1 broadcasts Take tx claiming 1 BTC from peg UTXO.
   - If any invalid -> watcher broadcasts Disprove revealing equivocation.
7. Take confirms -> peg UTXO balance reduced by 1 BTC; operator
   collateral returned.
```

## Common pitfalls

- **Operator censorship**: if no operator picks up the peg-out request,
  user must wait (or post a higher fee). Mitigation: protocol-mandated
  rotation of operator queue.
- **Long challenge window**: typical 1-2 weeks. User receives BTC
  immediately (front-payment), but operator capital is locked until
  Take. Operator fee compensates.
- **Bit-commitment size**: BitVM2 disprove tx is large (~50-100 KB) at
  high fee rates. Make sure challenger collateral covers the worst-case
  fee.
- **Single-operator failure**: collateral covers one fraud event per
  operator; multiple simultaneous frauds require multiple challenges.
- **Reorg of Take tx**: a Bitcoin reorg post-Take could invalidate the
  operator claim; protocol requires deep confs before peg UTXO is
  considered moved.

## References

- Linus, Aumayr, Maffei. "BitVM2: Permissionless Verification on
  Bitcoin" 2024.
- Clementine repo (chainwayxyz/clementine).
- Citrea bridge paper (docs.citrea.xyz).
