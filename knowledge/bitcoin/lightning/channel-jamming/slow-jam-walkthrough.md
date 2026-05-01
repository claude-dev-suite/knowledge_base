# Slow-Jam Channel Jamming Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/channel-jamming`.
> Canonical source: https://blog.bitmex.com/lightning-network-attacks-channel-jamming/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/channel-jamming/SKILL.md

## Concept

Channel jamming is a Lightning DoS attack where the attacker deliberately
opens HTLCs and never settles, locking up channel slots and capacity for
the duration of the HTLCs' CLTV expiry. **Slow jamming** is the
high-value variant: attacker uses long-CLTV HTLCs to lock channel capacity
for hours or days, while paying minimal cost (only routing fees on
failure).

This contrasts with **fast jamming**: many small short-CLTV HTLCs that
exhaust the per-channel HTLC slot count (typically 483 max in-flight).

## Walkthrough / mechanics

### Lightning channel slot limits

Each channel has BOLT-02 imposed limits:

- `max_accepted_htlcs`: 483 in-flight HTLCs per direction (default).
- `max_htlc_value_in_flight_msat`: total msat in flight.
- Capacity limit: total channel balance.

An attacker can saturate any of these to cause failures for honest
users.

### Slow-jam construction

The attacker:

1. Picks a target channel they want to jam (e.g. a busy routing node).
2. Sends an HTLC through the route via that channel with very long
   CLTV (e.g. 144 blocks ≈ 24h).
3. Lightning protocol holds the HTLC pending until preimage or timeout.
4. Attacker is the LAST hop and never reveals preimage; HTLC stays
   pending until CLTV expires.
5. After timeout, HTLC fails; the attacker pays no fee (routing fees
   on Lightning are charged on success only).

Repeat until the channel's slots or capacity are exhausted.

### Cost analysis

- Capital lockup: attacker doesn't really lock capital because they
  ARE the recipient; they just don't claim. The honest sender (whom
  the attacker controls) loses inbound liquidity for the CLTV duration.
- Fee cost: zero (routing fees only on success).
- Setup cost: minimal — just a node connected to Lightning.

### Slot-saturation impact

If attacker sends 483 HTLCs through a victim's channel each with 144
CLTV, the channel cannot accept any new HTLCs from honest users for
24h. Total capacity locked: 483 * msat per HTLC. Attacker can use
many small HTLCs (each 1 sat) to maximize slot consumption while
minimizing balance lock.

### Capacity-saturation impact

If attacker sends fewer, larger HTLCs (e.g. 100 HTLCs each ~1 % of
channel capacity), they can lock the entire capacity for the CLTV
duration. Honest payments can't route through.

### Multi-channel jam

Attacker constructs a route through multiple victim channels using a
single onion. Each HTLC saturates one slot in each victim. With one
HTLC the attacker can jam multiple channels.

## Worked example

Victim: Bob, a busy routing node with channel to Alice (capacity 1 BTC).

Attacker setup:
- Mallory runs a node with one channel to Alice (or routes via Alice).
- Alice has channel to Bob.

Attack:

```
Mallory sends 483 HTLCs, each 1 sat, through Alice -> Bob -> ... -> Mallory.
Each has CLTV = 144 blocks.
Bob -> Alice channel (Bob's outgoing) has 483 slots taken.
Bob -> Alice channel can't forward new HTLCs from honest users.
```

After 24h:

```
HTLCs all time out, Mallory pays nothing (success-only fees).
Repeat with new batch.
```

Honest user Alice tries to route a payment through Bob, fails repeatedly.
Bob's reputation suffers.

## Common pitfalls

- **Mitigation challenge**: protocol-level mitigations are difficult
  because routes are anonymous; node can't tell jamming from genuine
  long-pending payments.
- **Forwarding fees up-front**: research proposals (Riard, Beslic 2022)
  suggest charging upfront fees for HTLC forwarding to deter jamming
  but adds UX friction.
- **Reputation systems**: nodes can blacklist peers that send many
  failing HTLCs but identification across nodes is difficult (onion
  routing hides source).
- **Slot rotation**: increase max_accepted_htlcs to dilute attacker
  ratio; increases force-close size though.
- **Watchtowers don't help**: this is a liveness attack on routing,
  not on settlement.

## References

- Antoine Riard, Gleb Naumenko. "Channel Jamming Mitigation" 2022.
- BitMEX Research. "Lightning Network Attacks: Channel Jamming" 2020.
- BOLT-02 § HTLC slot limits.
