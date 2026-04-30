# Blinded Paths - Walkthrough

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/routing`.
> Canonical source: BOLT 4 "Route Blinding", `option_route_blinding` (bits 24/25)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/routing/SKILL.md

## Concept

Traditional Lightning routing requires the sender to know the full
path to the destination. Blinded paths invert this: the receiver
publishes a *blinded path* (a sequence of nodes wrapped in
encrypted-then-encrypted envelopes) and any sender can pay it without
learning the destination, intermediate nodes, or even the path
length. Blinded paths are the privacy primitive behind BOLT 12 offers
and onion messages.

## Walkthrough / mechanics

The receiver constructs a path `[N1 -> N2 -> ... -> Nk]` ending at
themselves. They generate:

```
ephemeral_key e (private), E = e * G (public point, "blinding point")
```

For each hop `Ni` with public key `Pi`, the receiver computes per-hop
"shared secret" via ECDH:

```
ss_i = HKDF(e_i * Pi, "blinded_node_id_i" || ...)
e_{i+1} = HKDF(e_i, ss_i || "blinded_path")  // ratchet
P'_i = Pi + HKDF(ss_i, "blinded_node_pubkey")  * G  // blinded node id
encrypted_payload_i = ChaChaPoly(ss_i, plaintext_i)
```

`plaintext_i` includes the next hop info (real next node id, fee/cltv
deltas, etc.) so node `Ni` can forward.

The receiver publishes the path:

```
blinded_path = {
    introduction_node_id: P_1            (clear)
    blinding_point:       E              (clear)
    blinded_hops: [
        { blinded_node_id: P'_2, encrypted_payload: c_2 },
        { blinded_node_id: P'_3, encrypted_payload: c_3 },
        ...
        { blinded_node_id: P'_k, encrypted_payload: c_k }
    ]
}
```

The sender:
1. Routes a Sphinx onion to `P_1` (introduction node) normally.
2. The introduction node uses `E` and its private key to derive
   `ss_1`, decrypts `payload_1`, learns next hop and gets a new `E_2`
   to forward.
3. Each subsequent node receives `E_i`, derives `ss_i`, decrypts
   `payload_i`. Each node sees only its predecessor and its successor's
   *blinded* identity, not the real ID.

The introduction node is trusted to know the receiver exists in their
direction but not which specific node it is.

The receiver may require a `path_id` in the encrypted payload as
authentication (linking the payment back to the offer/invoice).

## Worked example

Receiver Bob constructs a 3-hop blinded path:

```
Bob = N4 (himself, terminal)
N3, N2, N1 chosen by Bob

introduction_node_id = N1
blinding_point E0 (pub of e0)

For N1:
  ss1 = ECDH(e0, N1.pub)
  encrypted_payload_1 = encrypt({
    short_channel_id: N1->N2,
    payment_relay: { fee_base: 0, fee_proportional: 0, cltv_delta: 144 },
    blinding_override: None
  }, ss1)

For N2 (blinded):
  e1 = HKDF(e0, ss1)
  blinded_node_id_2 = N2.pub + HKDF(ss1, "...") * G
  encrypted_payload_2 = encrypt({...next hop info...}, ss2)

For N3 (blinded):
  similarly

For N4 (Bob, blinded, terminal):
  encrypted_payload_4 = {
    payment_constraints: { max_cltv_expiry, htlc_minimum_msat },
    path_id: invoice_path_id   // Bob's auth
  }
```

Bob shares the path via BOLT 12 invoice or on a website.

Sender Alice computes a route to N1, includes blinded path data in
onion-final-payload of her route's last hop (N1):

```
update_add_htlc to N1's ingress with onion containing:
    hop[N1]: { blinding_point: E0, encrypted_payload: c_1 }
N1 forwards along the blinded path to N2 with E1 and c_2:
    update_add_htlc(... onion containing { blinding_point: E1, encrypted_payload: c_2 } ...)
... etc.
```

CLN ships blinded paths under BOLT 12 (`fetchinvoice`); LND's BOLT 12
support added in 0.18:

```
lightning-cli pay <bolt12_invoice>     # CLN
lncli payinvoice --pay_offer ...       # LND BOLT 12 support
```

LDK exposes `BlindedPath::new_for_message` and `new_for_payment` in
`lightning/src/blinded_path/`.

## Common bugs / pitfalls

- **Introduction node DoS surface**: introduction nodes carry the
  brunt of the receiver's anonymity. Choosing well-connected nodes
  lowers cost, but attackers may target them.
- **Padding to obscure path length**: a 2-hop blinded path looks
  different from a 5-hop one in onion size. Spec recommends padding to
  fixed length.
- **`payment_constraints` mismatch**: receiver-set htlc_minimum_msat
  in the encrypted payload of the terminal hop must align with what
  the invoice/offer advertises. Mismatch causes `invalid_onion_blinding`.
- **Reuse of blinding seed**: re-using `e0` across distinct paths
  leaks correlation. Receivers must generate fresh `e0` per path or
  per offer.
- **Mixed-feature sender**: a sender without `option_route_blinding`
  can't construct the route to N1 with proper blinded onion; they get
  a generic failure. Receiver advertises feature in invoice TLV.
- **Failure attribution**: if a hop in the blinded path fails the
  HTLC, the sender receives a generic `invalid_onion_blinding`
  failure. They cannot tell which hop failed → can't penalize that
  hop in mission control.
- **CLTV padding**: the encrypted payload must specify cltv-delta
  per hop, but the sender doesn't know real per-hop deltas. Receiver
  must pad to a safe maximum or use `payment_relay` honestly.

## References

- BOLT 4 Route Blinding: https://github.com/lightning/bolts/blob/master/04-onion-routing.md#route-blinding
- BOLT 12 Offers (uses blinded paths): https://github.com/lightning/bolts/blob/master/12-offer-encoding.md
- LDK blinded path: `lightning/src/blinded_path/`
- "Route Blinding" proposal (Riard, 2020): https://github.com/lightning/bolts/pull/765
- CLN BOLT 12 plugin: `plugins/fetchinvoice.c`
