# Waste Metric (Murch) - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/wallets/coin-selection`.
> Canonical source: Murch's coin-selection PhD work and Bitcoin Core PR 17331
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/wallets/coin-selection/SKILL.md

## Concept

The "waste metric" formalises the cost of a coin-selection candidate in a way
that captures both immediate fees and **future** costs. A candidate selection
producing a change output incurs a current fee for that output and a future
fee to spend it. A candidate without a change output forfeits any value above
the target as fee.

Bitcoin Core uses this number to decide between two acceptable candidates
when multiple succeed (e.g., BnB returned and SRD returned).

## Formula

```
waste = sum_of_input_fees * (current_feerate - long_term_feerate)
      + cost_of_change                    if change_output_present
      + excess                            if change_output_absent
```

Where:
- `sum_of_input_fees = sum_i(input_vsize_i * current_feerate)`
- `cost_of_change = output_vsize * current_feerate
                  + future_input_vsize * long_term_feerate`
- `excess = sum(values) - target - fee` (paid to miner as fee surplus)

The first term captures **input timing**: spending UTXOs at a high feerate
costs more than spending them later at a low feerate. If `current > long_term`,
selecting more inputs makes the timing worse (positive waste). If
`current < long_term`, more inputs is *good* — you're prepaying the future
spend at a discount.

The change/excess term captures the **change vs. drop-to-fee** tradeoff.
- Change output: pay output cost now + spend cost later. Total ≈ 30 + 68 vB.
- No change: forfeit the excess into fee right now.

If `excess < cost_of_change`, dropping to fee is cheaper. BnB exploits this
to skip change creation when the math works out.

## Walkthrough / mechanics

Consider three candidates for the same target=6 000, fee=10 sat/vB,
long_term=5 sat/vB:

| Candidate | Inputs | Sum   | Excess | Change? | Waste calc | Waste |
|-----------|--------|-------|--------|---------|------------|-------|
| C1        | A      | 10000 | 4000   | yes     | 1*68*(10-5) + (31*10 + 68*5) = 340 + 650 | 990 |
| C2        | B+C    | 7500  | 1500   | yes     | 2*68*(10-5) + 650 = 680 + 650 | 1330 |
| C3        | B+C    | 7500  | 1500   | NO (drop excess) | 2*68*(10-5) + 1500 = 680 + 1500 | 2180 |
| C4        | B+C+D  | 8700  | 200    | NO      | 3*68*(10-5) + 200 = 1020 + 200 | 1220 |

The wallet would prefer **C1 (waste=990)** even though it has only one input.
Why? Because long_term < current means each extra input is expensive at the
moment, so consolidating into one big input (and producing a 4 000-sat change
output to recover later cheaply) is best. Note also that C4 is preferable
to C2 (1220 < 1330) because C4 drops excess and saves the future change
spend.

If we flip the rates (current=5, long_term=10), preferences invert:

```
C1 waste = 1*68*(5-10) + (31*5 + 68*10) = -340 + 835 = 495
C4 waste = 3*68*(5-10) + 200 = -1020 + 200 = -820   <-- best!
```

Now consolidating *more* inputs is rewarded — long-term fees are higher, so
spending UTXOs while it's cheap is wise. C4 has negative waste, meaning the
wallet is effectively saving money relative to the no-fee baseline.

## Worked example

Bitcoin Core lets you tune long-term fee assumption via the wallet's
`fallbackfee`/`m_long_term_feerate`. To see the effective decision, build
the same tx twice with different rates:

```bash
# Hot fees right now
bitcoin-cli -rpcwallet=hot estimatesmartfee 1
# {"feerate": 0.00010000}     -> 10 sat/vB

bitcoin-cli -rpcwallet=hot walletcreatefundedpsbt \
  '[]' '[{"bc1q...":0.0001}]' 0 \
  '{"feeRate":0.0001,"changePosition":1,"includeWatching":false}'
# Wallet selected inputs based on internal waste calc.

# Inspect tx
bitcoin-cli decodepsbt "$psbt" | jq '.tx.vin[].txid, .tx.vout[].value'
```

For BDK applications you can plug in a custom selection strategy and watch
the trace:

```rust
let selection = LargestFirstCoinSelection::default();
let psbt = wallet.build_tx()
    .add_recipient(addr, 6_000)
    .fee_rate(FeeRate::from_sat_per_vb(10.0))
    .coin_selection(selection)
    .finish()?;
```

BDK exposes the candidate set; you can implement a "min-waste" wrapper that
runs multiple strategies and picks the one with lowest waste.

## Common pitfalls

- Treating waste as an absolute cost. It is comparative: a negative waste
  doesn't mean "free", it means "better than the reference (one P2WPKH input
  + one P2WPKH output)" given the rate gap.
- Forgetting that `cost_of_change` includes a *future* input cost. Using
  only the change output's vsize understates the total ~by 50%.
- Hardcoding `long_term_feerate = current_feerate`. This collapses the
  metric — every solution has waste = `change_cost` or `excess`, removing
  any input-count signal.
- BnB requires `cost_of_change` for its search window; using waste's
  `cost_of_change` directly couples the two correctly.
- The waste metric does **not** model privacy. Two solutions can have the
  same waste while one leaks the user's wallet structure. Privacy-aware
  wallets layer SRD on top of waste comparisons.

## References

- Bitcoin Core PR 17331 (introduces the metric): https://github.com/bitcoin/bitcoin/pull/17331
- Murch's coin-selection paper: https://murch.one/wp-content/uploads/2016/11/erhardt2016coinselection.pdf
- See also: [bnb-walkthrough.md](bnb-walkthrough.md), [srd-walkthrough.md](srd-walkthrough.md), [../fee-estimation/estimatesmartfee-deep.md](../fee-estimation/estimatesmartfee-deep.md)
