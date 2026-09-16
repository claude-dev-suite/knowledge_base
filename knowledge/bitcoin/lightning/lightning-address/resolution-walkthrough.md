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
delegation in front (uncommon). The TXT-record scheme that shares the
same `user@domain` spelling is a *different* protocol, BIP 353 - see
"BIP 353 resolution" below.

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

## BIP 353 resolution (DNS instead of HTTPS)

BIP 353 "DNS Payment Instructions" (Matt Corallo, Bastien Teinturier)
resolves the same `user@domain` spelling without contacting the payee's
web server at all. It is displayed with a `₿` prefix to distinguish it
from an email address, and the instructions it returns are a BIP 21 URI -
for Lightning, normally an `lno=` BOLT12 offer. Status in the BIPs repo
is `Complete` as of September 2026 (Draft -> Proposed in September 2025,
Proposed -> Complete in January 2026 under the revised BIP process).

Steps:

1. Split `user@domain`, stripping a leading `₿` if present. Non-ASCII
   labels must be punycode-encoded (RFC 3492/5891); wallets SHOULD avoid
   creating non-ASCII identifiers at all.
2. Query the TXT RRset at:

```
{local_part}.user._bitcoin-payment.{domain}.
```

   The literal `user` label sits between the local part and
   `_bitcoin-payment`; omitting it is the single most common bug.
3. Ignore any TXT record at that label not starting with `bitcoin:`
   (case-insensitive). If two `bitcoin:` records exist, the whole RRset
   is invalid - refuse to pay.
4. Rebuild the URI by concatenating that RR's `<character-string>`s in
   RDATA order with no separator. Never concatenate across RRs.
5. Validate the full DNSSEC chain to the root locally, including any
   CNAME/DNAME hops. Reject SHA-1 and RSA keys under 1024 bits (SHA-1 DS
   records MAY be accepted). Trusting a remote resolver's `AD` bit is
   explicitly non-conformant.
6. Parse the BIP 21 URI and pay the `lno=` offer (or the on-chain path if
   that is all the record carries).

Cache limit: never longer than the resolver's TTL, and never longer than
the lowest initial TTL in the DNSSEC chain from the root down.

Worked record:

```
matt.user._bitcoin-payment.mattcorallo.com. 1800 IN TXT "bitcoin:?lno=lno1qsgr30k..."
```

(shape taken from the example in BIP 353 itself)

Two resolution transports exist:

- **Clearnet / DoH.** Cheap, but the query tells a resolver you are about
  to pay that name. ACINQ's phoenixd (`PayDnsAddress`) does this against
  `dns.google` and checks the response `AD` flag instead of validating
  locally - convenient, weaker than the spec requires.
- **bLIP-32 "Onion Message DNS Resolution" (Active).** `dnssec_query`
  (type 65536) out, `dnssec_proof` (65538) or `dnssec_error` (65550)
  back over a `reply_path`, from a node advertising the `dns_resolver`
  feature bit 258/259 (bLIP-2). The proof is RFC 9102 formatted and
  validated by the sender, so the resolver need not be trusted. LDK's
  `lightning-dns-resolver` crate implements this on top of
  `dnssec-prover`.

Hardware-wallet path: a PSBT can carry the proof per output in
`PSBT_OUT_DNSSEC_PROOF = 0x35` (1-byte-length-prefixed name without the
`₿`, then an RFC 9102 `AuthenticationChain`), letting the signing device
display `₿user@domain` instead of a raw script. Verify RRSig inception
is not in the future and expiry is no more than an hour in the past.

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
- BIP 353: https://github.com/bitcoin/bips/blob/master/bip-0353.mediawiki
- bLIP-32 (Onion Message DNS Resolution): https://github.com/lightning/blips/blob/master/blip-0032.md
- bLIP-42 (Bolt 12 Contacts): https://github.com/lightning/blips/blob/master/blip-0042.md
- LDK `lightning-dns-resolver` crate (bLIP-32 client)
- ACINQ phoenixd `PayDnsAddress.kt` (BIP 353 over DoH)
