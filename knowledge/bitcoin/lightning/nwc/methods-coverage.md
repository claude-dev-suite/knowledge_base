# NWC Methods Coverage - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/nwc`.
> Canonical source: NIP-47 method definitions
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/nwc/SKILL.md

## Concept

NIP-47 standardizes JSON-RPC methods for wallet remote control, ranging
from full payment send / receive to read-only balance and transaction
listing. On 2026-08-01 (nostr-protocol/nips#2419) the spec was split: the
core NIP-47 command set is now just `pay_invoice`, `make_invoice`,
`lookup_invoice`, `get_balance` and `get_info`, and everything else moved
into optional NWC extension specs kept at
`github.com/nostr-wallet-connect/nwc`. Wallets selectively support
subsets; clients MUST inspect the kind 13194 capability event - both its
`content` method list and its `extensions` tag - before assuming a method
works. This article documents each method's params, results, common error
codes, and which Lightning impls support what.

## Walkthrough / mechanics

Method matrix (selected). The `spec` column is the document that defines
the method as of September 2026:

| Method               | Spec    | Params (key)                        | Result                 |
|----------------------|---------|-------------------------------------|------------------------|
| `pay_invoice`        | core    | `invoice`, optional `amount` msat, `metadata?` | `{ preimage, fees_paid }` |
| `make_invoice`       | core    | `amount`, `description?`, `description_hash?`, `expiry?`, `metadata?` | `{ type:"incoming", invoice, payment_hash, ...}` |
| `lookup_invoice`     | core    | `payment_hash` or `invoice`         | full transaction obj   |
| `get_balance`        | core    | `{}`                                | `{ balance: msat }`    |
| `get_info`           | core    | `{}`                                | wallet info struct, incl. `methods[]` + `extensions[]` |
| `make_hold_invoice`  | NWC-03  | `amount`, `payment_hash`, `description?` | `{ invoice }`         |
| `pay_keysend`        | NWC-04  | `amount`, `pubkey`, `preimage?`, `tlv_records?` | `{ preimage }` |
| `list_transactions`  | NWC-05  | `from`, `until`, `limit`, `offset`, `unpaid?`, `type?` | `{ transactions:[...] }` |
| `lookup_payment`     | NWC-09  | payment-type selectors              | payment record envelope |
| `make_offer`         | NWC-12  | `amount?`, `description?`, `issuer?`, `single_use?`, `expires_at?` | `{ offer_id, offer, amount, single_use, ... }` |
| `pay` / `receive`    | NWC-321 | BIP-321 payment instruction         | type-dependent         |

`multi_pay_invoice` and `multi_pay_keysend` were core NIP-47 commands
until nostr-protocol/nips#2210 (merged 2026-02-11) removed them, six
months ahead of the August 2026 split. `sign_message` was never specified
in NIP-47 at all - it appeared only inside the appendix example info
event's content string, and is a wallet-level extension (Alby Hub
implements one). None of the three is defined in the core spec or in any
current NWC extension spec. Wallets that shipped them may still answer;
treat them as non-standard as of September 2026 and gate on the capability
event rather than on this table.

Each method has standard error codes; most-common are
`INSUFFICIENT_BALANCE`, `RESTRICTED` (method not granted by user),
`NOT_IMPLEMENTED`, `RATE_LIMITED`, `QUOTA_EXCEEDED`, `INTERNAL`.
Method-specific codes in the core spec: `PAYMENT_FAILED` (`pay_invoice`),
`NOT_FOUND` (`lookup_invoice`), plus `UNSUPPORTED_ENCRYPTION` when the
request's `encryption` tag names a scheme the wallet service lacks.

### list_transactions schema

```json
{
  "type": "incoming" | "outgoing",
  "invoice": "lnbc...",
  "description": "...",
  "description_hash": "...",
  "preimage": "...",       // null for unsettled
  "payment_hash": "...",
  "amount": 1000000,        // msat
  "fees_paid": 1000,        // msat, only for outgoing
  "created_at": 1714400000,
  "expires_at": 1714400600,
  "settled_at": 1714400123  // null if pending
}
```

### Permission scoping

