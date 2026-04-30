# BOLT Wire Format - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/bolts`.
> Canonical source: BOLT 1 (https://github.com/lightning/bolts/blob/master/01-messaging.md)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/bolts/SKILL.md

## Concept

Every Lightning peer-to-peer message is a length-prefixed binary blob carried
inside a Noise (BOLT 8) framed packet. BOLT 1 specifies how those bytes are
laid out: a 2-byte big-endian message type, followed by a payload composed of
fixed-width fundamental types and an optional TLV extension stream. This
contract is what lets four independent implementations (LND, CLN, LDK,
Eclair) interop with bit-exact precision.

## Walkthrough / mechanics

A raw decrypted Lightning message looks like:

```
+----------+-----------------+-------------------+
| 2 bytes  | N bytes         | M bytes (TLV)     |
| msg type | typed payload   | extension stream  |
+----------+-----------------+-------------------+
```

Fundamental types (BOLT 1 section "Fundamental Types"):

| Type            | Size       | Notes                                  |
|-----------------|------------|----------------------------------------|
| `byte`          | 1          |                                        |
| `u16`/`u32`/`u64` | 2/4/8    | big-endian                             |
| `chain_hash`    | 32         | block hash of genesis                  |
| `channel_id`    | 32         | derived from funding outpoint          |
| `sha256`        | 32         |                                        |
| `signature`     | 64         | compact secp256k1                      |
| `point`         | 33         | compressed secp256k1                   |
| `short_channel_id` | 8       | block(3) | tx-index(3) | output(2)     |
| `bigsize`       | 1/3/5/9    | varint a-la BIP-21 but big-endian      |

A TLV extension stream is `bigsize type | bigsize length | length bytes value`,
ordered strictly ascending by type. Even types are mandatory (peer MUST
understand); odd types are discardable. Unknown odd TLVs MUST be silently
skipped, never cause disconnect.

## Worked example

Decoding an `update_add_htlc` (type 128) on the wire (Noise-decrypted):

```
00 80                                              # type = 128
aa bb cc dd ee ff 00 11 22 33 44 55 66 77 88 99
aa bb cc dd ee ff 00 11 22 33 44 55 66 77 88 99    # channel_id (32B)
00 00 00 00 00 00 00 05                            # id (u64) = 5
00 00 00 00 00 0f 42 40                            # amount_msat = 1_000_000
ab cd ... (32 bytes)                               # payment_hash
00 0a 1b 2c                                        # cltv_expiry = 0x000a1b2c
01 02 03 ... (1366 bytes)                          # onion_routing_packet
fd 00 fe 41                                        # TLV: type=254, length=...
...                                                # blinding_point TLV (BOLT 4)
```

The trailing `fd 00 fe` decodes as bigsize 254 (`option_route_blinding`
extension carrying the `blinding_point`). A peer without route-blinding
support must still parse past it because the type is even but the spec
gates its presence behind the negotiated feature bit.

In Go (LND `lnwire/update_add_htlc.go`) this is parsed by reading fixed
fields then calling `tlvStream.Decode(r)`. CLN's `wire/peer_wire.csv`
generates parser code from a CSV schema; a wire-format bug in 2022
caused CLN <0.11 to mis-order TLV records and disconnect from LDK peers.

## Common bugs / pitfalls

- **Endianness**: every multi-byte integer is big-endian. Hosts that copy
  `u64` directly from memory (little-endian on x86) corrupt the message.
- **TLV ordering**: types MUST be strictly ascending. Duplicate types or
  out-of-order types are a `must-fail-channel` error per BOLT 1.
- **Unknown even TLV types**: receiver MUST close the connection. Many
  implementers wrongly fall through and keep going — this hides interop
  drift until production.
- **Length-prefix mismatch**: the Noise frame length must equal the inner
  message length exactly. Trailing garbage is rejected by BOLT 8.
- **`bigsize` vs Bitcoin `varint`**: Lightning's `bigsize` is big-endian;
  Bitcoin's `varint` is little-endian. Mixing them is the single most
  common parser bug in new implementations.
- **Wire type collisions**: BLIPs reserve experimental ranges, but the
  spec is silent on type squatting. Two BLIPs colliding at type 47821
  was real (BLIP-3 vs BLIP-7 draft).

## References

- BOLT 1 - Base Protocol: https://github.com/lightning/bolts/blob/master/01-messaging.md
- BOLT 8 - Noise transport: https://github.com/lightning/bolts/blob/master/08-transport.md
- LND wire codecs: `github.com/lightningnetwork/lnd/lnwire`
- CLN wire CSV: `lightning/wire/peer_wire.csv`
- LDK reader: `lightning/src/ln/wire.rs`
