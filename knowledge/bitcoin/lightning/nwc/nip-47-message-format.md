# NIP-47 Message Format - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/nwc`.
> Canonical source: NIP-47 (https://github.com/nostr-protocol/nips/blob/master/47.md)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/nwc/SKILL.md

## Concept

NIP-47 (Nostr Wallet Connect) defines three Nostr event kinds that ferry
JSON-RPC-style messages between an app and a remote Lightning wallet
through standard Nostr relays. The wallet and app each hold a Nostr
keypair; messages are end-to-end encrypted so relays cannot read them.
NIP-44 v2 is the encryption; NIP-04 is deprecated and kept only for peers
that have not migrated, with the two sides negotiating via an `encryption`
tag (see below). The format is intentionally small: kind 23194 for
requests, kind 23195 for responses, kind 13194 for capability
advertisement.

## Walkthrough / mechanics

Three event kinds:

| kind  | author        | direction       | purpose                       |
|-------|---------------|-----------------|-------------------------------|
| 13194 | wallet        | published once  | advertise supported methods   |
| 23194 | app (client)  | request         | encrypted JSON-RPC request    |
| 23195 | wallet        | response        | encrypted JSON-RPC response   |
| 23196 | wallet        | notification    | NIP-04-encrypted async event (NWC-02) |
| 23197 | wallet        | notification    | NIP-44-encrypted async event (NWC-02) |

Only 13194 / 23194 / 23195 are core. Notifications moved out of NIP-47
into the optional NWC-02 extension spec in the 2026-08-01 split
(nostr-protocol/nips#2419); a wallet supporting both encryption schemes
publishes each notification twice, 23196 (NIP-04) and 23197 (NIP-44), and
a NIP-44-only wallet publishes 23197 alone.

### Kind 13194 (info)

Public, unencrypted, content is a space-separated list of method names:

```json
{
  "kind": 13194,
  "pubkey": "<wallet_service_pubkey>",
  "created_at": 1714400000,
  "tags": [
    ["encryption", "nip44_v2 nip04"],
    ["extensions", "02 03 04 06 07 08"],
    ["notifications", "payment_received payment_sent"]
  ],
  "content": "pay_invoice get_balance make_invoice lookup_invoice get_info",
  "sig": "..."
}
```

The `encryption` tag lists the schemes the wallet accepts; its absence
means NIP-04 only. The `extensions` tag lists the optional NWC extension
specs the wallet implements - gate any non-core method on it.

### Kind 23194 (request)

```json
{
  "kind": 23194,
  "pubkey": "<app_pubkey>",
  "created_at": 1714400123,
  "tags": [
    ["encryption", "nip44_v2"],
    ["p", "<wallet_service_pubkey>"]
  ],
  "content": "<NIP-44 v2 encrypted JSON>",
  "sig": "..."
}
```

The request's `encryption` tag names the scheme used for `content` and
MUST be one the info event advertised; omitting it means NIP-04. A
request MAY also carry an `expiration` tag (unix seconds) after which the
wallet service ignores it.

Decrypted content shape:

```json
{
  "method": "pay_invoice",
  "params": { "invoice": "lnbc100u1..." }
}
```

### Kind 23195 (response)

```json
{
  "kind": 23195,
  "pubkey": "<wallet_service_pubkey>",
  "created_at": 1714400125,
  "tags": [["p", "<app_pubkey>"], ["e", "<request_event_id>"]],
  "content": "<NIP-44 v2 encrypted JSON>",
  "sig": "..."
}
```

Decrypted content:

```json
{
  "result_type": "pay_invoice",
  "result": { "preimage": "abcd...64-hex" }
}
```

Or error form:

```json
{
  "result_type": "pay_invoice",
  "error": { "code": "INSUFFICIENT_BALANCE", "message": "..." }
}
```

Standard error codes: `RATE_LIMITED`, `NOT_IMPLEMENTED`, `INSUFFICIENT_BALANCE`,
`QUOTA_EXCEEDED`, `RESTRICTED`, `UNAUTHORIZED`, `INTERNAL`,
`UNSUPPORTED_ENCRYPTION`, `OTHER`.

The `e`-tag on responses lets the client correlate without a parallel
request id.

## Worked example

Connection URI:

```
nostr+walletconnect://b3c0a8...e5d1
  ?relay=wss://relay.damus.io
  &secret=8d1a5b...4f
  &lud16=alice@example.com
```

App publishes request to relay:

```
{
  "kind": 23194,
  "pubkey": "9af3...c7",   // = secret * G
  "created_at": 1714400123,
  "tags": [["encryption","nip44_v2"], ["p","b3c0a8...e5d1"]],
  "content": "fGh4...==",  // nip44_v2(secret, b3c0a8...).encrypt(
                           //   '{"method":"pay_invoice",
                           //     "params":{"invoice":"lnbc100u..."}}')
  "id": "<sha256 of canonical>",
  "sig": "<schnorr by secret>"
}
```

Wallet subscribes with REQ filter `{ "kinds": [23194], "#p": ["b3c0a8...e5d1"], "since": now-300 }`.
Receives event, decrypts, processes, publishes 23195. App's subscription
`{ "kinds": [23195], "#p": ["9af3...c7"], "since": ... }` receives the
response, decrypts, returns to caller.

## Common bugs / pitfalls

- Using event id instead of `e`-tag for correlation. Some implementations
  rely solely on `e`-tag; clients must support both.
- Clock skew rejecting events: relays often drop events with
  `created_at` more than 5 min off real time. Use NTP-synced clocks.
- Replay protection absent in spec: a relay or eavesdropper cannot
  decrypt but can resubmit a 23194 to the wallet's subscription.
  Wallets MUST track event ids and reject duplicates.
- 23196 / 23197 notifications are optional (NWC-02); clients that
  subscribe blindly may miss support and silently degrade. Subscribing to
  23196 only, against a NIP-44-only wallet, silently receives nothing.
- Assuming NIP-04. As of the September 2026 spec NIP-04 is deprecated and
  NIP-44 v2 is what to use; negotiation is via 13194's `encryption` tag
  (`["encryption", "nip44_v2 nip04"]`) plus a matching `encryption` tag on
  each request. Absence of the tag on the info event - and only then -
  means the wallet is NIP-04-only. Sending a scheme the wallet did not
  advertise returns `UNSUPPORTED_ENCRYPTION`.
- Per-relay rate limits: high-volume apps must use multiple relays
  declared in the URI's `relay` query param.

## References

- NIP-47: https://github.com/nostr-protocol/nips/blob/master/47.md
- NWC extension specs (notifications, keysend, hold invoices, ...):
  https://github.com/nostr-wallet-connect/nwc
- NIP-04 encryption: https://github.com/nostr-protocol/nips/blob/master/04.md
- NIP-44 encryption: https://github.com/nostr-protocol/nips/blob/master/44.md
- NWC reference SDK: https://github.com/getAlby/js-sdk
