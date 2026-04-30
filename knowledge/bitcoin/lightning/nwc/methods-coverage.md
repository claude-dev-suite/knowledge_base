# NWC Methods Coverage - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/nwc`.
> Canonical source: NIP-47 method definitions
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/nwc/SKILL.md

## Concept

NIP-47 standardizes ten or so JSON-RPC methods for wallet remote control,
ranging from full payment send / receive to read-only balance and
transaction listing. Wallets selectively support subsets; clients MUST
inspect the kind 13194 capability event before assuming a method works.
This article documents each method's params, results, common error codes,
and which Lightning impls support what.

## Walkthrough / mechanics

Method matrix (selected):

| Method               | Params (key)                        | Result                 |
|----------------------|-------------------------------------|------------------------|
| `pay_invoice`        | `invoice`, optional `amount` msat   | `{ preimage, fees_paid }` |
| `multi_pay_invoice`  | `invoices: [{id, invoice, amount}]` | `[ { id, preimage } ]` |
| `pay_keysend`        | `amount`, `pubkey`, `preimage?`, `tlv_records?` | `{ preimage }` |
| `multi_pay_keysend`  | `keysends: [...]`                   | `[ ... ]`             |
| `make_invoice`       | `amount`, `description?`, `description_hash?`, `expiry?` | `{ type:"incoming", invoice, payment_hash, ...}` |
| `make_hold_invoice`  | `amount`, `payment_hash`, `description?` | `{ invoice }`         |
| `lookup_invoice`     | `payment_hash` or `invoice`         | full transaction obj   |
| `list_transactions`  | `from`, `until`, `limit`, `offset`, `unpaid?`, `type?` | `{ transactions:[...] }` |
| `get_balance`        | `{}`                                | `{ balance: msat }`    |
| `get_info`           | `{}`                                | wallet info struct     |
| `sign_message`       | `message`                           | `{ signature, message }` |

Each method has standard error codes; most-common are
`INSUFFICIENT_BALANCE`, `RESTRICTED` (method not granted by user),
`NOT_IMPLEMENTED`, `RATE_LIMITED`, `QUOTA_EXCEEDED`, `INTERNAL`.

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

NWC URIs may carry `&methods=pay_invoice,make_invoice,get_balance` to
restrict which methods the app can use. Wallets enforce: a request for
a non-listed method returns `RESTRICTED`. Some wallets layer additional
budget caps (e.g. `&budget=10000/daily`) - Alby NWC supports per-day
sat budgets.

## Worked example

App-side library wrapping NWC:

```
async pay(invoice, amount_msat):
    req = {
        "method": "pay_invoice",
        "params": { "invoice": invoice, "amount": amount_msat }
    }
    event = nip04_encrypt_and_sign(secret, wallet_pubkey, req)
    publish_to_relay(event)

    wait_for(kind=23195, e_tag=event.id, timeout=30s):
        resp = nip04_decrypt(secret, wallet_pubkey, content)
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
- Alby OAuth NWC docs
- LNbits NWC extension source
- nostr-wallet-connect-rust: https://github.com/getAlby/nwc-rust
