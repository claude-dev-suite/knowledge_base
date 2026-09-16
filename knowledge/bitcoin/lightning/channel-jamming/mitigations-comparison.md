# Channel Jamming Mitigations - Comparison Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/channel-jamming`.
> Canonical source: https://github.com/lightning/bolts/issues/845 and Riard/Naumenko 2022
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/channel-jamming/SKILL.md

## Concept

Channel jamming is one of Lightning's most stubborn DoS vulnerabilities.
Several mitigation proposals exist; none have been canonically adopted
in BOLT yet (as of September 2026). This article compares the leading
ideas: upfront fees, reputation, prepaid forwarding, route blinding,
slot quotas, and - newest - hold/withhold fees settled by an on-chain
proof-of-message-transfer contract.

## Walkthrough / mechanics

### Comparison matrix

| Mechanism | Type | Pros | Cons | BOLT status |
|-----------|------|------|------|-------------|
| Upfront forwarding fees | Economic | Direct cost on attacker | Bad UX for normal payments | Proposed |
| Reputation system | Behavioural | Cheap; learns over time | Sybil-vulnerable | Implemented in some apps |
| HTLC endorsement tokens (distinct from bLIP-4) | Cryptographic | Pre-paid auth tokens | Adds protocol complexity | Research |
| bLIP-4 `endorsed` signal | Signalling | Cheap sender hint; gathers real-world data | Barred from resource allocation; measures nothing by itself | bLIP-4 Active, not in BOLT |
| Local slot reservation | Operational | Easy to implement | Reduces capacity | Some impls |
| Route blinding (BOLT-12) | Privacy | Limits source visibility | Doesn't directly stop jam | Adopted (BOLT-12) |
| Channel reputation slashing | Economic | Strong deterrent | Requires escrow | Research |
| Withhold fee via CMTC (Riard 2026) | Economic | Prices hold time directly; no consensus change | Unproven crypto + incentives; on-chain only so far | Research |
| Attribution data hold times | Measurement | Verifiable per-hop hold times for the sender | Measures, does not enforce | Adopted (BOLT 4/9, `option_attribution_data`) |

### Upfront fees (Riard 2022)

Each HTLC pre-pays a small fee at each hop, regardless of success.

- Honest user pays for both successful and failed routes.
- Attacker pays for every jam attempt.
- Issue: increases base cost for legitimate users; fee market becomes
  more complex.

### Reputation-based (Lightning Labs work)

A node tracks per-peer success rate of forwarded HTLCs:
- Peer with high success rate -> normal slot allocation.
- Peer with low success rate -> reduced slot allocation.

Pros: invisible to honest users; jammers' future requests degraded.
Cons: Sybil attack (open many channels with different keys); cold-start
problem (new peer has no reputation).

