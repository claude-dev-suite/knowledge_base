# LUD-21 LNURL-verify - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/lnurl`.
> Canonical source: LUD-21 (https://github.com/lnurl/luds/blob/luds/21.md)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/lnurl/SKILL.md

## Concept

LUD-21 adds a `verify` URL to the LNURL-pay callback response, letting a
client poll for proof-of-payment after sending. Without LUD-21, a client
that paid an invoice via a routing node only knows whether ITS HTLC
settled; it cannot prove to the merchant (or to its own UI) that the
merchant actually received the funds. LUD-21 closes that gap by exposing
the merchant's view, including the preimage when settled.

## Walkthrough / mechanics

The pay-callback response is extended:

```json
{
  "pr": "lnbc50u1...",
  "verify": "https://merchant.example.com/verify/payhash-or-id",
  "successAction": {...},
  "routes": []
}
```

After paying, the client polls the `verify` endpoint:

```
GET https://merchant.example.com/verify/<id>
```

Possible responses:

```json
// Pending
{ "status": "OK", "settled": false, "preimage": null, "pr": "lnbc..." }

// Settled
{ "status": "OK", "settled": true,
  "preimage": "0123456789abcdef...64-hex", "pr": "lnbc..." }

// Error / unknown
{ "status": "ERROR", "reason": "unknown payment" }
```

The client checks:

```
sha256(hex_decode(preimage)) == bolt11.payment_hash
```

If it matches, the merchant has cryptographic proof that the payment
arrived. The client itself can also use the preimage to terminate any
ongoing HTLC retry loops (e.g. if the original payment "stuck").

## Worked example

```
Client pays:
  invoice payhash = ab12...
  HTLC sent via routing path; client times out at 60s.
  Client unsure whether merchant got paid (network partition).

Client polls verify endpoint every 2s for 30s:
  GET /verify/ab12 -> { "settled": false, "preimage": null }
  GET /verify/ab12 -> { "settled": false, "preimage": null }
  GET /verify/ab12 -> { "settled": true,
                        "preimage": "f0e1d2c3..." }

Verify:
  sha256(f0e1d2c3...) == ab12  -> OK
  Show user: "Payment confirmed by merchant".
  Cancel any pending retries, finalize order, deliver successAction.
```

This is critical for podcast-style boost or paywall flows where the
client display must reflect the merchant's receipt-of-funds, not just
the local wallet's report. LUD-21 is also useful for offline / async
flows where a server (e.g. a cashier desktop) checks payment status
without holding the user's wallet keys.

## Common bugs / pitfalls

- Polling without backoff. Hammering `/verify` every 100 ms wastes both
  sides; recommended: 1 s for first 5 s, then 2 s, then 5 s up to a 5 min
  cap.
- Trusting `settled: true` without checking the preimage. A misconfigured
  or hostile merchant could lie. Always recompute SHA-256.
- Client confuses HTLC settled (own balance debited) with merchant
  settled (merchant's invoice paid). The first does not imply the second
  if the route was held in the air or routed via a faulty intermediary.
- Race between BOLT12 and LUD-21: BOLT12 has its own proof_of_payment;
  do not mix them. LUD-21 is for LNURL-pay (BOLT11) flows.
- Not supplying `verify` URL when the merchant uses a HODL invoice. With
  HODL the preimage is held until merchant logic releases; the verify
  endpoint is the only way for the client to learn the eventual state.

## References

- LUD-21: https://github.com/lnurl/luds/blob/luds/21.md
- LUD-06: https://github.com/lnurl/luds/blob/luds/06.md
- BOLT 11 payment_hash semantics
- LNbits LUD-21 implementation reference
