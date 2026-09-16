# Strata Architecture Overview - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/strata`.
> Canonical source: https://docs.alpen.org/ (docs.stratabtc.org now
> redirects here)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/strata/SKILL.md

## Concept

Strata is a Bitcoin-anchored zk-rollup stack that combines an EVM
execution environment (built on Reth) with an SP1 zk-prover and a
garbled-lock ("glock") two-way peg. Like Citrea, it uses Bitcoin for
data availability and posts validity proofs of L2 batches. Strata adds
a "checkpoint" architecture: each anchor commits a range of L2 blocks
in one Bitcoin transaction.

As of September 2026 the stack is split into two named chains:

- **Alpen** - the EVM-compatible execution rollup users transact on.
- **Strata** - the lightweight orchestration layer (OL) that aggregates
  Alpen's proven state commitments, checkpoints them to Bitcoin, and
  owns the enshrined **Strata bridge**.

Material written before mid-2026 (including earlier revisions of this
article) uses "Strata" for the whole rollup; that naming predates the
split. Strata can in principle host several SNARK-account programs
sharing one bridge, but Alpen is the only one deployed as of
September 2026.

## Walkthrough / mechanics

### Components

- **Alpen Reth fork**: EVM execution client; tracks the L2 state.
- **Alpen sequencer**: assembles L2 blocks (~5 s), seals batches
  (~2 h / 1,440 blocks), posts state diffs to Bitcoin for DA, and
  produces the SNARK Account Update Proof.
- **Strata sequencer**: validates account-update proofs, produces
  Strata blocks (~60 min) and posts checkpoints to Bitcoin (~10 h).
- **Anchor State Machine (ASM)**: parses checkpoints from Bitcoin,
  determines the canonical Strata chain, tracks deposits/withdrawals.
- **SP1 prover**: generates the proof of execution + state-transition
  correctness using Succinct Labs' SP1 zkVM (RISC-V proving), behind
  Alpen's `zkaleido` zkVM abstraction.
- **DA layer**: Bitcoin - aggregated batch state diffs plus checkpoints.
- **Bridge**: glock-based fraud-proof bridge (the "Strata bridge");
  operators front BTC and can be challenged by watchtowers.
- **Bundler**: ERC-4337 `UserOperation` relay (optional path).
- **Full nodes**: Strata full nodes read checkpoints from Bitcoin;
  Alpen full nodes derive their canonical chain from them.

### Batches and checkpoints

L2 time is layered. Roughly every 2 hours (1,440 Alpen blocks) the
Alpen sequencer:

1. Posts the batch's aggregated state diff to Bitcoin as DA.
2. Produces one SNARK proof attesting both that the batch executed
   correctly and that the DA transaction is in a valid Bitcoin block.
3. Submits that proof to the Strata sequencer as a SNARK Account
   Update Transaction.

