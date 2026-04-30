# LUD-06 LNURL-pay - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/lnurl`.
> Canonical source: LUD-06 (https://github.com/lnurl/luds/blob/luds/06.md)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/lnurl/SKILL.md

## Concept

LNURL-pay is a two-step HTTP flow that produces a fresh BOLT11 invoice for
an arbitrary amount within `[minSendable, maxSendable]`. The wallet first
fetches a JSON descriptor containing the metadata, then issues a `GET`
to the descriptor's `callback` with the chosen amount. The descriptor's
metadata is cryptographically bound into the returned invoice via a
description hash, preventing mid-flow swaps.

## Walkthrough / mechanics

Step 1 - descriptor fetch (GET):

```
GET https://merchant.example.com/lnurl/pay/abc HTTP/1.1
Accept: application/json
```

Response (descriptor):

```json
{
  "tag": "payRequest",
  "callback": "https://merchant.example.com/lnurl/pay/abc/cb",
  "minSendable": 1000,
  "maxSendable": 100000000,
  "metadata": "[[\"text/plain\",\"Coffee\"],[\"text/identifier\",\"alice@merchant.example.com\"]]",
  "commentAllowed": 200,
  "payerData": {
    "name":  { "mandatory": false },
    "email": { "mandatory": false },
    "auth":  { "mandatory": false, "k1": "<hex32>" }
  }
}
```

Step 2 - callback (GET) with amount in millisat:

```
GET https://merchant.example.com/lnurl/pay/abc/cb
        ?amount=5000000
        &comment=Latte+please
        &payerdata=%7B%22name%22%3A%22Alice%22%7D
```

Response:

```json
{
  "pr": "lnbc50u1p3xz...",
  "successAction": {
    "tag": "url",
    "description": "Receipt",
    "url": "https://merchant.example.com/receipts/abc"
  },
  "routes": []
}
```

Step 3 - integrity check:

```
expected = sha256(utf8(metadata_json_string + json_str(payerdata)))
actual   = invoice.description_hash    (BOLT11 'h' tag)
require expected == actual
```

If they differ, the wallet MUST refuse to pay - the server (or a MITM)
swapped the invoice for one with different displayed information.

Step 4 - pay invoice over Lightning, then optionally honor `successAction`.

## Worked example

```
Wallet decodes LNURL -> URL = https://merchant.com/lnurl/pay/abc

GET descriptor:
  HTTP/1.1 200 OK
  Content-Type: application/json
  { "tag":"payRequest","callback":".../cb","minSendable":1000,
    "maxSendable":100000000,
    "metadata":"[[\"text/plain\",\"Coffee\"]]","commentAllowed":50 }

User picks 50,000 sat (50_000_000 msat), comment "Thanks":
GET .../cb?amount=50000000&comment=Thanks
  -> { "pr":"lnbc500u1p3...", "successAction":{...}, "routes":[] }

Wallet decodes pr:
  amount: 500000 sat? -> wait, 50_000_000 msat = 50_000 sat = 50k.
  Mismatch detection: invoice msat MUST equal callback amount.
  description_hash: SHA256("[[\"text/plain\",\"Coffee\"]]") = e0c9...
  Compare to BOLT11 'h' tag -> match -> proceed to pay.
```

Note the **msat** unit: `amount` in the callback query string is in
**millisat**, while wallet UIs commonly display sat. A 1000x off-by-one
here is the most common LNURL-pay integration bug.

## Common bugs / pitfalls

- Sending `amount` in sats instead of msat. Server interpretation varies;
  some accept sat, most reject. Always use msat.
- Skipping description_hash verification. Without it the server can issue
  an invoice for the same amount but with a different (e.g. attacker-
  controlled) description, indistinguishable from honest behaviour.
- Accepting `commentAllowed > 1024`. Some receivers truncate longer
  comments silently; the spec recommends max 1024 chars.
- Treating `metadata` as JSON. It's a string that JSON-encodes a list, but
  the hash MUST be over the **string** as returned, not over a re-serialized
  variant (Unicode escapes, key order differences).
- `successAction.url` not on same eTLD+1 as callback domain. Phishing risk.
- Insufficient timeout: invoice expiry is typically 10 min; if the wallet
  takes longer to pay (slow LSP), payment fails.

## References

- LUD-06: https://github.com/lnurl/luds/blob/luds/06.md
- LUD-09 successAction: https://github.com/lnurl/luds/blob/luds/09.md
- LUD-12 comment field: https://github.com/lnurl/luds/blob/luds/12.md
- LUD-18 payerData: https://github.com/lnurl/luds/blob/luds/18.md
