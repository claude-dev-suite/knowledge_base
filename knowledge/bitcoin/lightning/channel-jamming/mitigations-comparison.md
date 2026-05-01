# Channel Jamming Mitigations - Comparison Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/channel-jamming`.
> Canonical source: https://github.com/lightning/bolts/issues/845 and Riard/Naumenko 2022
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/channel-jamming/SKILL.md

## Concept

Channel jamming is one of Lightning's most stubborn DoS vulnerabilities.
Several mitigation proposals exist; none have been canonically adopted
in BOLT yet (as of 2026). This article compares the leading ideas:
upfront fees, reputation, prepaid forwarding, route blinding, and
slot quotas.

## Walkthrough / mechanics

### Comparison matrix

| Mechanism | Type | Pros | Cons | BOLT status |
|-----------|------|------|------|-------------|
| Upfront forwarding fees | Economic | Direct cost on attacker | Bad UX for normal payments | Proposed |
| Reputation system | Behavioural | Cheap; learns over time | Sybil-vulnerable | Implemented in some apps |
| HTLC endorsement | Cryptographic | Pre-paid auth tokens | Adds protocol complexity | Research |
| Local slot reservation | Operational | Easy to implement | Reduces capacity | Some impls |
| Route blinding (BOLT-12) | Privacy | Limits source visibility | Doesn't directly stop jam | Adopted (BOLT-12) |
| Channel reputation slashing | Economic | Strong deterrent | Requires escrow | Research |

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

### HTLC endorsement (Endorse-HTLCs proposal)

Nodes issue endorsement tokens proving prior good behaviour. To get
slots in a busy channel, you need tokens. Tokens are minted by
forwarding successful HTLCs.

Pros: Sybil-resistant economically.
Cons: bootstrap problem; complex protocol.

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
Layer 1: Local per-peer limits (operator config)
Layer 2: Reputation tracking (peer-level)
Layer 3: Upfront fees on suspicious paths (future)
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
- Pickhardt-Richter "JIT Routing" research.
- BOLT-12 route blinding.
- BitMEX Research jamming studies.
