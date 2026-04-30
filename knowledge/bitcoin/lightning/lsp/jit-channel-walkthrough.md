# JIT Channel Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/lsp`.
> Canonical source: BLIP-52 (https://github.com/lightning/blips/blob/master/blip-0052.md)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/lsp/SKILL.md

## Concept

A JIT (Just-In-Time) channel is opened **during** an inbound payment. The
client has zero existing inbound capacity; the LSP intercepts the incoming
HTLC, opens a channel to the client funded by the payment, and forwards
the remainder via the new channel as the final hop. The client receives
their invoice amount minus the LSP fee. BLIP-52 formalizes the API and the
hop-hint construction.

## Walkthrough / mechanics

Setup phase (one-time per invoice):

1. Wallet creates a temporary node id (or uses a persistent one) and
   asks the LSP for a JIT-capable invoice template via BLIP-52
   `lsps2.get_versions` then `lsps2.get_info`.
2. LSP returns supported feature flags and fee parameters
   (e.g. `min_payment_size_msat`, `max_payment_size_msat`,
   `proportional`, `min_lifetime`, `max_client_to_self_delay`).
3. Wallet asks `lsps2.buy` with desired params. LSP returns an
   `intercept_scid` (ephemeral short channel id) and a `cltv_expiry_delta`.
4. Wallet builds the BOLT11 invoice with a `route_hint` containing
   the LSP's pubkey + `intercept_scid`. The amount and payment_hash are
   real; the SCID is fake (LSP will recognize it).

Payment phase:

5. Payer pays the invoice. The HTLC reaches the LSP. The LSP sees the
   `intercept_scid` and recognizes "this needs JIT".
6. LSP holds the HTLC, then opens a 0-conf channel to the client of size
   `payment_amount - lsp_fee` (or larger, see fee modes).
7. LSP forwards the (now smaller) HTLC over the freshly opened channel
   as the final hop.
8. Client wallet receives the HTLC, releases preimage, settles. LSP
   collects preimage, settles upstream, claims its fee.

Trust note: the channel is 0-conf. Between LSP receiving the upstream
HTLC and the funding tx confirming, the client is trusting the LSP not
to keep the funds and refuse to broadcast. Mitigated by reputation +
small per-payment amounts.

## Worked example

```
Wallet -> LSP (BLIP-52): lsps2.get_versions -> [1]
Wallet -> LSP: lsps2.get_info { "version": 1 }
LSP   -> Wallet:
  { "opening_fee_params_menu": [
      { "min_fee_msat": 5000000,
        "proportional": 1000,    // 0.1% in ppm
        "valid_until": "2026-04-29T11:00:00Z",
        "min_lifetime": 4032,
        "max_client_to_self_delay": 2016,
        "min_payment_size_msat": 100000,
        "max_payment_size_msat": 100000000,
        "promise": "<HMAC-signed by LSP>" }
    ],
    "min_payment_size_msat": 100000, ... }

Wallet picks a fee_params item, calls:
Wallet -> LSP: lsps2.buy {
  "opening_fee_params": <selected_item_with_promise>,
  "payment_size_msat": 50000000   // 50k sat invoice
}
LSP   -> Wallet: {
  "jit_channel_scid": "65535x65535x1",
  "lsp_cltv_expiry_delta": 144,
  "client_trusts_lsp": true
}

Wallet builds BOLT11:
  amount = 50000 sat
  payment_hash = <random>
  routing_hints = [(lsp_node_id, "65535x65535x1", base_fee=0,
                    proportional=0, cltv_delta=144)]

Payer pays. Timeline:
  T+0:    HTLC arrives at LSP with payhash X, target SCID 65535x65535x1.
  T+0:    LSP holds HTLC, signs commit txn for new 0-conf channel with client.
  T+1s:   funding tx broadcast, LSP forwards 49.995k sat HTLC down new channel.
  T+1.2s: client wallet sees HTLC, releases preimage.
  T+1.3s: LSP claims preimage upstream, fees collected.
```

## Common bugs / pitfalls

- Fee params menu has expired (`valid_until` past). LSP rejects buy. Always
  re-call `get_info` if more than ~5 min elapsed since selection.
- Invoice amount below `min_payment_size_msat`. LSP refuses to allocate
  channel. Communicate the minimum to the user before invoice creation.
- Wallet forgot to add the routing hint. Payer routes via public graph,
  finds no path to wallet (wallet has no public channels), payment fails.
- `client_trusts_lsp = false` mode (BLIP-52 v2) requires client to verify
  the funding tx out before releasing preimage. Many wallet impls only
  support v1 + trust mode.
- HTLC slot exhaustion on LSP side under burst traffic; some LSPs queue
  buys and drop excess.
- Channel reserve interplay: LSP enforces `min_initial_client_balance_sat`;
  the JIT-credited amount counts toward client balance, but reserve maths
  can leave the channel unusable if the payment is at the lower bound.

## References

- BLIP-52: https://github.com/lightning/blips/blob/master/blip-0052.md
- ACINQ Phoenix splice-on-demand technical write-up
- Voltage Olympus JIT documentation
- Breez SDK JIT integration source
