# Citrea Clementine BitVM2 Bridge Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/citrea`.
> Canonical source: https://github.com/chainwayxyz/clementine
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/citrea/SKILL.md

## Concept

Clementine is Citrea's BitVM2-based two-way peg. Pegging in is
permissionless and trustless (anyone can deposit and mint cBTC on L2),
but it is **fixed-denomination**: a single Clementine operation moves
exactly 10 BTC in or 10 cBTC out, with no partial or custom amounts
(Citrea docs, as of September 2026).
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

### Fixed 10 BTC denomination

Clementine is usable **only at a fixed size**. Each deposit moves
exactly 10 BTC and mints 10 cBTC; each withdrawal burns exactly 10
cBTC and pays out ~10 BTC net of fees. Bridging 40 BTC means four
separate 10 BTC deposits. The constraint falls out of the pre-signed
BitVM2 transaction graph — it is not a tunable policy parameter
(Citrea docs, as of September 2026).

Amounts below 10 BTC do not touch Clementine at all. Citrea's docs
route those users to third-party cross-chain swaps — Symbiosis, and
Atomiq Exchange (which also carries a Lightning route) as of September
2026. Those routes are AMM/liquidity-provider trades subject to
quotes and slippage, and inherit none of Clementine's 1-of-N security
model.

### Peg-in (deposit)

1. User sends 10 BTC to a Taproot address encoding two spend paths: a
   **bridge path** requiring all signers' signatures with the user's
   EVM address inscribed in the witness script, and a **refund path**
   the user can take after a 200-block `OP_CSV` timelock if the bridge
   fails to act.
2. **Move to vault**: signers sign the `MovetoVault` transaction that
   moves the deposit into a vault UTXO spendable only along the
   bridge's pre-signed exits (covenant emulation via MuSig-style
   aggregation).
3. **Mint**: once `MovetoVault` finalises (6+ blocks), the `Bridge`
   system contract on Citrea validates it through the
   `BitcoinLightClient` contract and mints 10 cBTC to the Citrea
   address bound in the deposit.
4. Citrea state root reflects the mint after sequencer batches it.

### Peg-out (withdrawal)

1. User L2 transaction burns 10 cBTC and emits a withdrawal event with
   target Bitcoin scriptPubKey.
2. Sequencer includes the burn in a batch and posts the batch on
   Bitcoin (with state root + STARK).
3. **Operator front-payment**: any operator scans batches, then sends
   ~10 BTC net of fees from its own wallet to the user. Operator now
   has a reimbursement claim.
4. **Reimbursement claim** (after a delay window): operator constructs a
   BitVM2 chained transaction structure:
   - **Kickoff** tx anchored to the operator's claim.
   - **Assert** chain: operator publishes commitments to the L2 state
     root, the burn tx hash, and intermediate STARK verifier outputs.
   - **Take** tx: spends the peg UTXO to operator after the challenge
     window (~1.5 days, Citrea docs as of September 2026) if no
     challenge.
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

Operator pre-stakes collateral into a separate UTXO with timelocks. On
Citrea mainnet the bond is **~2 BTC** and backs a whole *round* of many
user withdrawals rather than scaling per withdrawal — if any claim in
the round is proven fraudulent, the bond is slashed and none of that
round's reimbursements pay out (Citrea docs, as of September 2026). A
successful disprove takes operator's collateral and refunds the user
(or compensates challenger, depending on protocol parameter).

## Worked example

User Alice peg-outs 10 cBTC (one Clementine operation — the only size
the bridge accepts).

```
1. L2 tx: Alice burns 10 cBTC -> withdrawal event (script=bc1q...alice, amount=10)
2. Sequencer batches; Bitcoin block N includes the batch inscription.
3. Operator Op1 sends ~10 BTC minus fee to Alice on Bitcoin from its
   liquidity wallet.
4. Op1 pre-publishes Kickoff at block N+50.
5. Op1 publishes Assert chain at blocks N+60..N+90.
6. Challenge window N+90..N+306 (~216 blocks, i.e. the ~1.5 days
   Citrea docs give at 10 min/block, as of September 2026):
   - Watcher checks each commitment vs replay of L2 state.
   - If all valid -> Op1 broadcasts Take tx claiming 10 BTC from peg UTXO.
   - If any invalid -> watcher broadcasts Disprove revealing equivocation.
7. Take confirms -> peg UTXO balance reduced by 10 BTC; operator
   collateral returned.
```

## Common pitfalls

- **Operator censorship**: if no operator picks up the peg-out request,
  user must wait (or post a higher fee). Mitigation: protocol-mandated
  rotation of operator queue.
- **Long challenge window**: ~1.5 days on Citrea mainnet (Citrea docs,
  as of September 2026). User receives BTC immediately (front-payment),
  but operator capital is locked until Take. Operator fee compensates.
- **10 BTC granularity**: the canonical bridge is unusable for
  retail-size flows. Anyone moving less than 10 BTC is on a
  third-party swap route with its own trust and slippage profile.
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
- Citrea docs, "Using Clementine" and "Bridge to Citrea" — fixed 10
  BTC / 10 cBTC sizing and the sub-10-BTC third-party routes (read
  September 2026).
- Citrea blog, "6 Months of Clementine" (4 August 2026) — ~150 BTC
  cumulative bridging volume in the first six months of production.
