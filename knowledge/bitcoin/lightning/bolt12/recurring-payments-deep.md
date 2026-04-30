# BOLT12 Recurring Payments - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/bolt12`.
> Canonical source: BOLT 12 recurrence TLVs (https://github.com/lightning/bolts/pull/798)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/bolt12/SKILL.md

## Concept

Recurring payments in BOLT12 use a single offer that defines a recurrence
schedule (period, count, optional base time). A payer's wallet treats the
offer as a subscription and pulls a fresh invoice on each period boundary.
There is no on-protocol "auto-debit" - the payer keeps control and may
abort at any period without notifying the merchant.

## Walkthrough / mechanics

The offer carries an `offer_recurrence` TLV with:

```
recurrence:
    time_unit:    1 byte (0=seconds 1=days 2=months 3=years)
    period:       u32 number of units
    paywindow:    optional u32 - seconds before/after period boundary
    limit:        optional u32 - max periods (open-ended if absent)
    base_time:    optional u64 - unix seconds anchor
```

`offer_recurrence_paywindow` defines the grace window during which a payer
may pay for a given period. Outside the window, the merchant rejects the
invoice request with `error: payment outside paywindow`.

Each `invoice_request` MUST include `invreq_recurrence_counter` (the period
index, 0 for first) and optionally `invreq_recurrence_start` if the offer has
no `base_time`. The merchant's invoice carries `invoice_recurrence_basetime`
to lock in the anchor for the payer's future requests.

Payer's wallet logic per period:

```
while subscription_active:
  now = current_time()
  period_idx = (now - basetime) / period_seconds
  if last_paid_period < period_idx and now in paywindow(period_idx):
    inv = fetch_invoice(offer, recurrence_counter=period_idx)
    pay(inv)
    last_paid_period = period_idx
  sleep(check_interval)
```

## Worked example

Coffee subscription, 1000 sat/month for 12 months, 7-day paywindow:

```
offer_amount: 1000000 (msat)
offer_description: "Daily Brew - monthly"
offer_recurrence:
  time_unit: 2 (months)
  period: 1
  paywindow: 604800 (7 days)
  limit: 12
offer_node_id: <merchant>
```

Period 0 invoice request (Jan 5):

```
invreq_payer_id: <random_per_payer>
invreq_amount: 1000000
invreq_recurrence_counter: 0
invreq_recurrence_start: 0
invreq_payer_note: "thanks!"
```

Merchant returns invoice with `invoice_recurrence_basetime = 1704412800`
(Jan 5 unix seconds). Payer pays. On Feb 5, wallet auto-fetches with
`invreq_recurrence_counter = 1`. If wallet is offline until Feb 13, it is
within paywindow (Feb 5 + 7 days = Feb 12) - request might fail by 1 day -
this is a common pitfall, see below.

## Common bugs / pitfalls

- `paywindow` measured from period start; off-by-one when period_idx changes
  near midnight UTC vs local time. Always use UTC seconds.
- Payer wallet retry storms: device offline for weeks, wakes up, tries to
  back-pay 5 missed periods. Merchants typically reject historic counters.
  Recommended: enforce `recurrence_counter == current_period_idx` only.
- `invreq_payer_id` re-use across multiple offers links subscriptions. Use a
  fresh blinded payer key per offer.
- LDK partial-recurrence support means you cannot rely on automatic period
  rollover in LDK <= 0.0.130. Implement period-tracking in the integrating
  wallet code.
- A subscription cannot be "cancelled" cryptographically; the merchant has
  no on-protocol cancel signal. The merchant must time out subscriptions
  if a counter is not paid within `paywindow + grace`.

## References

- BOLT12 recurrence draft PR: https://github.com/lightning/bolts/pull/798
- Rusty Russell "Why BOLT12" blog post
- CLN `lightning-fetchinvoicerecurrence` (where supported)
- Phoenix recurring-offer experimental UX
