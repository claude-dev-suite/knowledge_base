# Zero-Conf Channels - Tradeoffs Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/channels`.
> Canonical source: BOLT 2 (`option_zeroconf` bits 38/39, `option_scid_alias` bits 40/41)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/channels/SKILL.md

## Concept

A zero-conf channel skips Bitcoin's confirmation layer: peers exchange
`channel_ready` immediately after `funding_signed`, without waiting for
any block confirmations. It is the engine behind LSP "JIT channels"
(channel opens during the first received payment) and instant on-boarding
for mobile wallets. The price: the receiver of the channel takes on full
counterparty risk that the funder never broadcasts the funding tx, or
that a mempool reorg invalidates it.

## Walkthrough / mechanics

Pre-zeroconf, the funding flow waits `min_depth` blocks (typically 3-6).
With `option_zeroconf` negotiated:

```
A -> B: funding_signed
B -> A: channel_ready (with scid_alias)
A -> B: channel_ready (with scid_alias)
```

`scid_alias` (BOLT 2 + BOLT 9 bits 40/41) is the missing piece: a
short_channel_id is normally derived from `block_height|tx_index|output_index`,
which doesn't exist yet. The alias is a synthetic SCID chosen by the
peer in the format `0xfe<random>` (the high bits avoid collision with
real SCIDs whose block heights are realistic).

Aliases serve two purposes:
1. Pre-confirmation routing: HTLCs flow over the alias as if it were a
   real SCID.
2. Permanent privacy aliases: even after confirmation, the alias can
   continue to be used to hide the real SCID in routing hints.

Once the funding tx confirms, the *real* SCID is also valid; both
identify the same channel. `channel_update` may use either.

Risk model:

- **Funder of the channel** has no risk pre-confirm: they sent funds
  into a 2-of-2 they control half of, and they hold the latest commitment
  to recover.
- **Counterparty (typically receiver)** trusts the funder to broadcast
  and not double-spend. If the funder broadcasts a conflicting tx
  spending one of the funding inputs, the channel is dead and any HTLC
  flowing INTO the receiver during the unconfirmed window is lost.

The `option_zeroconf` channel MUST be considered "non-standard" for
gossip purposes — never announced via `channel_announcement` until
confirmed (privacy + soundness).

## Worked example

LSP-flow (Phoenix-style JIT channel):

```
1. User opens Phoenix wallet, generates BOLT 11 invoice for 50,000 sats.
2. Sender pays to ACINQ's LSP node.
3. ACINQ forwards via a brand-new zero-conf channel to user.
   - LSP funds 100,000 sats, takes 50,000 as channel-open fee from the
     incoming HTLC.
   - tx_signatures exchanged, channel_ready with scid_alias sent.
4. User's wallet receives update_add_htlc on the freshly opened
   channel, fulfills with preimage.
5. LSP broadcasts funding tx with normal RBF policy.
6. ~6 blocks later: real SCID becomes valid; alias still used for
   privacy.
```

CLN configuration to accept zero-conf from a specific peer (plugin):
```c
plugin_option(p, "zero-conf-peer", "string",
              "Allow zero-conf channels from this nodeid",
              json_set_string, &p->zeroconf_peer);
```

LND dynamic gating via `accept_channel` interceptor:
```go
acceptor := lnrpc.ChannelAcceptResponse{
    Accept:        true,
    ZeroConf:      true,
    UpfrontShutdown: "",
    MinAcceptDepth: 0,
}
```

LDK config:
```rust
let mut config = UserConfig::default();
config.manually_accept_inbound_channels = true;
// Then in event handler accept with zero_conf = true
event_handler.handle_open_channel_request(... ZERO_CONF);
```

## Common bugs / pitfalls

- **Funder double-spend**: the only mitigation is to whitelist trusted
  funders. Public LSPs may post a bond or rely on reputation.
- **Mempool eviction**: low-fee funding tx evicted by RBF replacement
  → channel never confirms. Receiver's HTLCs are lost. Fee should match
  market or use anchor CPFP.
- **Alias collision**: spec mandates random alias generation. A predictable
  algorithm could collide with a real SCID. Use 64-bit random with
  block_height field set to 0x00 to disambiguate.
- **Privacy leakage on confirmation**: many implementations use real
  SCID after confirmation, defeating the alias's privacy benefit. To
  preserve privacy, *always* route via alias even after confirm and
  never gossip-announce.
- **`channel_announcement` accidentally sent**: pre-confirm or aliased
  channels announced to gossip break BOLT 7 invariants (block_height
  refers to a non-existent block). Other nodes will reject the
  announcement and may downgrade your reputation.
- **Reorg after channel_ready**: a 1-block reorg invalidating the
  funding tx can lock the channel into a "ready but no on-chain output"
  state. Spec recommends rolling back to "awaiting_funding" if reorg
  depth exceeds your `min_depth`. Most implementations simply error and
  request manual recovery.
- **HTLC stuck after failed open**: HTLCs forwarded over the alias
  before confirm cannot be recovered on chain — they exist only in
  off-chain commitment that has no on-chain anchor.

## References

- BOLT 2 channel_ready: https://github.com/lightning/bolts/blob/master/02-peer-protocol.md#the-channel_ready-message
- BOLT 9 bits 38/39 + 40/41: https://github.com/lightning/bolts/blob/master/09-features.md
- Zero-conf PR (BOLT): https://github.com/lightning/bolts/pull/910
- Phoenix on-the-fly channel design: https://acinq.co/blog/phoenix-splicing-update
- LDK zero-conf example: `lightning/src/util/config.rs` (`UserChannelConfig::manually_accept_inbound_channels`)