The wire-level signalling piece is **bLIP-4 "Experimental Endorsement
Signaling"** (Carla Kirk-Cohen), listed Active in the `lightning/blips`
repo as of September 2026. It adds an experimental `endorsed` TLV
(type 106823) to `update_add_htlc`: the original sender sets
`endorsed`=7 when it expects the payment to resolve immediately, 0
otherwise, and forwarders relay or re-set it at their discretion. The
bLIP is explicitly a data-gathering experiment - nodes MUST NOT use
`endorsed` in resource-allocation decisions for the duration of the
experiment, whose `experiment_end` is unix 1767225600 (1 January 2026).
As of September 2026 the bLIP still carries `experiment_start`: "TODO:
set once feature bit is widely deployed" (never set) while that
`experiment_end` has already elapsed, and the repo status is unchanged
at Active. The matching BOLT change (`lightning/bolts` PR #1071, "HTLC
Endorsement to Mitigate Channel Jamming") was closed unmerged in August
2025, so endorsement is still outside the BOLTs as of September 2026.

### HTLC endorsement (Endorse-HTLCs proposal)

Nodes issue endorsement tokens proving prior good behaviour. To get
slots in a busy channel, you need tokens. Tokens are minted by
forwarding successful HTLCs.

Pros: Sybil-resistant economically.
Cons: bootstrap problem; complex protocol.

Distinct from the bLIP-4 `endorsed` signal above: that is a 1-byte
sender hint with no token, no minting and no allocation role. As of
September 2026 this token-minting design has no bLIP or BOLT of its
own - the endorsement work that reached the spec repos is bLIP-4.

### Attribution data: the measurement substrate

Reputation and hold fees both need a trustworthy answer to "how long did
each hop hold this HTLC?". That primitive is now specified:
`option_attribution_data` (feature bits 36/37) is in BOLT 9 as of
September 2026. Attribution data travels as TLV type 1 on
`update_fulfill_htlc` and `update_fail_htlc` and carries
`htlc_hold_times` - up to 20 per-hop hold times in units of 100 ms, each
committed to by the reporting hop's HMAC, so the origin node can score
hops on latency and identify the failure source.

It is measurement only: nothing in BOLT 4 charges for a long hold. It is
the substrate the schemes below want.

### Withhold fees via CMTC (Riard, August 2026)

The obstacle to charging for hold time has always been that there is no
universal clock and no way to prove a preimage was delivered at a given
moment. Antoine Riard's **conditional message transfer contract (CMTC)**,
posted to Delving Bitcoin on 7 August 2026, is the first concrete
construction aimed at that obstacle.

A CMTC is a Bitcoin Script construction - adaptor signatures plus
timelocks, no consensus change - that lets two channel counterparties
later prove whether a specific message (e.g. a payment preimage) was
exchanged between them by a given block height, with block height acting
as the universal clock. The counterparties agree on a discrete temporal
window and assign an "oracle-time" adaptor point to each point in it, so
the withhold fee can be settled according to when the message was
actually delivered.

Three proof paths:

| Path | Trigger | Outcome |
|------|---------|---------|
| Message transfer success | Bob delivers the preimage; Alice acknowledges cryptographically | Withhold fee split by delivery time |
| Liveness challenge | Alice is offline and cannot counter-sign | Bob exits, recovers locked funds minus an equilibrium penalty fee |
| Message transfer failure | Bob is offline or never delivers | Alice exits and takes the withhold fee |

Incentive sketch: Alice cannot know when Bob really learned the
decryption key for his adaptor point, so she is pushed to run the
liveness challenge at every temporal point, earning back chunks of the
withhold fee; Bob who skips the challenge forfeits the equilibrium
penalty fee.

Caveats, stated by Riard himself: only an on-chain version is presented,
lifting it into an off-chain channel as extra tapscript leaves is left
to future work, and both the cryptographic correctness and the
cryptoeconomic equilibrium "deserve to be further analyzed". Treat it as
a research prolegomenon, not a deployable design.

### Local quota / per-peer limits

Operator-side: configure max_htlcs per peer, refresh per epoch.
Simple and effective for known-jammer-set defense; doesn't help against
unseen attackers.

### Route blinding (BOLT-12)

Blinded routes hide the destination, preventing precise targeting.
Attacker can still jam the visible route's channels but can't
single-target the recipient.

This is partial mitigation; jamming general routing is still possible.

### Channel reputation with slashing

Each channel open includes a small reputation deposit. Jammer's
deposit slashed if their HTLCs fail at unusual rates.

Effective economic deterrent but introduces escrow / dispute mechanism.

### Combining mitigations

Real-world deployments use several:

```
Layer 0: Attribution data hold times (measurement, BOLT 4)
Layer 1: Local per-peer limits (operator config)
Layer 2: Reputation tracking (peer-level), bLIP-4 endorsement signal
Layer 3: Upfront / withhold fees on suspicious paths (future)
Layer 4: Route blinding for receiver privacy
```

## Worked example: reputation-based throttling

Bob's channel to Alice:
- Total capacity: 1 BTC.
- HTLCs allowed: 483.

Bob tracks peer reputation:
- Alice (peer): 99 % success rate over 10k forwards -> allow full 483 slots.
- New unknown peer Carol opens channel: starts at 50 % allocation -> 240 slots.
- Carol successfully forwards 1000 HTLCs -> reputation rises -> allocation increases.

If Mallory (a jammer) opens a fresh channel:
- Initial allocation: 240 slots.
- After Mallory's first 100 jams, Bob notices low success rate.
- Allocation drops to 50 slots.
- Eventually Mallory is blacklisted.

Cost to Mallory: must repeatedly open new channels to bypass; on-chain
fees + capital reserve.

## Common pitfalls

- **No silver bullet**: each mitigation addresses one vector.
  Combination is necessary.
- **Reputation gaming**: a long-term jammer could behave for months
  to build reputation, then jam.
- **Slot configuration**: too restrictive = legitimate user payments
  fail; too loose = jammer succeeds.
- **Sybil resistance fundamental challenge**: any reputation system
  fights cheap identity creation.
- **Cross-implementation interop**: if some nodes implement reputation
  and others don't, jammers can target the lax ones.

## References

- Riard, Naumenko. "Channel Jamming Mitigation" 2022 lightning-dev.
- Riard, "Conditional Message Transfer Contract To Solve Jamming" (Delving Bitcoin, 7 August 2026): https://delvingbitcoin.org/t/conditional-message-transfer-contract-to-solve-jamming/2772
- bLIP-4 "Experimental Endorsement Signaling": https://github.com/lightning/blips/blob/master/blip-0004.md
- BOLT 4 attribution data: https://github.com/lightning/bolts/blob/master/04-onion-routing.md#returning-errors
- BOLT 9 feature bits 36/37 `option_attribution_data`: https://github.com/lightning/bolts/blob/master/09-features.md
- Pickhardt-Richter "JIT Routing" research.
- BOLT-12 route blinding.
- BitMEX Research jamming studies.
