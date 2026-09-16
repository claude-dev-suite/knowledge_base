# BLIP-50 LSP Transport and Order Flow - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/lsp`.
> Canonical source: BLIP-50 (https://github.com/lightning/blips/blob/master/blip-0050.md)
> Spec surface checked against blips master as of September 2026 (BLIP-50/51).
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/lsp/SKILL.md

## Concept

BLIP-50 is **LSPS0: LSP Spec Transport Layer**. It is not an API surface
of its own: it standardizes how a wallet and an LSP talk, so wallets can
shop across providers without bespoke integrations. The transport is
JSON-RPC 2.0 spoken over **BOLT 8 peer-to-peer messages with message ID
37913** - not HTTPS, and with no API key. The client is identified and
authenticated by the node id of the BOLT 8 tunnel it is already using.

BLIP-50 defines exactly one method, `lsps0.list_protocols`, which takes
no parameters and returns the LSPS specification numbers the LSP
implements. The discovery, ordering, and payment-tracking methods shown
below are **BLIP-51 (LSPS1, Channel Requests)** riding on that transport;
BLIP-52 (LSPS2, JIT) and BLIP-55 (LSPS5, Webhook Registration) ride on it
too.

## Walkthrough / mechanics

### lsps0.list_protocols

Request payload inside a BOLT 8 message ID 37913:

```json
{
  "jsonrpc": "2.0",
  "id": "example#3cad6a54d302edba4c9ade2f7ffac098",
  "method": "lsps0.list_protocols",
  "params": {}
}
```

Result: `{ "protocols": [1, 3] }` - blip-0050's own non-normative
example, glossed there as "indicating it supports LSPS1 and LSPS3".
Treat the numbers as illustrative: no LSPS3/BLIP-53 exists in the blips
master tree as of September 2026, so a real answer is drawn from
{1, 2, 5}. LSPS0 itself is never listed; its support is advertised by
`features` bit 729 (`option_supports_lsps`) in gossip instead.

The three LSPS1 order methods, all JSON-RPC over the same transport:

```
lsps1.get_info
lsps1.create_order
lsps1.get_order
```

Amounts are encoded per LSPS0 as JSON **strings** of decimal integers
(`"5000000"`), never JSON numbers - a 64-bit msat value does not survive
an IEEE 754 round-trip.

### lsps1.get_info

Request: no parameters.

Response (excerpt):

```json
{
  "min_required_channel_confirmations": 0,
  "min_funding_confirms_within_blocks": 6,
  "supports_zero_channel_reserve": true,
  "max_channel_expiry_blocks": 20160,
  "min_initial_client_balance_sat": "0",
  "max_initial_client_balance_sat": "100000000",
  "min_initial_lsp_balance_sat": "50000",
  "max_initial_lsp_balance_sat": "500000000",
  "min_channel_balance_sat": "50000",
  "max_channel_balance_sat": "500000000"
}
```

The wallet uses these limits to size a feasible order. The LSP SHOULD NOT
change these values more than once per day, so explorers can scrape them.

### lsps1.create_order

Request:

```json
{
  "lsp_balance_sat": "1000000",
  "client_balance_sat": "0",
  "required_channel_confirmations": 0,
  "funding_confirms_within_blocks": 6,
  "channel_expiry_blocks": 6048,
  "token": "<optional coupon or auth string>",
  "refund_onchain_address": "bc1q...",
  "announce_channel": false
}
```

Response:

```json
{
  "order_id": "bb4b5d0a-8334-49d8-9463-90a6d413af7c",
  "lsp_balance_sat": "1000000",
  "client_balance_sat": "0",
  "channel_expiry_blocks": 6048,
  "created_at": "2026-04-29T09:30:00.000Z",
  "announce_channel": false,
  "order_state": "CREATED",
  "payment": {
    "bolt11": {
      "state": "EXPECT_PAYMENT",
      "expires_at": "2026-04-29T10:00:00.000Z",
      "fee_total_sat": "5000",
      "order_total_sat": "5000",
      "invoice": "lnbc50u1pr..."
    },
    "onchain": {
      "state": "EXPECT_PAYMENT",
      "expires_at": "2026-04-29T10:00:00.000Z",
      "fee_total_sat": "5500",
      "order_total_sat": "5500",
      "address": "bc1q...",
      "min_onchain_payment_confirmations": 1,
      "min_fee_for_0conf": 253,
      "refund_onchain_address": "bc1q..."
    }
  },
  "channel": null
}
```

The `payment` object is keyed by payment option (`bolt11`, `bolt12`,
`onchain`); the LSP MAY omit any of them and each carries its own fee and
expiry. The wallet pays one of them. The BOLT 11 `invoice` MUST be a HOLD
invoice, which is what lets the LSP refund automatically if the open
fails.

### lsps1.get_order

Takes `{ "order_id": ... }` and returns the same object as
`lsps1.create_order`. Two independent state machines are being tracked.

`order_state` has only three values:

```
CREATED -> COMPLETED   (LSP published the funding transaction)
        \-> FAILED     (order failed; LSP MUST issue a refund)
```

`payment.<option>.state` is where the progress actually shows. For the
Lightning options (`bolt11`, `bolt12`):

```
EXPECT_PAYMENT -> HOLD -> PAID
                       \-> REFUNDED
```

`HOLD` means the Lightning payment arrived but the preimage is not
released; the LSP opens the channel from `HOLD` and only releases the
preimage (moving to `PAID`) once the open succeeded. (`CANCELLED` is a
deprecated spelling of `REFUNDED`; clients should still accept it.)

The `onchain` option has no `HOLD` state (BLIP-51 section 3.3): it goes
`EXPECT_PAYMENT -> PAID` once the payment confirms, or `-> REFUNDED`. The
LSP opens the channel when `payment.state` is `HOLD` (Lightning) or
`PAID` (on-chain).

Once `order_state` is `COMPLETED`, the response includes:

```json
"channel": {
  "funded_at": "2026-04-29T10:01:23.000Z",
  "funding_outpoint": "<txid>:<vout>",
  "expires_at": "2026-09-15T..."
}
```

`expires_at` here is the earliest datetime at which the LSP MAY close the
channel, derived from `channel_expiry_blocks`.

## Worked example

```
Wallet connects to LSP over BOLT 8, then sends msg 37913:
Wallet -> LSP: { "method": "lsps0.list_protocols", "params": {} }
LSP   -> Wallet: { "protocols": [1, 2] }

Wallet -> LSP: { "method": "lsps1.get_info", "params": {} }
LSP   -> Wallet: { "min_initial_lsp_balance_sat": "50000", ... }

Wallet picks 1M sat inbound capacity.

Wallet -> LSP: lsps1.create_order
  { "lsp_balance_sat": "1000000", "client_balance_sat": "0", ... }
LSP   -> Wallet: { "order_id": "abc", "order_state": "CREATED",
                   "payment": { "bolt11": { "fee_total_sat": "5000",
                     "invoice": "lnbc50u..." } } }

Wallet pays the BOLT11 HOLD invoice (5000 sat).

Wallet polls lsps1.get_order { "order_id": "abc" }:
  T+0s:  order_state = "CREATED",   payment.bolt11.state = "EXPECT_PAYMENT"
  T+3s:  order_state = "CREATED",   payment.bolt11.state = "HOLD"
  T+30s: order_state = "COMPLETED", payment.bolt11.state = "PAID",
         channel.funding_outpoint = "abcdef:0"

Wallet now sees an inbound channel and can receive.
```

## Common bugs / pitfalls

- Confusing `lsp_balance_sat` (inbound liquidity for client) with channel
  size. The channel size is `lsp_balance_sat + client_balance_sat`.
- Submitting `client_balance_sat > 0` (push) when LSP doesn't support push.
  Always check `min_initial_client_balance_sat` from `lsps1.get_info`.
- Polling `lsps1.get_order` more than once per second wastes both sides;
  recommended: 1 s for first 30 s, 5 s thereafter, give up after 30 min.
- Treating `funding_confirms_within_blocks` as a guarantee. It is the LSP's
  best effort - it exists so the LSP can batch several channel opens into
  one funding transaction. Wallet should still tolerate longer delays.
- Skipping `refund_onchain_address`. If channel open fails after on-chain
  payment, the LSP needs an address to refund - and the LSP MUST disable
  on-chain payment entirely when the client omits it.
- Watching only `order_state` and concluding nothing is happening. It sits
  at `CREATED` for the whole payment phase; `payment.<option>.state` is the
  field that moves.
- Sending amounts as JSON numbers. LSPS0 requires decimal strings for every
  `_sat`/`_msat` field; a JSON parser that widens to IEEE 754 silently
  corrupts large msat values.
- Batching several JSON-RPC requests into one array. LSPS0 forbids it, and
  the BOLT 8 payload cap of 65533 bytes applies to the whole message.
- Reusing or incrementing the JSON-RPC `id`. LSPS0 requires a high-entropy
  `id` string per request, and responses with an unknown `id` are ignored.
- Assuming the request survives a disconnect. There is no HTTP session to
  resume - the wallet must be a connected BOLT 8 peer of the LSP for the
  duration, and re-query `lsps1.get_order` after reconnecting.
- TLS pinning to a single LSP cert. Over BOLT 8 there is no certificate at
  all - the node id is the identity - but some LSPs still expose LSPS1 as
  an HTTPS REST endpoint instead (Megalith, whose own docs warn "our
  current implementation uses HTTP communication, instead of BOLT8",
  September 2026). For those, do not pin one cert: the wallet breaks when
  the LSP rotates it. Prefer Web PKI with proper revocation handling.

## References

- BLIP-50 (LSPS0 Transport): https://github.com/lightning/blips/blob/master/blip-0050.md
- BLIP-51 (LSPS1 Channel Requests): https://github.com/lightning/blips/blob/master/blip-0051.md
- BLIP-52 (LSPS2 JIT Channel Negotiation): https://github.com/lightning/blips/blob/master/blip-0052.md
- BLIP-55 (LSPS5 Webhook Registration): https://github.com/lightning/blips/blob/master/blip-0055.md
- `lightning-liquidity` reference implementation (rust-lightning):
  https://github.com/lightningdevkit/rust-lightning/tree/main/lightning-liquidity
- Olympus by ZEUS public LSPS1 API (checked September 2026):
  https://lsps1.lnolymp.us/api/v1/get_info - Voltage's Flow 2.0 LSP is
  retired and Voltage no longer documents an LSP product
- Megalith LSPS1 docs and Flashsats integration guide:
  https://docs.megalithic.me/lightning-services/lsp1-get-inbound-liquidity-for-mobile-clients/
  and https://lsp.flashsats.xyz/api/v1/get_info
