# MPP Flow Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/amp-mpp`.
> Canonical source: https://github.com/lightning/bolts/blob/master/04-onion-routing.md (basic_mpp)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/amp-mpp/SKILL.md

## Concept

Multi-Part Payment (MPP) lets a Lightning sender split one logical
payment across multiple HTLC-routed parts. The recipient holds incoming
HTLCs until total received >= invoice amount, then settles all with the
shared preimage. MPP is the canonical solution to "I want to send 1 BTC
but no single channel has 1 BTC of capacity".

## Walkthrough / mechanics

### BOLT-11 invoice features

The recipient signals MPP support via feature bit 17 (basic_mpp) in
their invoice. The invoice also includes:

- `payment_hash` (H)
- `payment_secret` (S, 32 bytes, anti-probing) — feature bit 15.
- `min_final_cltv_expiry`

### Sender split

Sender chooses K parts summing to invoice amount. Each part has its
own outer onion, but identical:

- `payment_hash`
- `payment_secret`
- `total_msat`

Per-part `amount_msat` differs; sum equals `total_msat`.

### HTLC payload at recipient

Final hop TLV payload:

| TLV type | Field |
|----------|-------|
| 2 | amt_to_forward (this part's value) |
| 4 | outgoing_cltv_value |
| 8 | payment_data (TLV containing payment_secret + total_msat) |

When recipient sees the HTLC, they verify:

1. payment_secret matches their invoice's S.
2. payment_hash matches.
3. amount_msat <= total_msat (rejects overpayment).

### Wait-and-settle state machine

Recipient adds HTLC to a "pending MPP set" keyed by payment_hash.

```
On HTLC arrival:
  if payment_secret invalid -> fail HTLC immediately.
  if total_msat conflicts with prior parts -> fail this HTLC.
  add to set.
  if sum(set) >= total_msat:
    reveal preimage to all parts atomically.
    fulfilled.
On timeout (60-90s typical):
  fail all parts in set.
```

### Atomicity

Recipient MUST NOT settle a subset of parts. Either all settle or all
fail. This atomicity is enforced by the recipient's wait-and-settle
logic: settle = reveal preimage to ALL pending parts simultaneously,
or none.

### Sender retry

If part N fails, sender can retry that part (potentially via different
route). The retry uses the same `payment_secret` + `payment_hash` so
recipient correctly aggregates. Sender must avoid retrying succeeded
parts (preimage already revealed).

### Overpayment protection

The `total_msat` field caps total. If sender sends parts summing to
> total_msat, recipient rejects extras. If <, recipient eventually
times out.

## Worked example

Alice pays Bob 100 000 sat invoice.

```
Invoice features: basic_mpp=true, payment_secret=S, payment_hash=H
total_msat = 100,000,000 msat

Alice splits:
  Part 1: 40,000,000 msat via path P1
  Part 2: 35,000,000 msat via path P2
  Part 3: 25,000,000 msat via path P3

All parts' final hop payload contains:
  payment_secret = S
  total_msat     = 100,000,000

Bob receives:
  T+0.5s: Part 1 arrives, sum=40M, hold
  T+1.2s: Part 2 arrives, sum=75M, hold
  T+2.0s: Part 3 arrives, sum=100M, REACHED.
Bob reveals preimage P; all 3 HTLCs settle simultaneously.
```

If Part 3 fails on a route:

```
Alice gets HTLC failure for part 3.
Alice retries part 3 via different path P3'.
T+5s: retry succeeds, sum=100M, Bob reveals.
```

## Common pitfalls

- **payment_secret missing**: pre-MPP invoices lacked this; receiver
  cannot distinguish two different payers' parts. Always require
  feature 15.
- **total_msat mismatch across parts**: one buggy hop modifies the
  field; recipient rejects entire payment. Sender must construct
  identical TLV across parts.
- **Recipient timeout too short**: high-latency paths may exceed
  default 60s; receiver MUST configure to match expected MPP
  reassembly window.
- **Stuck HTLCs**: if part is in-flight and can't fail (e.g. peer
  disconnect), recipient's MPP set holds incomplete sum until CLTV
  expiry (~24h); HTLC may force-close channels.
- **MPP probing**: probing without payment_secret is impossible (recipient
  rejects). Useful security property.

## References

- BOLT-04 § "Basic Multi-Path Payments".
- BOLT-11 feature bits 15 (payment_secret) and 17 (basic_mpp).
- Eclair / LND / CLN MPP implementations.
