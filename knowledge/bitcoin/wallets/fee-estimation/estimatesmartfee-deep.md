# estimatesmartfee Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/wallets/fee-estimation`.
> Canonical source: Bitcoin Core source `src/policy/fees.cpp`
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/wallets/fee-estimation/SKILL.md

## Concept

`estimatesmartfee` is Bitcoin Core's built-in fee-rate oracle. It does not
look at the current mempool. Instead, it consults a rolling history of
which fee rates have *successfully* gotten transactions confirmed within
N blocks. This makes it a **lagging indicator** — robust against transient
spikes but slow to react to genuine demand shifts.

## Walkthrough / mechanics

Internally, `policy/fees.cpp` maintains three estimators with three different
decay constants:

| Estimator | Half-life | Used for |
|-----------|-----------|----------|
| Short    | 12 blocks  | targets <= 6 |
| Medium   | 24 blocks  | targets 7-24 |
| Long     | 48 blocks  | targets > 24 |

Each estimator buckets observed transactions by fee-rate (logarithmic
buckets, ~150 of them between 1 sat/vB and ~10 000 sat/vB) and records two
counters per bucket:

- `txCtAvg[bucket]` — exponentially weighted count of txs entering the
  mempool at this rate.
- `confAvg[depth][bucket]` — for each block-depth `1..maxDepth`, count of
  txs at this rate that confirmed within `depth` blocks.

A bucket is "successful at depth N" if confirmation rate >= success
threshold (95% for `CONSERVATIVE`, 60% for `ECONOMICAL`).

When you call `estimatesmartfee N`, Core walks bucket boundaries from
high-fee to low-fee and returns the lowest fee rate whose success rate at
depth `N` clears the threshold.

## Modes

| Mode | Threshold | Behaviour |
|------|-----------|-----------|
| `CONSERVATIVE` | 95% success | Picks the most aggressive fee. Higher costs but rarely stuck. |
| `ECONOMICAL`   | 60% success | Cheaper, occasionally late. Default since v0.15. |

`CONSERVATIVE` also takes the **maximum** of multiple horizons (short,
medium, long) — it returns the worst case. `ECONOMICAL` uses only the
shortest horizon that confirms within `N`.

## Worked example

```bash
$ bitcoin-cli estimatesmartfee 1 ECONOMICAL
{
  "feerate": 0.00006524,
  "blocks": 2
}
```

Two notable bits:
1. `feerate` is in **BTC/kvB**. Multiply by 100 000 to get sat/vB
   (here 6.524 sat/vB).
2. `blocks: 2` is the depth at which Core actually achieved the
   threshold. You asked for 1 block; Core couldn't make a confident
   estimate at depth 1 (insufficient buckets clear) and downgraded.

Inspect the raw data:

```bash
bitcoin-cli estimaterawfee 6 0.85
```

Returns the per-horizon details:

```json
{
  "short": {
    "startrange": 8.00,
    "endrange": 8.42,
    "withintarget": 12,
    "totalconfirmed": 58,
    "inmempool": 0,
    "leftmempool": 1
  },
  ...
}
```

`startrange`/`endrange` are the bucket boundaries; `withintarget` is the
count of txs in this bucket that confirmed within the target depth;
`totalconfirmed` is how many ever confirmed. The success ratio
`withintarget / totalconfirmed` must clear your supplied threshold (`0.85`
in the call) for this bucket to be eligible.

## Cold-start behaviour

A freshly synced node has empty estimator state. Three things happen:

1. For ~3 hours after start, calls return `{"feerate": -1, "errors":
   ["Insufficient data or no feerate found"]}`.
2. Core does not load history from disk by default; restart resets data.
   The `estimatefee.dat` file (bundled with `bitcoind`) preserves recent
   data across restarts but only if the node was running long enough.
3. Bring-up tip: warm the node with `acceptnonstdtxn=0` and let it observe
   ~12 blocks before relying on estimates.

## Bumping a stuck transaction

Use `bumpfee` which internally calls the estimator:

```bash
bitcoin-cli bumpfee abcdef... '{"conf_target": 6, "estimate_mode": "ECONOMICAL"}'
```

Or supply a hardcoded rate:

```bash
bitcoin-cli bumpfee abcdef... '{"fee_rate": 25}'
```

Note the units: `fee_rate` here is **sat/vB**, not BTC/kvB. Core's units
are inconsistent across RPCs.

## Common pitfalls

- Calling on a fresh node and getting `-1` → bail-out logic should fall
  back to a sensible default (e.g., 1.5x current floor) or query an
  external oracle.
- Treating BTC/kvB as sat/vB → 100 000x error. Always normalise.
- Using `CONSERVATIVE` for a low-priority test transaction on a busy
  network → overpaying by 3-10x.
- Believing `blocks` in the response equals your requested target:
  Core returns the depth it could actually clear, not what you asked for.
- Polling once per minute and trusting the result for the next hour: the
  estimate updates lazily, so a sudden fee spike may not appear for several
  blocks. Combine with `getmempoolinfo.mempoolminfee` for a "right-now"
  floor check.
- Forgetting `mempoolminfee`: even if Core's history says 5 sat/vB is fine,
  the current mempool floor may be 8 sat/vB; broadcasting at 5 fails with
  `min relay fee not met`.

## References

- Bitcoin Core fees.cpp: https://github.com/bitcoin/bitcoin/blob/master/src/policy/fees.cpp
- Original PR adding smart-fee: https://github.com/bitcoin/bitcoin/pull/9387
- See also: [mempool-based-strategies.md](mempool-based-strategies.md), [../rbf-cpfp/bip125-rules-walkthrough.md](../rbf-cpfp/bip125-rules-walkthrough.md)
