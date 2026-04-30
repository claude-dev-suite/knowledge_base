# Mempool-Based Fee Estimation Strategies - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/wallets/fee-estimation`.
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/wallets/fee-estimation/SKILL.md

## Concept

Where `estimatesmartfee` looks **backward** at confirmed history,
mempool-based estimation looks **forward** at the queue of unconfirmed
transactions and the next miner's likely block content. The basic insight
is the **block template**: at any moment, the next miner builds a block by
sorting mempool by package fee rate (descending), then including until they
hit the 1 MWU weight limit. The fee rate of the *last* tx in that template
is the marginal rate to enter the next block.

This produces a far more responsive estimate during fee spikes — the
mempool reveals demand the instant transactions arrive — at the cost of
volatility in quiet periods.

## Walkthrough / mechanics

The pipeline:

1. Pull `getrawmempool true` (verbose mode includes `vsize`, `fees.modified`,
   ancestor data, descendant data).
2. Sort entries by `fees.modified / vsize` descending. This is *not*
   the actual mining order because miners use ancestor-aware sorting; for a
   first-pass approximation it's usable.
3. Cumulative sum of `vsize` until you reach 1 000 000 (next block) or
   2 000 000 (block after). Record the boundary fee rate.

Higher-fidelity implementations include ancestor scoring:

```
score(tx) = max(
    feerate(tx),
    feerate({tx} + ancestors),
    package_feerate of any package containing tx
)
```

Bitcoin Core itself produces a roughly correct view via `getblocktemplate`
which exposes the actual chosen txs in the next miner's block. It does
not, however, give you a smooth quantile function.

## Worked example

Manual Python prototype:

```python
import json, subprocess

raw = subprocess.check_output(["bitcoin-cli", "getrawmempool", "true"])
mp = json.loads(raw)

entries = sorted(
    ((info["fees"]["modified"] * 1e8 / info["vsize"], info["vsize"])
     for info in mp.values()),
    reverse=True
)

cum = 0
boundaries = {1_000_000: None, 2_000_000: None, 3_000_000: None}
for rate, vsz in entries:
    cum += vsz
    for limit in list(boundaries):
        if boundaries[limit] is None and cum >= limit:
            boundaries[limit] = rate

print("Next block:", boundaries[1_000_000], "sat/vB")
print("2nd block: ", boundaries[2_000_000], "sat/vB")
print("3rd block: ", boundaries[3_000_000], "sat/vB")
```

Sample output during a quiet period:

```
Next block: 2.05 sat/vB
2nd block:  1.34 sat/vB
3rd block:  1.0 sat/vB
```

During an inscriptions wave:

```
Next block: 184.6 sat/vB
2nd block:  127.4 sat/vB
3rd block:  98.2 sat/vB
```

`estimatesmartfee` would still be reading 30 sat/vB from history and lag
several blocks behind reality.

## Mempool projection APIs

External services consume the same data and offer the same projection,
which you can use as a cross-check or fallback:

```bash
# mempool.space - per-block projection
curl -s https://mempool.space/api/v1/fees/mempool-blocks | jq '.[0:3]'
```

```json
[
  {"blockSize": 999984,
   "blockVSize": 999998,
   "nTx": 2812,
   "totalFees": 7894400,
   "medianFee": 7.89,
   "feeRange": [7.0, 8.1, 9.5, ..., 184.6]},
  ...
]
```

`feeRange` is a 10-quantile array per block. Use the 90th-percentile entry
to enter the block confidently; the median will get you in most of the
time.

## Hybrid strategy

A robust wallet combines multiple sources:

```python
def fee_for_target(target_blocks):
    candidates = []

    # Source 1: estimatesmartfee
    try:
        r = rpc.estimatesmartfee(target_blocks, "ECONOMICAL")
        if "feerate" in r:
            candidates.append(r["feerate"] * 1e5)
    except Exception:
        pass

    # Source 2: local mempool projection
    try:
        candidates.append(local_mempool_projection(target_blocks))
    except Exception:
        pass

    # Source 3: external API
    try:
        candidates.append(mempool_space_recommended(target_blocks))
    except Exception:
        pass

    # mempoolminfee as absolute floor
    floor = rpc.getmempoolinfo()["mempoolminfee"] * 1e5
    return max(floor + 0.1, max(candidates))
```

The `+ 0.1` headroom is for rounding error and the incremental relay fee.

## Common pitfalls

- **Feature: `mempoolminfee`** — when mempool is full, this rises above
  1 sat/vB. Any tx below the floor is silently rejected at admission.
  Always cross-reference.
- **Eviction-aware**: an estimate of 4 sat/vB to "enter the next block" is
  meaningless if mempool is at 5 sat/vB minfee. Your tx never even reaches
  the queue.
- **Ancestor packages skew naive sorting**: a tx with low feerate but a
  high-feerate child (CPFP) will be mined together. Ranking only by raw
  feerate misses this.
- **L2 spikes**: ordinals/runes events dump tens of thousands of low-priority
  txs in seconds. Median fee in the mempool snapshot can spike before
  `estimatesmartfee` reacts; relying solely on estimatesmartfee leaves you
  stuck for hours.
- **Double-counting**: subscribing to ZMQ `rawtx` and also calling
  `getrawmempool` produces duplicates. Use one or the other.
- **Time-zone bias**: many strategies sample once per hour. Fee dynamics
  are strongly correlated with UTC business hours; an hourly snapshot
  averaged across days hides fast intra-day movement.

## References

- mempool.space API docs: https://mempool.space/docs/api/rest
- Core `getblocktemplate`: https://developer.bitcoin.org/reference/rpc/getblocktemplate.html
- See also: [estimatesmartfee-deep.md](estimatesmartfee-deep.md), [../rbf-cpfp/bip125-rules-walkthrough.md](../rbf-cpfp/bip125-rules-walkthrough.md)
