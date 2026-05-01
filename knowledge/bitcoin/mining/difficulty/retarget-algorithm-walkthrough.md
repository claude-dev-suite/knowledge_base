# Difficulty Retarget Algorithm Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/mining/difficulty`.
> Canonical source: Bitcoin Core `src/pow.cpp::CalculateNextWorkRequired`
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/mining/difficulty/SKILL.md

## Concept

Bitcoin re-targets every 2016 blocks (`params.DifficultyAdjustmentInterval()`)
so the next epoch's expected block time approaches 600 seconds. The algorithm
is deterministic: every node computes the same new `nBits` independently, so
no consensus message is needed.

Three quantities matter:
- `nFirstBlockTime` - timestamp of the first block of the previous 2016
  window.
- `nLastBlockTime` - timestamp of the last block of the previous 2016
  window (block at height `H - 1`, where `H` is the height being mined).
- `pindexLast->nBits` - target of the previous block, encoded compact.

## Walkthrough / mechanics

### The compact `nBits` encoding

`nBits` packs a 256-bit target into 32 bits:

```
nBits = (exponent << 24) | mantissa
target = mantissa * 256^(exponent - 3)
```

Example: `nBits = 0x17034219` -> exponent `0x17 = 23`, mantissa
`0x034219 = 213_529`. Target = `213529 * 256^20` ~ `2.78e54`.

The mantissa's high bit must be zero (otherwise the value is treated as
negative); if a calculation produces a high bit, it must be shifted right
and the exponent incremented.

### The retarget formula

```cpp
int64_t nActualTimespan = pindexLast->GetBlockTime() - nFirstBlockTime;
if (nActualTimespan < params.nPowTargetTimespan / 4)
    nActualTimespan = params.nPowTargetTimespan / 4;
if (nActualTimespan > params.nPowTargetTimespan * 4)
    nActualTimespan = params.nPowTargetTimespan * 4;

arith_uint256 bnNew;
bnNew.SetCompact(pindexLast->nBits);
bnNew *= nActualTimespan;
bnNew /= params.nPowTargetTimespan;

if (bnNew > params.powLimit)
    bnNew = params.powLimit;

return bnNew.GetCompact();
```

`params.nPowTargetTimespan = 2016 * 600 = 1_209_600` seconds.

The clamp `[1/4, 4]` is done on the timespan, not on the resulting
difficulty, but it has the same effect: difficulty can change by at most
4x up or 4x down per epoch.

### The famous off-by-one

Bitcoin uses 2015 intervals between 2016 timestamps. The reference code
takes `nActualTimespan = block[2015].time - block[0].time`. This is
`2015` block intervals, not `2016`. Satoshi's original code had this and
it was never fixed for backward compatibility - changing it would be a
hard fork. The result: average block time converges to `600 * (2015/2016)`
~ 599.7 seconds. Negligible in practice but well-documented.

### Where the next-target check lives

Every node, on receiving a block at height that is a multiple of 2016,
runs the formula and verifies `block.nBits == computed_new_bits`. A
mismatch is a hard rule violation; the block is rejected outright.

For non-retarget heights, `block.nBits` must equal `parent.nBits`.

## Worked example

Suppose epoch ending at height 893519:

- `pindexLast = block_893519`, `nBits = 0x17034219`
  -> target = `mantissa * 256^20 = 213529 * 256^20`.
- `nFirstBlockTime = block_891504.time = 1_712_790_000`.
- `nLastBlockTime  = block_893519.time = 1_713_998_400`.
- `nActualTimespan = 1_713_998_400 - 1_712_790_000 = 1_208_400 sec`.
- Target timespan = 1_209_600.
- Clamp: 1_208_400 is within [302_400, 4_838_400], no clamp.

New target:

```
new_target = old_target * 1_208_400 / 1_209_600
           = old_target * 0.99901
```

So difficulty rises by `1 / 0.99901 - 1 = 0.099%`. A tiny adjustment
indicating the network was running almost exactly on schedule.

If instead `nActualTimespan = 1_000_000` (epoch was fast, hashrate up):

```
new_target = old_target * 1_000_000 / 1_209_600 = old_target * 0.8267
difficulty_change = 1 / 0.8267 - 1 = +20.9%
```

Difficulty jumps 20.9%. The 4x clamp would only kick in if the actual
timespan dropped below 302_400 sec or rose above 4_838_400 sec.

## Common pitfalls

- **Clamping after multiplication** - if you compute `new_target =
  old_target * actual / expected` first, then clamp the difficulty,
  you can hit overflow or get a different result than the spec.
- **Wrong interval count** - using 2016 intervals instead of 2015 in
  re-implementations causes slow drift from network consensus.
- **Compact encoding sign bit** - if `bnNew * nActualTimespan` produces
  a mantissa with the high bit set, you must normalise (shift right,
  increment exponent) before encoding to compact.
- **Using wall-clock time** - the algorithm uses block-header timestamps
  only. Wall-clock skew on the validating node is irrelevant.
- **Missing genesis-timestamp anchor** - `nFirstBlockTime` is the
  timestamp of the *first* block of the previous window, height
  `(H - 1) - 2015 = H - 2016`. Off-by-one here breaks consensus.

## References

- Bitcoin Core `src/pow.cpp::CalculateNextWorkRequired`
- Bitcoin Core `src/arith_uint256.cpp::SetCompact`
- Satoshi's original `bitcoin.cpp` (2009)
- BIP 9 (versionbits) - independent of retarget but worth reading for context
