# BOLT12 Offer Format - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/bolt12`.
> Canonical source: BOLT12 spec (https://github.com/lightning/bolts/blob/master/12-offer-encoding.md)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/bolt12/SKILL.md

## Concept

A BOLT12 offer is a static, reusable bech32 string starting with `lno1` that
encodes a TLV (tag-length-value) record set signed by the issuer. Unlike BOLT11
invoices, an offer does NOT carry a payment_hash; it is a pointer that a payer
uses to fetch a fresh invoice over Lightning's onion-message transport. This
walkthrough decodes a real offer byte-by-byte to demystify the format.

## Walkthrough / mechanics

Bech32 layer:

```
hrp:  "lno"               (no chain prefix; chain encoded as TLV)
data: 5-bit groups, 1-bit checksum window
```

The TLV body uses standard BOLT 1 BigSize length prefixes. Key TLV types
defined for `offer_*` records:

| TLV | Name              | Required | Notes                                        |
|-----|-------------------|----------|----------------------------------------------|
| 2   | offer_chains      | optional | array of 32-byte chain hashes (mainnet defl)|
| 4   | offer_metadata    | optional | recipient cookie, opaque bytes               |
| 6   | offer_currency    | optional | ISO-4217 string, e.g. "USD"                  |
| 8   | offer_amount      | optional | u64 msat (or fiat unit if currency set)      |
| 10  | offer_description | optional | UTF-8 description                            |
| 12  | offer_features    | optional | feature bitmap                                |
| 14  | offer_absolute_expiry | opt | u64 unix seconds                             |
| 16  | offer_paths       | optional | array of `blinded_path` (hides node id)      |
| 18  | offer_issuer      | optional | UTF-8 issuer name                            |
| 20  | offer_quantity_max| optional | u64                                          |
| 22  | offer_node_id     | optional | 32-byte x-only pubkey                        |

If `offer_paths` is set, payer ignores `offer_node_id` for routing and uses
the blinded path's introduction node instead.

## Worked example

Decoding `lno1qsgqmqvgm96frzdg8m0gc6nzeqffvzsqzrxqy0jpqzpgr...` (truncated):

1. Bech32 decode → hrp=`lno`, 41 bytes of TLV.
2. Parse TLV stream:
   - type=10 len=18 value=`"Coffee subscription"` → description
   - type=8  len=8  value=0x00000000000003E8 → 1000 msat
   - type=22 len=32 value=33-byte compressed → x-only node id
   - type=20 len=2  value=0x000C → quantity_max=12
3. Verify signature: `tagged_hash("lightning", "offer", "signature") = sig`.

To request an invoice, payer constructs an `invoice_request` TLV (types 80-90
range), wraps it in an onion message, and sends it down the path declared in
`offer_paths`. Recipient responds with an `invoice` (lni1) carrying types
80-160 (real payment_hash, encrypted_recipient_data, etc.).

```
Payer                            Recipient
  | invoice_request (onion msg) →   |
  | ← invoice (onion msg)            |
  | --- HTLC payment over BOLT11-style routing --- |
```

## Common bugs / pitfalls

- Encoding the chain prefix in the hrp like BOLT11 does. BOLT12 uses chain
  hashes inside `offer_chains` TLV; hrp is always plain `lno`.
- Treating `offer_node_id` as required when `offer_paths` is present. With
  blinded paths the node id may be absent or be a dummy.
- Validating `offer_absolute_expiry` on the issuer side only. Payers must also
  refuse expired offers; otherwise an invoice request leaks metadata to dead
  endpoints.
- Off-by-one in BigSize parsing. BigSize encodes 0xfd/0xfe/0xff prefix for
  multi-byte; TLV lengths in the wild span 1-3 bytes for typical fields.
- Re-deriving signature over wrong canonical TLV ordering. TLVs MUST be
  serialized in strictly ascending type order before hashing.

## References

- BOLT12 specification: https://github.com/lightning/bolts/blob/master/12-offer-encoding.md
- Onion messages BOLT 4: https://github.com/lightning/bolts/blob/master/04-onion-routing.md
- Rusty Russell's bolt12.org reference impl
- CLN `lightning-fetchinvoice`/`lightning-offer` documentation