Roughly every 10 hours the Strata sequencer seals Strata blocks into an
epoch and posts a **checkpoint** to Bitcoin carrying the proven Strata
state commitment (which commits to Alpen's state) plus a SNARK proof of
correct Strata execution. Withdrawal intents ride along in the
checkpoint, where bridge operators pick them up. Once the Bitcoin block
carrying a checkpoint is buried to the maturity depth, the covered
Alpen blocks are final.

### Bridge trust model

The Strata bridge is a **1-of-N** design: a single functional honest
operator is enough to guarantee both deposit safety and withdrawal
liveness, unlike majority-multisig bridges. Operators stake a slashable
bond that pays watchtowers who successfully challenge a false claim.
Deposits and withdrawals use a fixed denomination set by bridge
consensus rules - **2 BTC on Testnet III**, with a 0.003 BTC withdrawal
fee; mainnet values are unannounced as of September 2026. Deposit UTXOs
are consumed FIFO by deposit index.

### Bridge: peg-in

1. User broadcasts a **Deposit Request Transaction** paying the
   denomination to a P2TR address where:
   - The key path is unspendable (BIP 341 NUMS point).
   - Script path 1, "deposit path": N-of-N MuSig2 multisig over the
     operator set.
   - Script path 2, "reclaim path": timelocked back to the user if the
     bridge does not move the funds within 36 blocks (~6 hours).

   The first output must be an `OP_RETURN` carrying Strata magic bytes,
   the user's x-only pubkey and the Alpen destination, so sequencer and
   bridge nodes can detect it.
2. Operators pre-sign the glock transaction graph for the deposit
   off-chain; no operator signs the Deposit Transaction until it holds
   every other operator's signature on that graph.
3. Operators broadcast the **Deposit Transaction** spending the deposit
   path into the N-of-N bridge address.
4. After the finality depth of **12 confirmations**, an equivalent
   amount of BTC is minted to the user's Alpen address.

### Bridge: peg-out

1. User sends the denomination to the bridge withdrawal precompile
   (`0x5400...0001`) on Alpen, burning the BTC there.
2. Once the Strata checkpoint proving the request reaches a maturity
   depth of 12 blocks, the protocol assigns an operator. If that
   operator does not front the funds within 504 blocks (~3.5 days), a
   new operator is assigned.
3. The assigned operator fronts BTC from its **own** address, minus the
   fulfillment fee (**withdrawal fulfillment transaction**).
4. After that confirms, the operator broadcasts a **claim transaction**
   spending a bridge UTXO to itself, opening the glock challenge game.
5. Watchtowers have **1,064 blocks (~1.4 weeks)** to challenge.
   Unchallenged, the operator spends the bridge payout after the window.
6. If challenged: the operator answers with a bridge proof transaction,
   the watchtower with a counterproof, the operator with a "watchtower
   NACK" to disprove it. A correct watchtower ACK blocks the payout and
   opens the operator's stake to slashing, claimable by the watchtower.
   After successfully defending, the operator has ~1.5 weeks to take the
   payout or the protocol treats the claim as false.

No operator cooperation is needed to fulfil or be reimbursed for a
withdrawal.

### History: the BitVM2 bridge design

Through 2024-2025 Alpen designed and published a **BitVM2**-style
bridge (chunked on-chain SNARK verifier, Take/Assert/Disprove flow) and
was one of BitVM2's largest contributors. In July 2025 Alpen announced
**Glock** - garbled circuit-based script locks - and reoriented the
company to it, citing BitVM2's prohibitive transaction cost, design
complexity and ~5 BTC operator stake as unfixable inside that paradigm;
the Glock paper (August 2025) claims 430-550x on-chain efficiency over
BitVM2. **Mosaic** (eprint 2026/812, received April 2026, revised July
2026; announced by Alpen 7 May 2026) supplied the practical
maliciously-secure verifier, and Testnet III (July 2026) shipped it as
what Alpen calls the first publicly usable garbled-circuit verifier for
Bitcoin applications. Alpen's own December 2024 bridge post now carries
a historical-note banner stating that Glock supersedes its earlier
BitVM and BitVM2 directions. Treat any BitVM-based description of the
Strata bridge as historical.

### Account abstraction

Alpen supports both EOA transactions and ERC-4337 `UserOperation`s.
Alpen Labs runs a bundler that accepts `UserOperation`s and relays them
through an entrypoint contract; anyone can run their own. EOA
transactions from any EVM wallet work unchanged - AA is an additional
path, not a replacement for `eth_sendRawTransaction`.

### Network parameters (Testnet III, as of September 2026)

- 5 s Alpen block time; 36,000,000 block gas limit; EVM chain ID 20310.
- EVM compatibility up to the Prague release (no `0x0a` KZG precompile).
- Alpen batch to Bitcoin ~every 2 hours; Strata checkpoint ~every 10 hours.
- Withdrawals: front-paid by an operator once the checkpoint matures;
  operator reimbursement after the ~1.4-week challenge window.

## Worked example

Alice deposits the bridge denomination (2 BTC on Testnet III).

```
1. Alpen CLI / bridge UI gives a P2TR deposit address (NUMS key path,
   N-of-N MuSig2 deposit path + user reclaim path).
2. Alice broadcasts the Deposit Request Transaction: OP_RETURN first
   (magic bytes + x-only pubkey + Alpen address), then the 2 BTC output.
3. Operators pre-sign the glock graph, then broadcast the Deposit Tx
   into the N-of-N bridge address.
4. After 12 confirmations, 2 BTC is minted to Alice's Alpen address.
5. Alice uses it in any EVM dApp on Alpen.
```

Withdrawal:

```
1. Alice sends 2 BTC to 0x5400...0001 on Alpen; the BTC is burned.
2. After the Strata checkpoint matures (12 blocks), operator Op1 is
   assigned and fronts 2 BTC minus the fee from its own address.
3. Op1 broadcasts the claim transaction against a bridge UTXO.
4. After 1,064 blocks (~1.4 weeks) with no watchtower challenge, Op1
   spends the bridge payout.
```

## Common pitfalls

- **Sequencer outage**: blocks paused; users with pending peg-out wait
  for next sequencer (force-inclusion path requires Bitcoin inscription
  by user).
- **Proof generation slowness**: SP1 proving over EVM execution takes
  minutes-to-hours; the batch/checkpoint cadence (~2 h and ~10 h) is
  sized to amortise it.
- **Operator capital lockup**: the ~1.4-week challenge window means
  operators front and hold capital; fee pricing must reflect
  time-value. Glock's smaller on-chain footprint cuts the *stake*
  needed relative to BitVM2's ~5 BTC; Alpen says the improved
  economics make the protocol "more accessible to both operators and
  challengers" (July 2025).
- **Fixed denomination**: deposits and withdrawals are only valid at the
  bridge's configured denomination (2 BTC on Testnet III). Arbitrary
  amounts are not bridgeable; splitting happens on L2.
- **Signet, not mainnet**: every BTC figure in Alpen docs means signet
  BTC as of September 2026. Mainnet parameters are unannounced.
- **Trust assumptions**: while the validity proof is trustless once
  finalised, operator front-payment is a counterparty trust within
  the challenge window if no operator picks up your withdrawal.

## References

- Alpen documentation (docs.alpen.org) - system architecture,
  transaction lifecycle, Bitcoin bridge, specifications.
- github.com/alpenlabs/alpen (formerly `alpenlabs/strata`),
  alpenlabs/strata-bridge, alpenlabs/mosaic.
- "Glock: A new standard for verification on Bitcoin", Alpen Labs,
  July 2025; Glock paper, August 2025.
- "Mosaic: Practical Malicious Security for Garbled Circuits on
  Bitcoin", eprint 2026/812 (received April 2026, revised July 2026);
  announced by Alpen Labs 7 May 2026.
- "Alpen enters partner integration phase" (Testnet III), Alpen Labs,
  28 July 2026.
- SP1 zkVM (Succinct Labs).
- BitVM2 (Linus et al. 2024) - historical basis of the earlier design.
