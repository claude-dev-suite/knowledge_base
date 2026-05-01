# DLC Contract Lifecycle Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/cryptography/dlcs`.
> Canonical source: dlcspecs (github.com/discreetlogcontracts/dlcspecs)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/cryptography/dlcs/SKILL.md

## Concept

A Discreet Log Contract has five named phases: **offer**, **accept**, **sign**,
**funding**, and **close**. Each phase produces a structured wire message
defined in `dlcspecs/Messaging.md`, and each transitions the contract state
machine on both parties. The construction binds money to oracle attestations
via adaptor signatures over Contract Execution Transactions (CETs); the
on-chain footprint is two transactions in the cooperative case (funding +
settlement) or three with the refund branch. This article traces every
message, every state, and the cleanup conditions that must hold for the
lifecycle to terminate without dust or stuck funds.

## Walkthrough / mechanics

### State machine

```
[OFFERED]    -- offer_dlc msg sent     --> [ACCEPTED]
[ACCEPTED]   -- accept_dlc msg back    --> [SIGNED]
[SIGNED]     -- sign_dlc msg back      --> [FUNDED_PENDING]
[FUNDED_PENDING] -- funding tx confirms --> [CONFIRMED]
[CONFIRMED]  -- oracle attests         --> [CLOSING]
[CLOSING]    -- CET confirms            --> [CLOSED]

Anti-paths:
[FUNDED_PENDING] -- funding never confs --> [ABANDONED]
[CONFIRMED]      -- timeout, refund     --> [REFUNDED]
```

### Phase 1: Offer (`offer_dlc`)

Initiator (Alice) sends:

```
offer_dlc {
  contract_flags
  chain_hash
  contract_info {
    total_collateral
    contract_descriptor   // payout function: outcome -> (a_payout, b_payout)
    oracle_info           // oracle_pubkey, event_id, R_event nonce(s),
                          // outcome_labels
  }
  funding_pubkey          // Alice's contribution to 2-of-2 funding
  payout_spk              // where Alice gets paid in CETs
  payout_serial_id        // for sorting CET outputs canonically
  offer_collateral
  funding_inputs[]        // PSBT-style input descriptors
  change_spk
  change_serial_id
  fund_output_serial_id
  cet_locktime, refund_locktime
  fee_rate_per_vb
}
```

State on Alice: OFFERED. State on Bob (when received): handle_offer.

### Phase 2: Accept (`accept_dlc`)

Bob counter-signs by responding with:

```
accept_dlc {
  temporary_contract_id
  accept_collateral
  funding_pubkey          // Bob's
  payout_spk              // where Bob gets paid
  payout_serial_id
  funding_inputs[]
  change_spk
  change_serial_id
  cet_adaptor_sigs[]      // Bob's adaptor sig per outcome
  refund_signature        // Bob's sig for refund tx
  negotiation_fields      // optional
}
```

`cet_adaptor_sigs[]` has one entry per discrete outcome (or per CET in a tree
for numerical contracts; see [cet-tree-numerical-outcomes.md]). Each entry is
a Schnorr adaptor sig over the outcome's CET, with adaptor point `T_outcome
= R_event + e_outcome * P_oracle`.

State: Bob -> ACCEPTED, Alice -> handle_accept.

### Phase 3: Sign (`sign_dlc`)

Alice verifies all of Bob's adaptor sigs (PreVerify) and refund sig, then
sends:

```
sign_dlc {
  contract_id              // hash of funding outpoint, finalized
  cet_adaptor_sigs[]       // Alice's adaptor sigs per outcome
  refund_signature         // Alice's sig
  funding_signatures[]     // Alice's sigs over funding-tx inputs
}
```

Alice has now committed all her signatures: CETs, refund, funding inputs.
Bob, on receipt, verifies adaptor sigs + refund sig + funding sigs, then
adds his own funding-tx witness data and broadcasts.

State: SIGNED.

### Phase 4: Funding

```
Bob broadcasts funding tx.
Both parties watch chain for confirmations.
At fundamental_threshold confs (default 6 on mainnet), state -> CONFIRMED.
```

At this point both parties have:
- One pre-signed CET per outcome, ready to be adapted with `s_outcome`.
- One pre-signed refund tx, locktimed.

The CETs are NOT broadcast yet.

### Phase 5: Close

Two close paths:

**Path A: Oracle attests**

```
Oracle publishes (s_outcome, outcome_label) for the actual outcome.
The party favored by that outcome (e.g., Alice if outcome = "BTC > 50k"):
  1. Looks up CET[outcome] and pre_sig_alice[outcome], pre_sig_bob[outcome]
  2. Computes t_outcome = s_outcome (the adaptor secret)
  3. Adapts both adaptor sigs to full sigs
  4. Combines into MuSig2/Schnorr-aggregated 2-of-2 witness
  5. Broadcasts CET
