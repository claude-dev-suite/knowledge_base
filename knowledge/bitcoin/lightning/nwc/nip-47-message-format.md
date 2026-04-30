# NIP-47 Message Format - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/nwc`.
> Canonical source: NIP-47 (https://github.com/nostr-protocol/nips/blob/master/47.md)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/nwc/SKILL.md

## Concept

NIP-47 (Nostr Wallet Connect) defines two Nostr event kinds that ferry
JSON-RPC-style messages between an app and a remote Lightning wallet
through standard Nostr relays. The wallet and app each hold a Nostr
keypair; messages are end-to-end encrypted (NIP-04) so relays cannot read
them. The format is intentionally small: kind 23194 for requests, kind
23195 for responses, kind 13194 for capability advertisement.

## Walkthrough / mechanics

Three event kinds:

| kind  | author        | direction       | purpose                       |
|-------|---------------|-----------------|-------------------------------|
| 13194 | wallet        | published once  | advertise supported methods   |
| 23194 | app (client)  | request         | encrypted JSON-RPC request    |
| 23195 | wallet        | response        | encrypted JSON-RPC response   |
| 23196 | wallet        | notification    | encrypted async event (NIP-47.1) |

### Kind 13194 (info)

Public, unencrypted, content is a space-separated list of method names:

```json
{
  "kind": 13194,
  "pubkey": "<wallet_service_pubkey>",
  "created_at": 1714400000,
  "tags": [["notifications", "payment_received payment_sent"]],
  "content": "pay_invoice get_balance make_invoice lookup_invoice list_transactions",
  "sig": "..."
}
```

### Kind 23194 (request)

```json
{
  "kind": 23194,
  "pubkey": "<app_pubkey>",
  "created_at": 1714400123,
  "tags": [["p", "<wallet_service_pubkey>"]],
  "content": "<NIP-04 encrypted JSON>",
  "sig": "..."
}
```

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
  "content": "<NIP-04 encrypted JSON>",
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
`QUOTA_EXCEEDED`, `RESTRICTED`, `UNAUTHORIZED`, `INTERNAL`, `OTHER`.

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
  "tags": [["p","b3c0a8...e5d1"]],
  "content": "fGh4...==",  // NIP-04(secret, b3c0a8...).encrypt(
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
- 23196 notifications are optional; clients that subscribe blindly may
  miss support and silently degrade.
- NIP-04 is being deprecated in favor of NIP-44 (better AEAD); some
  modern wallets prefer NIP-44 inside NWC. Negotiation is via 13194's
  `tags` (`["encryption", "nip44_v2 nip04"]`).
- Per-relay rate limits: high-volume apps must use multiple relays
  declared in the URI's `relay` query param.

## References

- NIP-47: https://github.com/nostr-protocol/nips/blob/master/47.md
- NIP-04 encryption: https://github.com/nostr-protocol/nips/blob/master/04.md
- NIP-44 encryption: https://github.com/nostr-protocol/nips/blob/master/44.md
- NWC reference SDK: https://github.com/getAlby/js-sdk
