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

Transitions are evaluated only at retarget boundaries (every 2016 blocks). The threshold for mainnet is 1815/2016 (~90% historically; SegWit/Taproot used 1815). Counting is over the `previous` retarget window.

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
- Using MTP at the tip rather than at `prev` for state evaluation, off-by-one resulting in wrong activation height.

## References

- BIP9: https://github.com/bitcoin/bips/blob/master/bip-0009.mediawiki
- BIP8: https://github.com/bitcoin/bips/blob/master/bip-0008.mediawiki
- BIP341 activation params (Speedy Trial): https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki
- BIP148 (UASF): https://github.com/bitcoin/bips/blob/master/bip-0148.mediawiki
- Bitcoin Core `src/versionbits.cpp`
