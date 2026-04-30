# In-Flight HTLC Limits and Channel Jamming - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/htlcs`.
> Canonical source: BOLT 2 ("max_accepted_htlcs", "max_htlc_value_in_flight_msat"), Lightning-dev jamming threads
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/htlcs/SKILL.md

## Concept

Each Lightning channel has hard limits on how many HTLCs can be in flight
simultaneously and how much value those HTLCs can carry. Those limits
exist for safety (commitment tx size, reserve protection) but they create
the surface for *channel jamming* attacks: a malicious sender holds
HTLCs on a victim channel without resolving them, denying service to
honest users without paying any fees (HTLCs that fail are free). This
article details the limits, the attack class, and current mitigation
research.

## Walkthrough / mechanics

Per-channel limits exchanged in `open_channel` / `accept_channel`:

| Field                           | Default        | Hard cap                |
|---------------------------------|----------------|-------------------------|
| `max_accepted_htlcs`            | 30-483         | 483 (BOLT 2)            |
| `max_htlc_value_in_flight_msat` | 10% capacity   | u64                     |
| `htlc_minimum_msat`             | 1000           | non-negative            |
| `dust_limit_satoshis`           | 354 (anchors)  | network policy          |
| `channel_reserve_satoshis`      | 1% capacity    | min dust limit          |

The 483 cap exists because the worst-case commitment tx (483 HTLCs +
to_local + to_remote + 2 anchors + worst-case witnesses) just fits
inside Bitcoin's 100k vbyte standardness limit.

Two jamming variants:

**Slow jamming**: attacker sends 483 HTLCs of any size, holds them
near CLTV expiry (~2 weeks). Channel cannot accept more HTLCs. Attack
costs nothing because failed HTLCs pay no fee.

**Fast jamming**: attacker rapidly fills the in-flight value, then
fails them, repeats. Each cycle is fast enough to keep the channel
saturated; honest payments hit `temporary_channel_failure`.

A single attacker controlling two nodes can saturate the channel
between them at zero cost: open 1 BTC channel, send 483 HTLCs from A
to B, hold for 2 weeks, fail. Repeat with new channel.

Mitigations under discussion (lightning-dev 2022-2024):

1. **Upfront fees**: charge a small fee per HTLC at attempt time, paid
   regardless of outcome. Breaks free jamming.
2. **Reputation tokens**: peer issues tokens to known-good senders;
   spend a token to send an HTLC. Drains under attack.
3. **Local reputation + bucketing**: split `max_accepted_htlcs` across
   incoming peers proportional to their historical good behavior.
4. **`hodl invoices`**: orthogonal — these intentionally hold HTLCs.
   Distinguish friendly vs adversarial holds via reputation.

LND 0.18+ ships "outgoing channel reputation" buckets that demote
peers with poor settlement ratios.

## Worked example

Attacker `A` jams victim hub `V`:

```
1. A opens 1M-sat channel to V.
2. A sends 483 HTLCs of 1000 sats each from A1 -> V -> A2 (A controls
   both endpoints). Total in flight: 483_000 sat.
3. Each HTLC has cltv_expiry 14 days from now.
4. V's incoming channel from A1 has 483 HTLCs locked. V cannot accept
   more HTLCs from A1 on that channel.
5. A holds. After 14 days minus epsilon, A sends update_fail_htlc on
   each. No fees paid.
6. Repeat with a new channel.

Cost to A: channel-open fee (one on-chain tx) + tied-up capital (zero
yield, but principal recovered).
Cost to V: channel utilization 0% for 14 days, lost forwarding revenue.
```

LND command to inspect current in-flight pressure:
```
lncli listchannels --pending_only=false | jq '.channels[] |
    select(.pending_htlcs | length > 50) |
    {alias: .peer_alias, in_flight: .pending_htlcs | length}'
```

CLN equivalent:
```
lightning-cli listpeers | jq '.peers[].channels[] |
    select((.htlcs // []) | length > 50)'
```

LND outgoing-channel reputation snapshot (0.18+):
```
lncli listchannels | jq '.channels[].peer_alias as $a |
    .channels[] | "\($a): \(.scid_alias) up=\(.uptime)"'
```

## Common bugs / pitfalls

- **Setting `max_accepted_htlcs = 483` without rate limiting**: this
  is the default but means *all* slots are vulnerable to one peer.
  Real defenses bucket per-peer.
- **Treating jamming as "low priority"**: it is the most-cited reason
  large hubs avoid forwarding for unknown peers; failure to address
  caps Lightning growth.
- **Confusing trimmed HTLCs with in-flight count**: trimmed HTLCs
  (below dust threshold) DON'T appear in the commitment but still count
  toward `max_accepted_htlcs`. An attacker exclusively using dust HTLCs
  jams just as effectively.
- **HodlInvoice handling**: `lncli sendpayment --keysend` plus a hodl
  invoice creates a friendly hold. Distinguish via per-peer reputation
  rather than blanket-failing held HTLCs.
- **`max_htlc_value_in_flight_msat` set too tight**: some operators
  cap at 1% of channel; honest MPP payments split into >100 parts get
  rejected.
- **Reserve reuse during jamming**: the reserve doesn't protect against
  in-flight slot exhaustion. A jammed channel still carries the
  reserve; attackers don't need to drain it.
- **Channel reset cost**: closing a jammed channel costs an on-chain
  tx + 2 weeks for HTLC timeouts (force-close pathway). Can be more
  expensive than enduring the jam.

## References

- BOLT 2 in-flight limits: https://github.com/lightning/bolts/blob/master/02-peer-protocol.md#requirements-3
- "Channel jamming attacks" (Mizrahi, Tsabary, 2021): https://arxiv.org/abs/2002.06564
- Lightning-dev unjamming proposal (Riard, 2022): https://gnusha.org/url/https://lists.linuxfoundation.org/pipermail/lightning-dev/2022-October/003724.html
- Carla Kirk-Cohen et al, Local Reputation: https://github.com/ClaraShk/LNJamming
- LND outgoing reputation: `htlcswitch/reputation`
