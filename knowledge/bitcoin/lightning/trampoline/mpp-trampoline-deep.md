# MPP Trampoline Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/trampoline`.
> Canonical source: https://github.com/lightning/bolts/blob/master/proposals/trampoline.md (multi-part section)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/trampoline/SKILL.md

## Concept

Multi-Path Payments (MPP) split a Lightning payment across multiple
parallel routes to overcome channel-balance and capacity constraints.
Combining MPP with **Trampoline routing** lets a mobile wallet split a
large payment across multiple trampoline-onion subpayments,
reaggregating at the recipient. This is essential for paying invoices
greater than the wallet's max channel balance to a single peer.

## Walkthrough / mechanics

### Two distinct MPP scopes

1. **Sender MPP**: sender splits the total payment into K parts and
   sends each via a (possibly different) trampoline path.
2. **Trampoline-internal MPP**: a single trampoline may further split
   its outgoing routes when forwarding to the next trampoline.

### Sender-side MPP with Trampoline

The sender constructs K outer onions, each containing a trampoline
onion specifying:

```
trampoline_onion (final hop):
  amount_to_forward = part_msat_i
  total_msat        = total_msat (same across all K)
  payment_hash      = H (same across all K)
  payment_metadata  = M
  payment_secret    = S (BOLT-11 PaySecret feature)
```

Each part has unique amount but shared `total_msat`, `payment_secret`,
`payment_hash`. Recipient (or final trampoline) reassembles when
`sum(amount_to_forward) >= total_msat`.

### Trampoline forwarding for MPP

Trampoline T1 receives an HTLC with `amount_to_forward = X`,
`total_msat = Y` for `H`. It forwards to T2:

- Option A: forward as a single HTLC with the same amount.
- Option B: split into smaller HTLCs across multiple outer onion paths
  to T2.

T2 sees multiple HTLCs with same `H`, accumulates until total reaches
the next trampoline's amount.

### Final reassembly

The recipient (or final trampoline) holds incoming HTLCs in `WAIT_MPP`
state. Once `sum(amount) >= total_msat` and within a configurable
timeout (~60 s), the recipient settles all HTLCs by revealing preimage.
If timeout hits without enough HTLCs, all are failed.

### Atomicity

MPP is atomic: either all parts succeed (payee gets total) or none
(refunded). Recipient must NOT settle individual HTLCs before
sufficient total is received.

### Fees and timing

Each part incurs separate routing fees. Sender pays sum-of-fees + base
fee per trampoline + outer routing fees. CLTV expiry differs per part;
the recipient must set timeout > maximum CLTV expiry.

## Worked example

Alice pays Bob 100 000 sat via Phoenix wallet.

```
Phoenix splits into 4 parts:
  part 1: 30 000 sat via T1=ACINQ_A trampoline -> Bob
  part 2: 25 000 sat via T1=ACINQ_A trampoline -> Bob
  part 3: 25 000 sat via T1=ACINQ_B trampoline -> Bob
  part 4: 20 000 sat via T1=ACINQ_A trampoline -> Bob

Each part's trampoline onion final hop:
  amount_to_forward = part_size
  total_msat = 100 000 * 1000 = 100,000,000 msat
  payment_hash = H
  payment_secret = S

Each trampoline (potentially with internal MPP) forwards onward.
Eventually Bob's node sees 4 HTLCs accumulating to 100 000 sat.
Bob settles all 4 with preimage P.
```

## Common pitfalls

- **Total_msat drift**: parts must declare the same total_msat. Bug in
  one part's payload causes recipient to wait forever.
- **payment_secret reuse**: must be the same across parts for the same
  invoice, but never reused across invoices.
- **Trampoline-internal vs. sender MPP**: a trampoline can additionally
  split an incoming part into outgoing sub-parts; reassembly point is
  the next trampoline OR final recipient.
- **CLTV variance across parts**: longest-CLTV part dictates payment
  timeout. Unevenly-routed parts may stall.
- **MPP failure cleanup**: if 2 of 4 parts succeed and 2 fail, sender
  must NOT retry indefinitely; failed parts may be replayed at
  unexpectedly low total. Use payment_secret + total_msat to bind.
- **Feature negotiation**: both sender and recipient must support
  feature bit `basic_mpp` (BOLT11 var_addr_payee feature 17). Older
  recipients reject.

## References

- BOLT-04 Multi-Part Payments section.
- BOLT-11 feature bit 17 (basic_mpp).
- Trampoline proposal § "Multi-part payments through trampoline".
