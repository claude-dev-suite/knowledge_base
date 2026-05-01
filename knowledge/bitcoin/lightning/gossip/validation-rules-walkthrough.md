# Lightning Gossip Validation Rules Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/gossip`.
> Canonical source: https://github.com/lightning/bolts/blob/master/07-routing-gossip.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/gossip/SKILL.md

## Concept

BOLT-07 specifies the **gossip protocol** by which Lightning nodes
exchange channel and node announcements forming the public network
graph. Each gossip message has strict validation rules: signature
checks, on-chain proof checks, and rate limits. Without these, anyone
could pollute the graph with fake channels and misroute payments.

## Walkthrough / mechanics

### Gossip messages

| Type | Name | Purpose |
|------|------|---------|
| 256 | channel_announcement | Announce new public channel |
| 257 | node_announcement | Node alias, addresses, features |
| 258 | channel_update | Edge weights (fees, CLTV) |
| 259 | gossip_timestamp_filter | Set per-peer gossip filter |
| 261-264 | query_short_channel_ids, etc. | Resync queries |

### channel_announcement validation

A `channel_announcement` includes:

- `short_channel_id` (block, tx_index, output_index of funding tx).
- `node_id_1`, `node_id_2` (channel peers).
- `bitcoin_key_1`, `bitcoin_key_2` (channel funding pubkeys).
- 4 signatures: each node + each bitcoin key.

Validation:

1. Verify all 4 signatures over the announcement message.
2. Look up `short_channel_id` on chain.
3. Verify the funding output is a P2WSH (or P2TR) of `bitcoin_key_1 +
   bitcoin_key_2` 2-of-2 multisig.
4. Verify the output is unspent.

If any check fails -> reject. The on-chain proof prevents fake-channel
spam.

### channel_update validation

- Signed by one of the channel's `node_id_X` keys.
- Sequence number > previous update for same direction.
- Timestamp within reasonable bounds (~2 weeks max).
- Fee fields within reasonable bounds.

### node_announcement validation

- Signed by `node_id`.
- Timestamp > previous node_announcement from same node.
- Address format validated (IPv4, IPv6, Tor, DNS).

### Rate limits

Per BOLT-07 §7.5:

- channel_update: max 1 per channel per direction per ~10 minutes.
- node_announcement: max 1 per node per ~hour.
- Spammers' future updates ignored.

### Graph propagation

Each peer maintains a "set of known channels" plus per-channel
timestamp. On new gossip:

1. Validate.
2. If newer than what we have: store, forward to other peers.
3. If older: drop.

### Stale channels

If `channel_update` not seen for 14 days, channel marked stale.
After 6 weeks, removed from graph. Force-closed channels (funding
output spent) detected via blockchain monitoring.

### Privacy: BOLT-12 blinded routes

Public gossip exposes channel topology. BOLT-12's blinded routes
sidestep this for receivers; routing nodes still gossip publicly.

## Worked example

Bob sees a new `channel_announcement` from peer Carol:

```
short_channel_id: 850000x42x0
node_id_1: 03Bob_pk
node_id_2: 03Charlie_pk
bitcoin_key_1: 03Bob_funding_pk
bitcoin_key_2: 03Charlie_funding_pk
signatures: <4 sigs>
```

Bob validates:

1. ✓ All 4 sigs match.
2. ✓ Looks up tx at block 850000, tx_index 42.
3. ✓ Output 0 of that tx is P2WSH of 2-of-2(03Bob_funding_pk,
   03Charlie_funding_pk).
4. ✓ Output unspent.

All pass: Bob adds the edge to his graph. Forwards to other peers.

Later, an attacker tries to inject:

```
short_channel_id: 850000x42x0
node_id_1: 03Eve_pk     <- different from Bob
node_id_2: 03Mallory_pk <- different from Charlie
signatures: <4 sigs>
```

Bob validates: signatures might match Eve+Mallory's keys, but the
on-chain output (lookup) shows the funding pubkeys are Bob+Charlie's.
Reject.

## Common pitfalls

- **Forwarding without validation**: a node that forwards unverified
  gossip helps spam propagate. All implementations MUST validate.
- **Pruning**: store all-time gossip = unbounded growth. Implement
  staleness pruning (14d / 6w windows).
- **Replay attacks**: gossip messages have timestamps; reject older
  duplicates.
- **DoS via valid spam**: attacker rapidly opens/closes channels to
  generate valid announcements. Mitigated by rate limits + cost of
  channel opens.
- **Blinded route leak via gossip**: well-known route hint nodes can
  be deanonymized; BOLT-12 mitigates partially.

## References

- BOLT-07 § 7.4-7.6 (gossip validation).
- LND `discovery` package (channel announcement validation).
- CLN `gossipd` daemon.
- Lightning Network Daemon paper.
