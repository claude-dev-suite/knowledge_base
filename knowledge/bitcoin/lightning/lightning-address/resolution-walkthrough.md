# Lightning Address Resolution - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/lightning-address`.
> Canonical source: LUD-16 (https://github.com/lnurl/luds/blob/luds/16.md)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/lightning-address/SKILL.md

## Concept

A Lightning Address `user@domain.tld` is a thin alias on top of LNURL-pay.
Resolution is purely DNS-and-HTTPS based: split the local part from the
domain, fetch a well-known URL, then continue with a standard LNURL-pay
flow. The clever trick is the absence of a separate registry: any HTTPS
server with a `/.well-known/lnurlp/{user}` endpoint can serve any local
part, allowing free self-hosting.

## Walkthrough / mechanics

Step-by-step:

1. Validate input matches `^[a-z0-9._+-]+@[a-z0-9.-]+\.[a-z]{2,}$`. The
   local part is restricted to lowercase ASCII subset; non-ASCII Unicode
   parts are NOT permitted by spec (avoids homograph attacks).
2. Lowercase both halves (Lightning Addresses are case-insensitive).
3. Construct URL:

```
https://{domain}/.well-known/lnurlp/{local_part}
```

4. GET that URL with `Accept: application/json`. Follow up to 3 redirects
   on the same eTLD+1.
5. Treat the response as a LUD-06 payRequest descriptor. From this point,
   the flow is identical to LUD-06: choose amount, GET callback, pay.

Implementation detail: the wallet does **not** require a TXT/SRV DNS
record. Standard A/AAAA + HTTPS is enough. Some custodians use SRV
delegation in front (uncommon).

## Worked example

User input: `alice@getalby.com`

```
Wallet computes URL:
  GET https://getalby.com/.well-known/lnurlp/alice HTTP/1.1
  Accept: application/json
  User-Agent: ExampleWallet/2.4

Server responds:
  HTTP/1.1 200 OK
  Cache-Control: max-age=60
  Content-Type: application/json

  {
    "tag": "payRequest",
    "callback": "https://getalby.com/lnurlp/alice/callback",
    "minSendable": 1000,
    "maxSendable": 10000000000,
    "metadata": "[[\"text/identifier\",\"alice@getalby.com\"],
                  [\"text/plain\",\"Sats for alice\"]]",
    "commentAllowed": 255,
    "payerData": { "name": { "mandatory": false } },
    "nostrPubkey": "<hex32>",  // LUD-19 zaps support if present
    "allowsNostr": true
  }

Wallet GETs callback for 5000 sat:
  GET https://getalby.com/lnurlp/alice/callback?amount=5000000

  -> { "pr": "lnbc50u1pr...",
       "successAction": { "tag":"message", "message":"Thanks!" },
       "verify": "https://getalby.com/lnurlp/alice/verify/abc",
       "routes": [] }

Wallet pays the invoice over Lightning.
```

Note that `metadata[0]` SHOULD include `text/identifier` set to the full
Lightning Address - this is a LUD-16 convention enabling wallets to
display "Paid alice@getalby.com" instead of an opaque BOLT11 description.

## Common bugs / pitfalls

- Case sensitivity. The local part is lowercased before path construction;
  servers MAY treat the path case-insensitively but most filesystem-backed
  servers do not. Always normalize.
- HTTP fallback. Some old wallets still try `http://` if `https://` fails.
  The spec mandates HTTPS; never accept the fallback.
- Trailing slashes / redirects. `/.well-known/lnurlp/alice/` and `/alice`
  may behave differently. Spec requires exact form `/.well-known/lnurlp/{user}`
  with no trailing slash.
- Treating the address as a wallet/account identifier. It is a server
  endpoint pointer; the funds path on the server side is independent.
- Caching the descriptor too aggressively. Server may rotate `callback`
  per-request; honor `Cache-Control` headers, default to no-cache.
- Allowing arbitrary local parts in the URL without validation. A `..`
  or path traversal in the local part can hit arbitrary endpoints on
  poorly-implemented servers. Validate against the regex above.

## References

- LUD-16: https://github.com/lnurl/luds/blob/luds/16.md
- LUD-06: https://github.com/lnurl/luds/blob/luds/06.md
- NIP-57 zap LNURL extension (LUD-19 references)
- BTCPay Server Lightning Address plugin source
