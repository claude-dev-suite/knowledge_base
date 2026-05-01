# BIP78 PayJoin Flow Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/privacy/payjoin`.
> Canonical source: https://github.com/bitcoin/bips/blob/master/bip-0078.mediawiki
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/privacy/payjoin/SKILL.md

## Concept

BIP78 (PayJoin / Pay-to-EndPoint, P2EP) is an interactive sender-receiver
protocol where the **receiver contributes one or more inputs** to the sender's
transaction. The on-chain output looks like an ordinary 2-input / 2-output
transaction, breaking the **common-input-ownership heuristic** used by chain
analysis. Both parties save a separate consolidation transaction fee, and the
amount paid is hidden because the receiver's contribution adds to the output
they get.

## Walkthrough / mechanics

**Roles**: `sender` (paying party) and `receiver` (running an HTTP endpoint).
The receiver advertises a `bitcoin:` URI containing a `pj=` parameter that
points to an `https://` (clearnet or onion) endpoint.

```
bitcoin:bc1q...?amount=0.01&pj=https://merchant.example/pj
```

Step-by-step:

1. **Original PSBT**. Sender constructs and signs a normal payment PSBT
   (`original_psbt`) paying the receiver's advertised address with the
   advertised amount. Sender can broadcast this if receiver is unreachable
   ("fall-back tx"), giving graceful degradation.
2. **POST**. Sender HTTP-POSTs the base64 PSBT to the `pj=` URL with query
   params `v=1`, `additionalfeeoutputindex`, `maxadditionalfeecontribution`,
   `disableoutputsubstitution=true|false`, `minfeerate`.
3. **Receiver validation**. Receiver MUST check:
   - PSBT is finalised on the sender side (signed inputs).
   - All sender inputs are SegWit (P2WPKH/P2SH-P2WPKH/P2TR) so receiver can
     add inputs of the same type without leaking `spk` heterogeneity.
   - Receiver address output amount matches what was advertised.
4. **PayJoin PSBT**. Receiver picks one of its UTXOs `Ur`, adds it as a new
   input, increases the receiver-output value by `value(Ur)`, signs only its
   own input, and returns the partially-signed `payjoin_psbt`. Receiver MAY
   adjust fee using `additionalfeeoutputindex` if extra weight pushes feerate
   below `minfeerate`.
5. **Sender finalisation**. Sender checks invariants: same outputs (except
   amount-substitution), no new outputs, `feerate >= minfeerate`, sender
   inputs unchanged, then signs the new input set and broadcasts.

## Worked example

Sender wants to pay 0.01 BTC. UTXO: 0.025 BTC. Receiver UTXO: 0.05 BTC.

| Step | Inputs | Outputs |
|------|--------|---------|
| original | 0.025 | 0.01 to merchant + 0.0148 change (fee 0.0002) |
| payjoin  | 0.025 + 0.05 | 0.06 to merchant + 0.0148 change (fee 0.0002) |

A blockchain observer sees a 2-in / 2-out transaction with a 0.06 BTC payment
and a 0.0148 BTC change. The actual sale amount (0.01 BTC) is invisible:
either output could be the payment.

## Common pitfalls

- **Snooping attack**: a malicious sender can request multiple PayJoin PSBTs
  with the same `original_psbt` to try to enumerate the receiver's UTXOs.
  Receivers MUST cache the original PSBT input set per sender session and
  return the same UTXO contribution for the same original (BIP78 section
  "Receiver's UTXO probing").
- **Fee bump must come from sender output** when the receiver adds a same-size
  input; otherwise the receiver subsidises the sender. Use
  `additionalfeeoutputindex` to point at sender change.
- **Mixing input script types** leaks the receiver: if sender is P2WPKH and
  receiver adds a P2TR input, chain analysis flags the unusual mix.
- **Output substitution** (replace receiver scriptPubKey with another) is
  a trust amplifier and is off by default.

## References

- BIP78 PayJoin specification.
- BTCPay Server reference receiver implementation.
- "PayJoin: a Better Way to Pay with Bitcoin" — Chris Belcher writeup.
