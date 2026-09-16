# JIT Channel Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/lsp`.
> Canonical source: BLIP-52 (https://github.com/lightning/blips/blob/master/blip-0052.md)
> Spec surface checked against blips master as of September 2026 (BLIP-52).
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

1. Wallet connects to the LSP over BOLT 8 and calls BLIP-52
   `lsps2.get_info` (optionally with a `token` discount/API string)
   over the LSPS0 transport. There is no version-negotiation call;
   `lsps0.list_protocols` is what tells the wallet LSPS2 is supported.
2. LSP returns `opening_fee_params_menu`, an array of fee-parameter
   objects (`min_fee_msat`, `proportional`, `valid_until`,
   `min_lifetime`, `max_client_to_self_delay`, `min_payment_size_msat`,
   `max_payment_size_msat`, `promise`).
3. Wallet asks `lsps2.buy` with a verbatim copy of one menu entry plus
   an optional `payment_size_msat`. LSP returns a `jit_channel_scid`
   (ephemeral short channel id) and an `lsp_cltv_expiry_delta`.
4. Wallet builds the BOLT11 invoice with a `route_hint` containing
   the LSP's pubkey + `jit_channel_scid`. The amount and payment_hash are
   real; the SCID is fake (LSP will recognize it).

Payment phase:

5. Payer pays the invoice. The HTLC reaches the LSP. The LSP sees the
   `jit_channel_scid` and recognizes "this needs JIT".
6. LSP holds the HTLC, then opens a 0-conf channel to the client of size
   `payment_amount - lsp_fee` (or larger, see fee modes).
7. LSP forwards the (now smaller) HTLC over the freshly opened channel
   as the final hop.
8. Client wallet receives the HTLC, releases preimage, settles. LSP
   collects preimage, settles upstream, claims its fee.

Trust note: BLIP-52 defines two trust models, and the LSP signals which
one applies via `client_trusts_lsp` in the `lsps2.buy` result (optional,
defaults to `false`, and the LSP SHOULD default to it).

- `false` - "LSP trusts client". The LSP broadcasts the funding tx
  first, so the client MAY hold the preimage until it has seen that
  funding output broadcast, or even confirmed, and checked its script
  and value. The client need not trust the LSP; it pays in latency.
- `true` - "client trusts LSP". The client MUST release the preimage as
  soon as the incoming HTLC is irrevocably committed. The LSP may then
  name a funding txid it never broadcasts, and the client has no
  on-chain recourse. A client that does not want to trust any LSP can
  simply refuse LSPs that signal `true`.

If neither side is willing to trust the other they deadlock: the LSP
withholds the funding tx until it sees the preimage, the client
withholds the preimage until it sees the funding tx.

## Worked example

```
Wallet -> LSP (BLIP-52 over LSPS0/BOLT 8 msg 37913):
  lsps2.get_info { }
LSP   -> Wallet:
  { "opening_fee_params_menu": [
      { "min_fee_msat": "5000000",
        "proportional": 1000,    // 0.1% in ppm
        "valid_until": "2026-04-29T11:00:00.000Z",
        "min_lifetime": 4032,
        "max_client_to_self_delay": 2016,
        "min_payment_size_msat": "100000",
        "max_payment_size_msat": "100000000",
        "promise": "<LSP-generated proof of these params>" }
    ] }

Wallet copies one menu item verbatim, calls:
Wallet -> LSP: lsps2.buy {
  "opening_fee_params": <selected_item_with_promise>,
  "payment_size_msat": "50000000"   // 50k sat invoice
}
LSP   -> Wallet: {
  "jit_channel_scid": "65535x65535x1",
  "lsp_cltv_expiry_delta": 144,
  "client_trusts_lsp": false
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
- Treating `client_trusts_lsp` as informational. There is no versioned
  fallback in the spec: when the LSP signals `true`, the client MUST send
  the preimage on irrevocable commitment and has no recourse if the
  funding tx never appears. Either refuse such LSPs, or stay on the
  default `false` model and wait for the funding output.
- Mutating `opening_fee_params` before calling `lsps2.buy`. The client
  MUST copy a menu entry verbatim; the `promise` is what proves the LSP
  offered exactly those params, and any edit fails as
  `invalid_opening_fee_params` (201).
- HTLC slot exhaustion on LSP side under burst traffic; some LSPs queue
  buys and drop excess.
- Channel reserve interplay: LSP enforces `min_initial_client_balance_sat`;
  the JIT-credited amount counts toward client balance, but reserve maths
  can leave the channel unusable if the payment is at the lower bound.

## References

- BLIP-52: https://github.com/lightning/blips/blob/master/blip-0052.md
- ACINQ Phoenix splice-on-demand technical write-up
- Olympus by ZEUS, the JIT/LSPS1 LSP behind the ZEUS wallet (checked
  September 2026): https://lsps1.lnolymp.us/api/v1/get_info - note it
  is run by ZEUS, not ZBD, and is unrelated to Voltage's retired Flow
  2.0 LSP
- Breez SDK JIT integration source
