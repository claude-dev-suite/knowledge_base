# Keysend Custom Records Conventions - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/keysend`.
> Canonical source: https://github.com/satoshisstream/satoshis.stream/blob/main/TLV_registry.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/keysend/SKILL.md

## Concept

Keysend onions can carry arbitrary TLVs in the final-hop payload.
A loose ecosystem convention has emerged around specific TLV types
for application metadata: chat messages, podcast Boost-grams, Nostr
zap details, sender identity, replies, etc. The `satoshisstream/TLV_registry`
repo serves as the de-facto registry.

## Walkthrough / mechanics

### Common TLV types

| TLV type (decimal) | TLV (hex) | Purpose | Format |
|--------------------|-----------|---------|--------|
| 5482373484 | 0x14668DAC | keysend_preimage | 32-byte preimage |
| 7629169 | 0x747FBC1 | sphinx-chat-msg | UTF-8 message |
| 34349334 | 0x20C81FF6 | sphinx-chat-msg-v2 | UTF-8 message |
| 7629171 | 0x747FBC3 | sphinx-chat-sig | sig over msg |
| 5482373483 | 0x14668DAB | sphinx public key | sender pubkey |
| 696969 | 0x0AA169 | podcast_boost_metadata | JSON (podcast info) |
| 133773310 | 0x07F8C3FE | nostr-zap-request | Nostr event JSON |

### Podcast 2.0 boost-grams (TLV 696969)

A "boost-gram" is a Lightning payment that pays the host of a podcast
plus optional split recipients. Metadata in TLV 696969:

```json
{
  "podcast": "Joe Rogan Experience",
  "feedID": 12345,
  "episode": "JRE #1234",
  "guid": "abc-uuid",
  "ts": 4920,        // timestamp in podcast (seconds)
  "value_msat_total": 100000,
  "name": "Bob",
  "sender_name": "$alice@strike.me",
  "message": "Great episode, here's a boost!",
  "app_name": "Fountain",
  "app_version": "2.1.0",
  "action": "boost"
}
```

Podcast hosts implement value-splits: the boost-gram is sent to the
host, who splits the sats per percentage to writer, producer, etc.,
according to the podcast feed's <podcast:value> tag.

### Nostr zaps

A Nostr zap (NIP-57) is a Lightning payment with a Nostr event embedded:

- TLV 133773310: signed Nostr "kind 9734" event (zap request).
- Receiver verifies signature against sender's npub.
- Receiver publishes a "kind 9735" zap receipt to relays.

This ties Lightning payments to Nostr social graphs.

### Sphinx chat

Sphinx wallet uses TLVs to send encrypted chat messages alongside
keysend payments. Message is encrypted with shared secret derived from
Lightning ECDH. Message length limited by onion payload (~1100 bytes
after other TLVs).

### Encoding rules

- TLV type: variable-length unsigned integer (BigSize encoding).
- TLV length: variable-length.
- Value: opaque bytes.
- Onion final-hop payload max: ~1300 bytes total (BOLT-04 onion).
- Multiple TLVs sorted by ascending type per BOLT-01.

### Custom records vs new BOLTs

The TLV custom-record approach lets ecosystems iterate without BOLT
changes. Downsides:

- No interoperability without out-of-band convention.
- TLV type collisions if registry not respected.
- No type checking; opaque bytes only.

## Worked example

Alice sends 1000 sat boost to Joe Rogan podcast via Fountain app.

```
TLVs in final-hop onion payload:
  type=2 (amt_to_forward): 1000000 msat
  type=4 (outgoing_cltv_value): 800100
  type=5482373484 (keysend_preimage): 0x...32 bytes
  type=696969 (podcast_boost_metadata):
    {
      "podcast": "JRE",
      "episode": "1234",
      "ts": 4920,
      "value_msat_total": 1000000,
      "sender_name": "alice",
      "message": "Loved the bit on Schrodinger's cat",
      "app_name": "Fountain"
    }
```

Joe Rogan's node receives the keysend, displays "1000 sat boost from
alice: 'Loved the bit on Schrodinger's cat'" in his Fountain dashboard.

If JR's RSS feed declares value-splits 80/20 (host/producer), JR's
node forwards 200 sat to producer's pubkey via subsequent keysend
preserving the same metadata TLV.

## Common pitfalls

- **TLV registry not authoritative**: relying on the satoshisstream
  registry; new types appear without coordination.
- **JSON in TLV is bulky**: leaves little room for chat msg or
  multiple metadata. Compact CBOR encoding under consideration.
- **No encryption by default**: TLV values are visible to recipient's
  node operator. Sensitive data should be encrypted before placing.
- **Privacy leak**: sender's pubkey may be deducible from TLV signature
  (e.g. nostr-zap-request signed by sender). If sender wants
  anonymity, omit identity TLVs.
- **Forward compatibility**: receivers should ignore unknown TLV types
  (BOLT-01 even-odd rule); senders SHOULD use odd types for "optional"
  fields.

## References

- satoshisstream/satoshis.stream TLV_registry.md.
- Podcast 2.0 specification (podcastindex.org).
- NIP-57 Lightning Zaps (nostr-protocol/nips).
- BOLT-01 § TLV BigSize encoding.
