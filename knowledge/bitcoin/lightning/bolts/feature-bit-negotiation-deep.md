# Feature Bit Negotiation - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/bolts`.
> Canonical source: BOLT 9 (https://github.com/lightning/bolts/blob/master/09-features.md)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/bolts/SKILL.md

## Concept

The quick-ref enumerates feature bits. This article is about the *negotiation
algorithm* itself: when bits are checked, what "context" they live in, and how
upgrades roll out without a flag day. Lightning has three negotiation contexts
(init, node_announcement, BOLT 11 invoice), and each treats bit semantics
slightly differently. Getting this wrong causes subtle interop bugs that
appear only during partial network upgrades.

## Walkthrough / mechanics

Three contexts (BOLT 9 "Context"):

| Context         | Carrier                | Time of evaluation         |
|-----------------|------------------------|----------------------------|
| `I` init        | `init.features` TLV    | Per-connection             |
| `N` node_ann    | `node_announcement`    | Network-wide gossip        |
| `9` invoice     | BOLT 11 `9` tag        | Per-payment, sender-side   |
| `B` blinded     | route-blinding TLV     | Per-route, opaque to mids  |

Init exchange (per BOLT 1 section "The init Message"):

```
A -> B: init { features: F_A, networks: [bitcoin] }
B -> A: init { features: F_B, networks: [bitcoin] }
```

Each side then computes:

```
required_by_peer = peer.features & EVEN_BITS_MASK
locally_unsupported = required_by_peer & ~self.features
if locally_unsupported != 0:
    send error; close connection
```

Only after this check do *any* other messages flow. The first non-init
message on the connection is therefore conditioned on full feature
agreement.

Bits come in pairs: `2k` is "required", `2k+1` is "optional". A node
supporting feature X advertises EITHER `2k+1` or `2k` (never both). When a
peer advertises `2k`, you MUST close if you don't have it. When `2k+1`, you
MAY use the feature if you also support it.

Compatibility upgrades use the odd bit first. Once a feature is universal
the spec can flip the recommendation to require the even bit, but that's a
multi-year process. `var_onion_optin` (bits 8/9) crossed this threshold in
2021 and is now de-facto required.

## Worked example

Real init from CLN 24.05 to LDK 0.0.124:

```
init
  features: 0x000000088a525a1ea
  networks_tlv: { chain_hashes: [bitcoin_genesis] }
  remote_addr_tlv: { addr: 198.51.100.5:9735 }
```

Decoding bits set (LSB first per BOLT 1 "Bit Numbering"):
```
bit 1   option_data_loss_protect (odd)
bit 7   gossip_queries (odd)
bit 9   var_onion_optin (odd)
bit 13  static_remotekey (odd)
bit 17  payment_secret (odd)
bit 19  basic_mpp (odd)
bit 21  option_anchors_zero_fee_htlc_tx (odd)
bit 27  option_shutdown_anysegwit (odd)
bit 45  option_route_blinding (odd)
bit 51  option_keysend (odd)
```

LDK responds with a partially overlapping set. Both sides take the
intersection. Notably, neither asserts an even bit — this is typical:
even bits are reserved for hard-required protocol breaks, and current
production traffic uses only odd bits.

Probing peer features non-destructively (LND CLI):
```
lncli getnodeinfo --include_channels=false <pubkey> | jq '.node.features'
```

CLN equivalent reads from gossip:
```
lightning-cli listnodes <pubkey> | jq '.nodes[0].features'
```

## Common bugs / pitfalls

- **Setting both bit `2k` and `2k+1`**: BOLT 9 says you MUST NOT. Some
  early eclair builds did, causing CLN to log a `BADMSG_FEATURES` and
  disconnect.
- **Required-bit creep**: setting `option_static_remotekey` as required
  (bit 12) before checking peer support breaks compatibility with old
  mobile wallets in the wild.
- **Invoice context (`9`)**: bits in the BOLT 11 `9` tag tell the SENDER
  what to use, not the peer. `payment_secret` (bit 14/15) here means the
  invoice contains an `s` tag; the bit is information-carrying, not an
  init-style precondition.
- **Forgetting `gossip_queries` semantics**: peer that advertises this
  bit MUST send `query_channel_range` instead of full table dump. Older
  LND versions ignored the bit and did both, blowing peer bandwidth.
- **Odd-bit gating**: features behind an odd bit MUST still validate
  cleanly when peer doesn't have it. If you send a TLV that's only
  legal under feature X, you must check `negotiated(X)` not just
  `local_supports(X)`.
- **`networks` TLV intersection**: if you support [bitcoin, testnet]
  and peer only [bitcoin], you continue. If peer sends an empty/missing
  `networks` TLV, behavior is implementation-defined (BOLT 1 says
  default to bitcoin mainnet but some impls disconnect).

## References

- BOLT 9 - Feature flag registry: https://github.com/lightning/bolts/blob/master/09-features.md
- BOLT 1 - init negotiation: https://github.com/lightning/bolts/blob/master/01-messaging.md#the-init-message
- LDK feature handling: `lightning/src/ln/features.rs`
- CLN feature negotiation: `lightning/common/features.c`
- LND `feature.Manager`: `github.com/lightningnetwork/lnd/feature`
