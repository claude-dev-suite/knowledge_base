# BLIP-50 LSP API Flow - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/lsp`.
> Canonical source: BLIP-50 (https://github.com/lightning/blips/blob/master/blip-0050.md)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/lsp/SKILL.md

## Concept

BLIP-50 standardizes a JSON-RPC-over-HTTPS API surface that any LSP can
implement so wallets can shop across providers without bespoke integrations.
It defines the discovery, ordering, and payment-tracking endpoints for a
liquidity service. BLIP-50 itself is generic; specific channel-purchase
flows (BLIP-51 paid channels, BLIP-52 JIT) layer on top.

## Walkthrough / mechanics

Three core methods, all `POST` with JSON body:

```
POST /api/v1/get_info
POST /api/v1/create_order
POST /api/v1/get_order
```

### get_info

Request: `{}` or empty.

Response (excerpt):

```json
{
  "options": {
    "min_required_channel_confirmations": 0,
    "min_funding_confirms_within_blocks": 6,
    "supports_zero_channel_reserve": true,
    "max_channel_expiry_blocks": 25920,
    "min_initial_client_balance_sat": 0,
    "max_initial_client_balance_sat": 100000000,
    "min_initial_lsp_balance_sat": 50000,
    "max_initial_lsp_balance_sat": 500000000,
    "min_channel_balance_sat": 50000,
    "max_channel_balance_sat": 500000000
  }
}
```

The wallet uses these limits to size a feasible order.

### create_order

Request:

```json
{
  "lsp_balance_sat": 1000000,
  "client_balance_sat": 0,
  "required_channel_confirmations": 0,
  "funding_confirms_within_blocks": 6,
  "channel_expiry_blocks": 6048,
  "token": "<optional auth>",
  "refund_onchain_address": "bc1q..."
}
```

Response:

```json
{
  "order_id": "bb4567cd-...",
  "lsp_balance_sat": 1000000,
  "client_balance_sat": 0,
  "channel_expiry_blocks": 6048,
  "order_state": "CREATED",
  "payment": {
    "state": "EXPECT_PAYMENT",
    "fee_total_sat": 5000,
    "order_total_sat": 5000,
    "bolt11_invoice": "lnbc50u1pr...",
    "onchain_address": "bc1q...",
    "min_onchain_payment_confirmations": 1,
    "min_fee_for_0conf": 5,
    "expires_at": "2026-04-29T10:00:00Z"
  },
  "channel": null
}
```

The wallet pays `bolt11_invoice` (or sends on-chain). The LSP polls
the payment, then opens the channel.

### get_order

Polled by wallet to track state transitions:

```
CREATED -> EXPECT_PAYMENT -> PAID -> CHANNEL_OPENING -> COMPLETED
                                  \-> FAILED (with reason)
```

Once `COMPLETED`, the response includes:

```json
"channel": {
  "funded_at": "2026-04-29T10:01:23Z",
  "funding_outpoint": "<txid>:<vout>",
  "expires_at": "2026-09-15T..."
}
```

## Worked example

```
Wallet -> LSP: POST /api/v1/get_info
LSP   -> Wallet: { options:{...} }

Wallet picks 1M sat inbound capacity.

Wallet -> LSP: POST /api/v1/create_order
  { "lsp_balance_sat": 1000000, "client_balance_sat": 0, ... }
LSP   -> Wallet: { "order_id": "abc", "fee_total_sat": 5000,
                   "bolt11_invoice": "lnbc50u..." }

Wallet pays the BOLT11 invoice (5000 sat).

Wallet polls /api/v1/get_order { "order_id":"abc" }:
  T+0s:  state = "EXPECT_PAYMENT"
  T+3s:  state = "PAID"
  T+8s:  state = "CHANNEL_OPENING"
  T+30s: state = "COMPLETED",
         channel.funding_outpoint = "abcdef:0"

Wallet now sees an inbound channel and can receive.
```

## Common bugs / pitfalls

- Confusing `lsp_balance_sat` (inbound liquidity for client) with channel
  size. The channel size is `lsp_balance_sat + client_balance_sat`.
- Submitting `client_balance_sat > 0` (push) when LSP doesn't support push.
  Always check `min_initial_client_balance_sat` from `get_info`.
- Polling `get_order` more than once per second wastes both sides;
  recommended: 1 s for first 30 s, 5 s thereafter, give up after 30 min.
- Treating `funding_confirms_within_blocks` as a guarantee. It is the LSP's
  best effort. Wallet should still tolerate longer delays.
- Skipping `refund_onchain_address`. If channel open fails after on-chain
  payment, the LSP needs an address to refund.
- TLS pinning to a single LSP cert: when LSP rotates certs, wallet breaks.
  Prefer Web PKI with proper revocation handling.

## References

- BLIP-50: https://github.com/lightning/blips/blob/master/blip-0050.md
- BLIP-51 Paid Channel Open: https://github.com/lightning/blips/blob/master/blip-0051.md
- Olympus and Voltage public LSP API docs
- Megalith and Flashsats integration guides
