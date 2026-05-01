# Time-Warp Protection - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/mining/difficulty`.
> Canonical source: Bitcoin Core `src/validation.cpp::ContextualCheckBlockHeader`, BIP 113
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/mining/difficulty/SKILL.md

## Concept

A "time-warp" attack manipulates block timestamps to artificially extend or
shorten the apparent epoch duration, causing the difficulty algorithm to
under- or over-correct. The attacker's goal: keep difficulty low while real
hashrate is high (or vice versa) to mine more blocks per unit work.

Bitcoin defends against this with two rules on every block header:

1. `block.timestamp > MTP(last 11 blocks)` - the median-time-past rule.
2. `block.timestamp <= adjusted_network_time + 2 * 60 * 60` - the future
   block rule (2 hour cap, where adjusted_network_time is the median
   time across connected peers).

These bound how far a single miner can drag a timestamp in either direction
without nodes rejecting the block.

## Walkthrough / mechanics

### Why a time warp matters

The retarget formula uses the timestamp of the *first* and *last* block of
the previous 2016-block window. If a colluding cartel could systematically
back-date the *last* block of each window, they would inflate
`nActualTimespan`, making the network think blocks are slow, and the
algorithm would lower difficulty. Since each retarget is independent, the
effect compounds across epochs.

This was the basis of the well-known attack on Bitcoin Cash (which weakened
its time anchor) and is why Bitcoin's two rules are conservative.

### Median Time Past (MTP)

```cpp
int64_t MedianTimePast(const CBlockIndex* pindexPrev, int n = 11) {
    std::vector<int64_t> times;
    const CBlockIndex* p = pindexPrev;
    for (int i = 0; i < n && p; ++i, p = p->pprev)
        times.push_back(p->GetBlockTime());
    std::sort(times.begin(), times.end());
    return times[times.size() / 2];
}
```

A new block's timestamp must be strictly greater than this median. Because
it's a median over 11, an attacker controlling fewer than 6 of the last 11
blocks cannot set the MTP arbitrarily.

BIP 113 also uses MTP for `nLockTime` and CSV time-locks - this guarantees
that a block's "claimed time" is bounded by the chain's history.

### Future block rule

Each node maintains a "network adjusted time" - the median offset between
the node's clock and its peers' reported clocks (capped to +-70 minutes).
A block whose timestamp exceeds `adjusted_time + 7200` seconds is rejected
as too far in the future.

This is a soft barrier: nodes accept it later once their clock catches up
or the block falls within 2h of the new now. The block isn't permanently
banned, just queued.

### Combined effect on retargets

For an attacker to lower difficulty, they would need to make the *last*
timestamp of an epoch as late as possible (within +2h of network time) and
the *first* timestamp of the same epoch as early as possible (subject to
MTP from the previous epoch's last 11 blocks). The maximum manipulable
range per retarget is bounded:

```
max_late  = +2h  (future cap)
max_early = MTP gives ~ -10 to -20 min depending on rate
```

Across the 2016 blocks, the realistic skew is bounded to a few minutes -
not enough for a meaningful difficulty drop. With > 50% hashrate, the
attacker has bigger problems to exploit anyway (51% reorgs).

## Worked example

Suppose the last 11 block timestamps (oldest to newest) are:

```
1_713_900_000
1_713_900_540
1_713_901_080
1_713_901_620
1_713_902_160
1_713_902_700  <- MEDIAN (index 5)
1_713_903_240
1_713_903_780
1_713_904_320
1_713_904_860
1_713_905_400
```

`MTP = 1_713_902_700`. A new block's timestamp must be strictly greater
than this. If the local node thinks it is `now = 1_713_905_500`, the block
timestamp must satisfy:

```
1_713_902_700 < block.timestamp <= 1_713_905_500 + 7200
                                  = 1_713_912_700
```

So the miner has a ~166-minute window. Rolling `ntime` within this range
is exactly what `mining.notify` and Stratum V2 allow miners to do without
re-issuing work.

A miner attempting to extend the apparent epoch by setting
`block.timestamp = 1_713_912_700` (max future) gives the next epoch's
first-block timestamp 2 hours of slack. But the previous epoch's final
timestamp can only be ~10 minutes off true median, so the manipulation
is one-shot per epoch and quickly damps out.

## Common pitfalls

- **Using local wall-clock time** - validators must use the network
  adjusted time, not the system clock. A wrong-clock node will reject
  valid blocks or accept invalid ones.
- **Forgetting strict inequality** - the rule is `timestamp > MTP`, not
  `>=`. Equal-to-median is rejected.
- **Mixing `nTime` rolling with retargets** - miners may roll `ntime`
  within the [MTP+1, now+2h] band per the Stratum spec. Pool software
  must filter rolled timestamps that exceed the band.
- **Off-by-one MTP window size** - some forks use 11, some use other
  values. Bitcoin mainnet is hardcoded 11.
- **Underestimating the BCH attack class** - if you implement a fork or
  altchain, copying Bitcoin's retarget without MTP/future-cap leaves
  you vulnerable to a deep epoch-skew time-warp.

## References

- BIP 113: Median time-past as endpoint for lock-time calculations
- Bitcoin Core `src/validation.cpp::ContextualCheckBlockHeader`
- "Time Warp Attack" analysis on Bitcoin Cash (2018-2020 research)
- `src/timedata.cpp` - network adjusted time implementation