```

When CET confirms, state -> CLOSED. Both parties have their payouts in their
respective `payout_spk` outputs.

**Path B: Refund**

```
If `refund_locktime` passes (e.g., 30 days) without oracle attestation:
  Either party:
  1. Combines refund_signature_alice + refund_signature_bob
  2. Broadcasts refund tx
```

State -> REFUNDED. Each party recovers their `offer_collateral` /
`accept_collateral` to their funding_inputs' change addresses.

### Wire encoding

dlcspecs uses Lightning-style TLV encoding. Type IDs registered in
`Messaging.md`:

```
| Type | Name              |
|------|-------------------|
| 42778 | offer_dlc        |
| 42780 | accept_dlc       |
| 42782 | sign_dlc         |
| 42784 | order_negotiation_fields |
```

Transport over Tor / clearnet via Bolt-8 Noise framing (or any reliable
direct channel between parties).

## Worked example

Bilateral price bet on BTC closing price tomorrow. Two outcomes: "above 50k"
or "below 50k", winner-take-all.

```
Setup:
  Alice contributes 0.5 BTC offer_collateral
  Bob contributes 0.5 BTC accept_collateral
  Total funding output: 1.0 BTC (minus fees)

  Oracle: P_oracle, R_event published 24h before settlement
  Outcomes:
    e_above = H(R_event || "above 50k")
    e_below = H(R_event || "below 50k")
    T_above = R_event + e_above * P_oracle
    T_below = R_event + e_below * P_oracle

  CETs:
    CET_above pays Alice 1.0 BTC, Bob 0
    CET_below pays Alice 0,       Bob 1.0 BTC

Phase 1: Alice -> offer_dlc -> Bob

Phase 2: Bob accepts. Builds CETs. Computes:
    pre_sig_bob_above = Adaptor.PreSign(d_bob, sighash(CET_above), T_above)
    pre_sig_bob_below = Adaptor.PreSign(d_bob, sighash(CET_below), T_below)
    refund_sig_bob    = Schnorr.Sign(d_bob, sighash(refund_tx))
  Sends accept_dlc with both pre-sigs + refund_sig.

Phase 3: Alice verifies. Computes own pre-sigs:
    pre_sig_alice_above = Adaptor.PreSign(d_alice, sighash(CET_above), T_above)
    pre_sig_alice_below = Adaptor.PreSign(d_alice, sighash(CET_below), T_below)
    refund_sig_alice    = Schnorr.Sign(d_alice, sighash(refund_tx))
  Signs funding tx. Sends sign_dlc.

Phase 4: Bob broadcasts funding tx. 6 confs. State CONFIRMED.

Phase 5: Oracle publishes s_above (price closed at 51k).
  Alice computes:
    t_above = s_above   (the discrete log of T_above)
    s_alice_full = pre_sig_alice_above.s' + t_above mod n
    s_bob_full   = pre_sig_bob_above.s'   + t_above mod n
    aggregated_sig = (R_above, s_alice_full + s_bob_full)
                     // for MuSig2-style 2-of-2; or 2 individual sigs in
                     // P2WSH 2-of-2
    Builds CET_above witness, broadcasts.
  CET_above confirms: Alice's payout_spk receives 1.0 BTC (minus fees).
```

Bob, watching the chain, sees CET_above confirm. He observes the published
sig and could `Extract` `t_above = s_above` from it (or already had it from
the oracle). Either way, no further action needed — Bob's payout is 0 for
this outcome.

## Common pitfalls

- **Releasing funding sigs before adaptor sigs verify.** Alice must run
  PreVerify on every CET adaptor sig from Bob *before* signing the funding
  tx. Otherwise Bob can DOS by sending bad pre-sigs and stealing the funding
  output via refund.
- **Skipping refund tx in setup.** If oracle never attests and there is no
  refund path, funds are permanently locked. The refund tx must be pre-signed
  in the accept/sign exchange.
- **Mismatched serial IDs.** payout_serial_id and fund_output_serial_id
  determine canonical ordering of tx outputs. Mismatched ordering between
  parties produces different sighashes and the funding tx becomes
  unspendable. dlcspecs requires deterministic ordering.
- **Locktime miscount.** `cet_locktime` must be ≤ now+oracle_attestation_time.
  `refund_locktime` must be > cet_locktime + close_window. Inverting them
  enables a race condition where refund and CET could both confirm.
- **Storing CETs only in volatile memory.** A CET set may be megabytes in
  size for numerical contracts. Both parties must persist them durably with
  the same encoding the wire protocol uses.

## References

- dlcspecs: https://github.com/discreetlogcontracts/dlcspecs
- Messaging spec:
  https://github.com/discreetlogcontracts/dlcspecs/blob/master/Messaging.md
- Transactions spec:
  https://github.com/discreetlogcontracts/dlcspecs/blob/master/Transactions.md
- rust-dlc lifecycle:
  https://github.com/p2pderivatives/rust-dlc/tree/master/dlc-manager
