# BOLT12 Blinded Paths - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/bolt12`.
> Canonical source: BOLT 4 (route blinding) + BOLT 12 (offer_paths)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/bolt12/SKILL.md

## Concept

A blinded path is a partial route encoded inside an offer (or invoice) where
each intermediate hop's identity is replaced by an ephemeral pubkey derived
via ECDH. The payer learns only the introduction node's clear pubkey; the
recipient and intermediate hops are concealed. This is BOLT12's primary
recipient-privacy mechanism and is also used in onion-message transport.

## Walkthrough / mechanics

Recipient builds a blinded path R = [N_intro, N_1, ..., N_K] (where N_K is
itself):

1. Pick session key `e_0` (32 random bytes).
2. For each hop i:
   - `E_i = e_i * G` (blinding point published in the path)
   - `ss_i = ECDH(e_i, P_i)` where `P_i` is hop i's real node pubkey
   - `B_i = P_i + HMAC("blinded_node_id", ss_i)*G` (hop i's blinded pubkey)
   - `e_{i+1} = SHA256(E_i || ss_i) * e_i`
3. Encrypt per-hop payload with key derived from `ss_i`. Payload contains
   `next_node_id` (real pubkey of hop i+1 in clear, since only this hop sees it)
   plus padding to defeat length analysis.

The path published in the offer is:

```
introduction_node_id = P_0      (clear)
blinding             = E_0      (clear)
encrypted_data_tlvs  = [enc_0, enc_1, ..., enc_K]
```

The payer treats the introduction node as the destination of its outer
onion. From there, each hop unwraps its `encrypted_recipient_data` using the
ECDH-derived key and passes a tweaked blinding to the next.

## Worked example

Recipient `Carol` (pubkey `P_C`) hides behind two hops `Alice`, `Bob`:

```
e_0 = 0x4242...4242
E_0 = e_0 * G
ss_A = ECDH(e_0, P_A)
B_A  = P_A + H("blinded_node_id" || ss_A) * G
enc_A = ChaCha20Poly1305(key=H("rho"||ss_A),
                         plaintext=TLV{ next_node_id: P_B, padding })
e_1 = SHA256(E_0 || ss_A) * e_0
... repeat for Bob, Carol ...
```

Offer `offer_paths` TLV value (simplified):

```
[ first_node_id=P_A,
  blinding=E_0,
  num_hops=3,
  hops=[(blinded_node_id=B_A, enc_payload=enc_A),
        (blinded_node_id=B_B, enc_payload=enc_B),
        (blinded_node_id=B_C, enc_payload=enc_C)] ]
```

Payer routes outer onion to `P_A`. `P_A` decrypts `enc_A`, learns next is
`P_B`, derives next blinding `E_1 = next_blinding_override OR SHA256(E_0||ss_A)*E_0`,
and forwards. By the time the HTLC reaches Carol, no intermediate has learned
the recipient identity - they only know they were a relay.

## Common bugs / pitfalls

- Payer's outer onion accidentally targets `B_0` (blinded id) instead of `P_0`
  (introduction node real pubkey). The introduction node is the only one that
  must be addressed directly.
- Forgetting the `blinding` field rotation. Each hop derives its own blinding;
  failing to forward the correct `current_blinding_point` breaks decryption
  at the next hop.
- Timing leak: a blinded path with very short padding reveals position. Use
  consistent padding sizes (e.g. multiples of 64 bytes) across hops.
- CLTV deltas in encrypted payloads must accommodate the longest realistic
  path; under-padding leaks recipient depth.
- Re-using the same blinded path across many invoice requests links them. For
  high-privacy recipients, rotate the session key per offer or use multi-path
  variants.

## References

- BOLT 4 route blinding section: https://github.com/lightning/bolts/blob/master/04-onion-routing.md#route-blinding
- BOLT 12: https://github.com/lightning/bolts/blob/master/12-offer-encoding.md
- "Route Blinding" (TonyAiuto, Bastien T.) draft
- CLN `lightning-bkpr-listbalances` + `lightning-decode` blinded path tools
