# BIP21 URI Scheme - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/wallets/uri-schemes`.
> Canonical source: https://github.com/bitcoin/bips/blob/master/bip-0021.mediawiki
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/wallets/uri-schemes/SKILL.md

## Concept

BIP21 defines the `bitcoin:` URI scheme for representing payment requests as a single string. A wallet handed a BIP21 URI knows the destination address, intended amount, and optional metadata without the user typing anything. The scheme uses standard `application/x-www-form-urlencoded` query parameters, so any URL parser plus a domain validator can decode it. The hard rule of BIP21 is the `req-` prefix: any unknown parameter starting with `req-` MUST cause the parser to reject the URI, while unknown parameters without that prefix MUST be ignored. This forward-compat hook is what lets PayJoin (BIP78), Silent Payments, and unified Lightning URIs extend BIP21 without breaking older wallets.

## Walkthrough / mechanics

Grammar (RFC 3986 derived):

```
bitcoinurn     = "bitcoin:" bitcoinaddress [ "?" bitcoinparams ]
bitcoinparams  = bitcoinparam [ "&" bitcoinparams ]
bitcoinparam   = amountparam | labelparam | messageparam | otherparam | reqparam
amountparam    = "amount=" *digit [ "." *digit ]
labelparam     = "label=" *qchar
messageparam   = "message=" *qchar
otherparam     = qchar *qchar "=" *qchar
reqparam       = "req-" qchar *qchar "=" *qchar
```

Decoder steps:

1. Strip the `bitcoin:` scheme prefix (case-insensitive).
2. Split on `?` once. The left side is the address; right side is the param string.
3. Validate the address (Base58Check or Bech32 / Bech32m for SegWit).
4. URL-decode each `key=value` pair (percent-decoding, `+` is NOT a space here per RFC 3986; some wallets accept it for HTML-form compat).
5. Convert `amount=` from BTC decimal to satoshis using exact decimal math (NOT `float`).
6. For each unknown key starting with `req-`, abort and surface the error.
7. Hand `(address, amount_sats, label, message, extras)` to the send screen.

Encoder steps:

1. Validate the address with the same rules.
2. Format `amount` as a decimal BTC string with no trailing zeros and no scientific notation (e.g., `0.00012345`, never `1.2345e-4`).
3. Percent-encode `label` and `message` (RFC 3986 unreserved set).
4. Concatenate.

## Worked example

Plain payment request:

```
bitcoin:bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh
```

Coffee shop with amount and label (`Coffee Shop` -> `Coffee%20Shop`):

```
bitcoin:bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh?amount=0.001&label=Coffee%20Shop
```

After decoding `amount=0.001`, the wallet must send exactly 100000 sats. Using IEEE-754 doubles a naive implementation gets `0.0009999999999999998 BTC = 99999 sats` because `0.001` is not representable; this is one of the most common BIP21 bugs in the wild.

Order with message and an unknown but ignorable parameter:

```
bitcoin:bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh?amount=0.0005&message=Order%20%23123&source=shopify
```

`source=shopify` is preserved by the wallet (sometimes used for analytics) but is not required for the payment.

A future-spec example that older wallets MUST reject:

```
bitcoin:bc1qxy2...?amount=0.001&req-bolt12=lno1pgz5gh...
```

The `req-bolt12=` parameter signals: "this URI demands BOLT12 fallback"; an old wallet without BOLT12 support MUST refuse to send rather than silently fall back to on-chain.

## Common pitfalls

- Parsing `amount` with `float()` or `parseFloat()`. Always parse as a decimal string (`Decimal` in Python, `BigDecimal` in Java/JS, custom string-shift in TS) and convert to integer satoshis with `Decimal(amount) * 10**8`.
- Treating `amount` as satoshis. The unit is BTC; an off-by-1e8 error sends 1 BTC instead of 0.01 BTC.
- Skipping the `req-` rejection rule. If a future spec adds `req-must-not-pay-to-this-address` and an old wallet ignores it, security regresses.
- Mixed-case Bech32 addresses inside the URI. The Bech32 checksum is only valid if the entire string is one case. `BITCOIN:BC1Q...?amount=0.001` is valid (denser QR), `bitcoin:Bc1q...` is not.
- Double-decoding `message=`. If the wallet displays the decoded `message` as HTML it must escape `<`, `>`, `&` to avoid XSS in the wallet UI.
- Accepting `bitcoin://` (with double slash). The BIP21 scheme is `bitcoin:` only; double-slash is a frequent copy-paste error from URLs that some wallets silently strip and others reject.

## References

- BIP21: https://github.com/bitcoin/bips/blob/master/bip-0021.mediawiki
- RFC 3986 (URI generic syntax): https://www.rfc-editor.org/rfc/rfc3986
- BIP72 (deprecated payment-request extension): https://github.com/bitcoin/bips/blob/master/bip-0072.mediawiki
- bitcoinjs-lib `bitcoin-uri` reference parser
- BDK `Address::parse_url` source (Rust)
