# Soft-Fork Activation - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/consensus`.
> Canonical source: https://github.com/bitcoin/bips/blob/master/bip-0009.mediawiki, https://github.com/bitcoin/bips/blob/master/bip-0008.mediawiki
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/consensus/SKILL.md

## Concept

A soft fork is a tightening of consensus rules that previously valid blocks may now be rejected, while previously invalid blocks remain invalid. Activation must coordinate the moment when a supermajority of hashpower (and ideally users) begin to enforce the new rule, without splitting the chain. Three deployment frameworks have been used: BIP9 (versionbits), BIP8 (BIP9 + lock-in-on-timeout), and Speedy Trial (BIP9 with a short signaling window). This article walks the state machine and miner-signaling mechanics for each.

## Walkthrough / mechanics

### Versionbits header field

Block `nVersion` is interpreted as a bit field when its top three bits equal `001` (i.e. `0x20000000` mask). Bits 0..28 are usable for parallel deployments. A miner signals "yes" for deployment X by setting the bit for X in `nVersion`.

```
nVersion: 0010 0000 0000 0000 0000 0000 0000 0010
          ^^^                              ^
          top 3 = 001 (versionbits flag)   bit 1 set (e.g. CSV)
```

### BIP9 state machine (per deployment)

```
DEFINED ---- (MTP >= starttime) ---> STARTED
STARTED  ---- (window has >=95% signaling) ---> LOCKED_IN
STARTED  ---- (MTP >= timeout, !signaled)  ---> FAILED
LOCKED_IN ---- (next 2016 boundary)        ---> ACTIVE
ACTIVE  (terminal)
FAILED  (terminal)
```

Transitions are evaluated only at retarget boundaries (every 2016 blocks). BIP9 as written specifies 1916/2016 (95%) on mainnet and 1512/2016 (75%) on testnet, and SegWit used 1916 (`nRuleChangeActivationThreshold` in `chainparams.cpp` at v0.13.1). Taproot's Speedy Trial lowered it to 1815/2016 (90%), which is still the per-deployment mainnet default in Core's `chainparams.cpp` today. Counting is over the `previous` retarget window.

### BIP8 changes

BIP8 replaces the `timeout` with a height-based `endheight` and adds a single bit `LOT` (lockinontimeout):

- `LOT=false`: behaves like BIP9 (FAILED if not signaled).
- `LOT=true`: at endheight, transitions STARTED -> MUST_SIGNAL, and miners must signal until LOCKED_IN; otherwise blocks are rejected. Forces activation if the BIP is deployed.

### Speedy Trial (Taproot)

Used for BIP341/342 deployment in 2021:

- `startheight = 681,408`, `timeout = 709,632`.
- 90% threshold (1815/2016).
- If LOCKED_IN by Aug 11 2021, ACTIVE at block 709,632 (mid-November).
- Otherwise: FAILED, no LOT escalation. Goal: short window, low contention.

Result: locked in at retarget boundary 687,456 with bit 2 signaling; activated at 709,632.

### BIP-110 (2026) - a modified BIP9 that failed

BIP-110 "Reduced Data Temporary Softfork" is the most recent mainnet deployment attempt, and the useful failure case. Parameters, from BIP-110 v1.0.1:

```
name                  = reduced_data
bit                   = 4
starttime             = 1764547200     (~1 December 2025)
timeout               = NO_TIMEOUT
min_activation_height = 0
max_activation_height = 965,664        (~1 September 2026)
active_duration       = 52,416 blocks  (~1 year, then EXPIRED)
threshold             = 1109/2016      (55%)
```

Five deviations from BIP9:

- Threshold 55% instead of 95%, justified in the BIP by the deployment being temporary.
- No time-based timeout. A BIP8-style height-based `max_activation_height` replaces it, so **FAILED is unreachable** - the BIP says so explicitly.
- A BIP8-like mandatory-signaling window over blocks 961,632-963,647, the retarget period immediately before forced lock-in at 963,648. Inside that window an enforcing node rejects any block that does not set bit 4.
- A new terminal `EXPIRED` state entered at `activation_height + active_duration`, after which the rules stop being enforced.
- A redefined state-transition progression: DEFINED -> STARTED -> LOCKED_IN -> ACTIVE -> EXPIRED, with LOCKED_IN forced no later than 963,648 by the mandatory-signaling window and `FAILED` dropped entirely.