Scope is bound to the connection secret on the wallet side, not carried
on the connection URI: NIP-47's `nostr+walletconnect://` query
parameters are only `relay`, `secret` and `lud16` as of September 2026.
A request for a method the connection was not granted returns
`RESTRICTED`; the client learns what it actually holds from the kind
13194 content list and from `get_info`.

A client can ask for a scope up front only under NWC-08
(client-initiated connections), whose authorization request carries
`request_methods` / `optional_request_methods` (URL-encoded,
space-separated) plus `max_amount` in msat with `renewal_period`
(`never` | `daily` | `weekly` | `monthly` | `yearly`):

```
nostr+walletauth://<client_pubkey>?relay=wss%3A%2F%2Frelay.example.com
  &state=<128-bit-hex>&request_methods=pay_invoice%20get_balance
  &max_amount=1000000&renewal_period=monthly
```

If `request_methods` is present the wallet service MUST grant every
listed method or decline the request; if `max_amount` is present it MUST
enforce it over that period or decline. `&methods=` and `&budget=` are
in no spec - per-app budgets outside NWC-08 are a wallet feature (Alby
Hub sets a spending budget per app connection in its own UI, as of
September 2026).

## Worked example

App-side library wrapping NWC:

```
async pay(invoice, amount_msat):
    req = {
        "method": "pay_invoice",
        "params": { "invoice": invoice, "amount": amount_msat }
    }
    # NIP-44 v2 is the encryption to use. NIP-04 only for wallets whose
    # info event lacks nip44_v2 in its `encryption` tag.
    event = nip44_encrypt_and_sign(secret, wallet_pubkey, req,
                                   tags=[["encryption", "nip44_v2"],
                                         ["p", wallet_pubkey]])
    publish_to_relay(event)

    wait_for(kind=23195, e_tag=event.id, timeout=30s):
        resp = nip44_decrypt(secret, wallet_pubkey, content)
        if resp.error:
            raise NwcError(resp.error.code, resp.error.message)
        return resp.result.preimage
```

Capability check:

```
get_info_event = fetch_kind(13194, author=wallet_pubkey, latest=1)
methods = get_info_event.content.split(' ')
if "pay_invoice" not in methods:
    raise UnsupportedMethod

# Non-core methods additionally need their extension advertised.
exts = tag_value(get_info_event, "extensions", default="").split(' ')
if "05" not in exts:
    raise UnsupportedMethod        # no list_transactions

# The same event carries the encryption negotiation.
enc = tag_value(get_info_event, "encryption", default="nip04").split(' ')
scheme = "nip44_v2" if "nip44_v2" in enc else "nip04"
```

Coverage by wallet (late 2025):

| Wallet          | pay_inv | make_inv | get_bal | multi_pay | sign_msg | hold_inv |
|-----------------|---------|----------|---------|-----------|----------|----------|
| Alby (custodial)| yes     | yes      | yes     | yes       | yes      | partial  |
| Mutiny          | yes     | yes      | yes     | yes       | no       | no       |
| Phoenix         | yes     | yes      | yes     | partial   | no       | no       |
| LNbits          | yes     | yes      | yes     | partial   | yes      | yes      |
| Cashu Mints     | yes     | yes      | yes     | no        | no       | no       |

## Common bugs / pitfalls

- Calling `pay_invoice` with both `invoice` (zero-amount) and `amount`
  parameter omitted -> wallet returns `INTERNAL`. Always specify amount
  for amountless invoices.
- Treating `multi_pay_invoice` as atomic. It is NOT - each sub-payment
  succeeds or fails independently. Client must reconcile partials.
- `make_hold_invoice` returns immediately, but settle requires a separate
  RPC out-of-band on most wallets (Alby exposes it; Mutiny does not).
- `lookup_invoice` after expiry: spec lets wallets purge old invoices.
  Don't rely on indefinite retention.
- Time-zone in `created_at`: always Unix UTC seconds, not ms or local.
- Budget exhaustion mid-day: wallets that enforce per-day caps return
  `QUOTA_EXCEEDED` with no auto-retry header. Surface clearly to user.

## References

- NIP-47 method specs: https://github.com/nostr-protocol/nips/blob/master/47.md
- NWC extension specs (02-321): https://github.com/nostr-wallet-connect/nwc
- Alby OAuth NWC docs
- LNbits NWC extension source
- nostr-wallet-connect-rust: https://github.com/getAlby/nwc-rust
