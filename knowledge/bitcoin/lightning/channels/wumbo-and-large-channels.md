# Wumbo and Large Channels - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/channels`.
> Canonical source: BOLT 2 (`option_support_large_channel`, bits 18/19)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/channels/SKILL.md

## Concept

The original BOLT 2 capped a channel at `2^24 - 1 = 16,777,215 sats`
(~0.168 BTC). `option_support_large_channel` (informally "wumbo") removes
this cap on the wire, but raises practical questions: how big can a
channel actually be, what HTLC limits scale with size, what dust
thresholds apply, and how does fee-rate volatility on a large channel
behave during force-close. This article covers those second-order
effects.

## Walkthrough / mechanics

The cap was a sanity check, not a protocol invariant. With wumbo, the
new bounds are:

- `funding_satoshis`: u64, but channel funding output must fit a 21M BTC
  Bitcoin tx. Practical cap: any value <= total UTXO supply.
- `max_htlc_value_in_flight_msat`: u64 msat (advertised in
  `accept_channel`). Implementations cap this typically at min(channel
  capacity, 100k sats) by default, raise via config.
- `htlc_minimum_msat`: per channel, advertised in `channel_update`.
- `max_accepted_htlcs`: still capped at 483 by BOLT 2 (this is *not*
  raised by wumbo).
- `dust_limit_satoshis`: each side's own. HTLCs below the larger of the
  two parties' dust limits become "trimmed" (not on commitment, paid as
  fees on force-close).

The 483-HTLC limit is a hard cap because of commitment-tx size: at 483
HTLCs the commitment tx approaches Bitcoin's standardness limits
(~100kB). Wumbo channels with high HTLC churn still hit this.

Fee rate dynamics on a large channel:

- Force-close fee for a 100M-sat channel at 50 sat/vB: roughly 600-900
  sats with anchors. Trivial.
- But: each HTLC adds ~172 vbytes. At 483 HTLCs and 50 sat/vB the
  commitment alone costs ~415,000 sats just to bump.
- `update_fee` is the only way to change the on-chain fee for the
  pre-signed commitment. The funder is responsible.

Capacity reservation rules (BOLT 2 "channel reserve"):

```
channel_reserve_satoshis >= max(dust_limit_satoshis, capacity / 100)
```

A 1 BTC channel has a 1M-sat reserve per side by default. This locks
2M sats unusable for routing; relevant for liquidity planning on big
channels.

## Worked example

Open a 1 BTC channel from LND to a wumbo-capable peer:

```
lncli openchannel \
    --node_key 03abc... \
    --local_amt 100000000 \
    --push_amt 0 \
    --sat_per_vbyte 25 \
    --remote_csv_delay 720 \
    --max_local_csv 1024
```

If the peer is wumbo-capable, LND advertises bit 19 in init. CLN
`fundchannel` requires `--large` flag (or `large-channels` feature in
`config`). LDK accepts wumbo by default if `accept_inbound_channels` is
true.

Inspecting on-chain: a 1 BTC anchor commitment without HTLCs:

```
inputs:  funding_outpoint  (1 input, ~157 vbytes)
outputs: to_local           (~43 vbytes)
         to_remote          (~31 vbytes, P2WPKH)
         to_local_anchor    (~43 vbytes)
         to_remote_anchor   (~43 vbytes)
total:   ~317 vbytes
```

At 50 sat/vB feerate, force-close costs 15,850 sats — under 0.02% of
channel capacity. At 483 HTLCs, the same tx is ~83,000 vbytes and
costs ~4.15M sats: noticeable.

CLN cleanup on a stuck wumbo force-close:
```
lightning-cli close <peer_id> 600 unilateraltimeout=1
lightning-cli bumpchannelopen <txid>:<vout> 100  # bump anchor CPFP
```

## Common bugs / pitfalls

- **HTLC throughput cap**: a 5 BTC wumbo channel cannot route 500 HTLCs
  in flight. The 483 limit is hit first.
- **`max_htlc_value_in_flight_msat` defaults**: some implementations
  default to 10% of capacity. On a 1 BTC channel, 10M sats in-flight is
  fine, but on a 10 BTC channel 10% means 100M sats — at average HTLC
  size 100k sats, only ~1000 HTLCs (still capped at 483 anyway).
- **Channel reserve trap**: opening a 1 BTC channel and immediately
  pushing 99M sats fails because 1M reserve must remain on the local
  side.
- **Fee bumping during high mempool**: force-close on 1 BTC channel with
  50 HTLCs needs ~400k sats CPFP at 100 sat/vB. Funder must have that
  liquidity *outside* the channel.
- **Trimmed HTLCs on huge channels**: dust limit ~330 sats means HTLCs
  under ~750 sats (anchors variant) are trimmed. On a 10 BTC channel
  routing micropayments, large fractions of in-flight value are trimmed
  and paid as fees on force-close.
- **`update_fee` mismatch on idle channels**: if mempool spikes while
  channel is idle, the commitment fee may be far below market rate. A
  force-close at that moment can stick in mempool indefinitely without
  CPFP via anchors.

## References

- BOLT 2 channel reserve: https://github.com/lightning/bolts/blob/master/02-peer-protocol.md#requirements-3
- BOLT 9 bits 18/19: https://github.com/lightning/bolts/blob/master/09-features.md
- `option_support_large_channel` discussion: https://github.com/lightning/bolts/pull/596
- LND wumbo flags: `lncli openchannel --help`
- CLN large-channels config: `man lightningd-config`