UTXO grandfathering - inputs spending UTXOs created before the activation height are exempt from all of the new rules - is *not* one of those five. It is a consensus rule in the BIP's Specification section, added in draft 0.0.2 (2025-11-05) so that no pre-activation coin could be frozen; it says nothing about the activation mechanism.

What actually happened: mainnet block 961,632 is timestamped 2026-08-08 19:35 UTC and carries `nVersion = 0x20006000` - versionbits flag set, bit 4 clear - so nodes enforcing BIP-110 rejected it and continued on their own branch. About 2.53% of blocks had signaled over the preceding two weeks, against the 55% needed. The minority branch inherited the full network difficulty with almost none of the hashrate and produced two blocks in roughly eight hours, by which time the majority chain had reached 961,681 (CoinDesk, 2026-08-09). The BIPs repo moved BIP-110 to Status **Closed** on 2026-08-10 (bitcoin/bips #2245); the BIP's own changelog records v1.0.1, 2026-08-09, "Mark as Closed, following a chain split with stalled mining".

The state-machine lesson: removing FAILED does not remove failure. With `timeout = NO_TIMEOUT` the deployment had no way to record a clean loss, so the only available expression of insufficient hashpower was a chain split - and because the split happened on a retarget boundary, the minority branch needed a full 2,016 blocks at mainnet difficulty before it could adjust. There is no BIP-110 rule in force on mainnet as of September 2026.

## Worked example

Pseudocode for evaluating BIP9 state at a given block (`prev`):

```python
def get_state(prev, deployment):
    if prev is None: return DEFINED
    height = prev.height + 1
    # state changes only on retarget boundary
    if height % 2016 != 0:
        return get_state(prev.parent_at_boundary, deployment)
    state = get_state(prev.previous_period, deployment)
    mtp = median_time_past(prev)
    if state == DEFINED:
        if mtp >= deployment.starttime: return STARTED
    if state == STARTED:
        count = sum(1 for b in last_2016(prev)
                    if (b.nVersion & 0xE0000000) == 0x20000000
                    and (b.nVersion >> deployment.bit) & 1)
        if count >= deployment.threshold: return LOCKED_IN
        if mtp >= deployment.timeout:    return FAILED
    if state == LOCKED_IN:
        return ACTIVE
    return state  # ACTIVE / FAILED terminal
```

For BIP148 UASF (SegWit), users (not miners) ran patched nodes that rejected blocks not signaling bit 1 between Aug 1 and Nov 15, 2017. Miners capitulated and SegWit locked in via BIP9.

## Common bugs / anti-patterns

- Counting signaling bits within the **current** window rather than the previous one - state advances at the boundary.
- Forgetting to mask `nVersion`: a block with nVersion=4 has no versionbits flag and does **not** count as signaling, even if bit 2 is set.
- Allowing `nVersion` bits other than the deployed bit to determine activation; only the deployment's specific bit matters.
- Confusing height-based (BIP8) with time-based (BIP9) thresholds when porting code.
- Treating LOCKED_IN as ACTIVE too early - new rules apply only from the retarget boundary after lock-in.
- Reading a non-standard threshold off BIP9 itself: each deployment carries its own (`threshold` in `chainparams.cpp`), and BIP-110 set 1109/2016.
- Using MTP at the tip rather than at `prev` for state evaluation, off-by-one resulting in wrong activation height.

## References

- BIP9: https://github.com/bitcoin/bips/blob/master/bip-0009.mediawiki
- BIP8: https://github.com/bitcoin/bips/blob/master/bip-0008.mediawiki
- BIP341 activation params (Speedy Trial): https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki
- BIP148 (UASF): https://github.com/bitcoin/bips/blob/master/bip-0148.mediawiki
- BIP110 (Reduced Data Temporary Softfork - Status: Closed since 2026-08-10): https://github.com/bitcoin/bips/blob/master/bip-0110.mediawiki
- bitcoin/bips PR #2245 (bip110: update status to closed, merged 2026-08-10): https://github.com/bitcoin/bips/pull/2245
- Bitcoin Core `src/versionbits.cpp`, `src/kernel/chainparams.cpp` (per-deployment `threshold`)
