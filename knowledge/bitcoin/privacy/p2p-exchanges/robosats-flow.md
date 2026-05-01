# RoboSats Trade Flow - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/privacy/p2p-exchanges`.
> Canonical source: https://learn.robosats.com/docs/protocol-summary/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/privacy/p2p-exchanges/SKILL.md

## Concept

RoboSats is a **Lightning-native P2P fiat <-> BTC exchange** running on Tor
hidden services. Unlike Bisq, all on-chain steps are replaced by **HODL
invoices**: BTC is escrowed via a Lightning HTLC whose preimage is held by
the coordinator until both parties confirm. No on-chain footprint per trade.

The "robot" identity (cute deterministic avatar) is derived from a 32-byte
randomly-generated token. No registration, no email.

## Walkthrough / mechanics

### Roles

- **Maker** posts an order; can be buyer or seller of BTC.
- **Taker** takes the order.
- **Coordinator** runs the matching server (Tor onion). It holds the
  HODL invoice secret but **never custodies funds**.

### HODL invoice mechanics

A Lightning HODL (or "hold") invoice is one where the receiver does not
release the preimage on receipt; payment is **locked** in HTLCs along the
route until receiver settles or cancels (`hold_invoice_settle` /
`hold_invoice_cancel`).

### Trade timeline

1. **Order creation**. Maker posts: side (buy/sell), amount, fiat method,
   premium %, expiry. Generates a 32-byte token; coordinator returns a
   robot avatar bound to it.
2. **Bond posting**. Both maker and taker pay a small **bond HODL**
   (e.g. 3 % of trade) to coordinator. Bond is held, not settled. If
   either cheats, bond is settled (slashed).
3. **Escrow HODL** (seller). Seller's wallet pays a HODL invoice for the
   full BTC amount. Coordinator now holds the in-flight HTLC preimage.
4. **Buyer invoice**. Buyer submits a normal Lightning **payout invoice**
   for the BTC amount (minus coordinator fee, e.g. 0.21 %).
5. **Fiat send + confirm**. Buyer sends fiat off-platform per agreed
   method (SEPA, Revolut tag, voucher). Buyer hits "fiat sent". Seller
   verifies, hits "received".
6. **Settlement**. Coordinator settles seller's HODL (claims BTC via
   preimage), then immediately pays buyer's payout invoice. Both bonds
   are returned (cancelled HODLs).
7. **Dispute path**. If steps 5-6 stall, party raises dispute. Coordinator
   reads chat logs and decides; loser's bond is slashed; payout to winner
   is sent over Lightning.

### Why HODL > on-chain escrow

- No mining-fee per trade; ~10 sat fee per hop.
- Sub-second settlement once both parties confirm.
- Forces a **time bound**: HODL invoices have a CLTV expiry (~24 h).
  If trade stalls past expiry, all HTLCs unwind atomically and bonds are
  refunded. No stuck-money outcome.

## Worked example

Alice sells 0.005 BTC for EUR via Revolut.

| Step | Lightning event |
|------|-----------------|
| 1. Maker bond | Alice pays 1500 sat HODL to coordinator |
| 2. Taker bond | Bob pays 1500 sat HODL to coordinator |
| 3. Escrow | Alice pays 0.005 BTC HODL to coordinator |
| 4. Payout invoice | Bob submits BOLT11 for 0.005 - 0.21 % |
| 5. Fiat send/recv | off-Lightning, in-app chat |
| 6. Settle | Coordinator claims escrow preimage, pays Bob's invoice |
| 7. Bonds released | HODL cancel returns 1500 sat to each |

Total Lightning fees: a few sats. No on-chain footprint.

## Common pitfalls

- **Coordinator outage**: if coordinator dies mid-trade, HTLCs eventually
  expire and HODLs unwind, returning funds. But the trade is dead until
  next launch and the chat log may be lost.
- **Privacy from coordinator**: coordinator sees onion connection, robot
  token, IP via Tor circuit (limited), Lightning node pubkey of payouts.
  A privacy-conscious user uses a fresh Lightning channel + LNP via Tor.
- **Buyer payment-method linkability**: same risk as Bisq; the off-chain
  fiat path leaks identity. Use voucher / cash mailers if required.
- **Bond too low**: 3 % may be insufficient deterrent for high-value
  trades; coordinator can require larger bonds for big orders.
- **Lightning failure modes**: route-not-found means the seller cannot
  pay the escrow HODL. RoboSats prompts to re-pay; coordinator-side
  amount tolerance is 100 ppm.

## References

- RoboSats protocol-summary (learn.robosats.com).
- BOLT11 invoice spec (HODL via `c=` and `cltv` fields handled by node).
- LND `invoicesrpc` HoldInvoice API.
