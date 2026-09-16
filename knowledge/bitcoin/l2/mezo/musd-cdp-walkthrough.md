# Mezo MUSD CDP - Walkthrough

> Phase B article. Companion to dev-suite skill `bitcoin/l2/mezo`.
> Canonical source: https://github.com/mezo-org/musd/tree/main/docs
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/mezo/SKILL.md

## Concept

MUSD is a Bitcoin-collateralised CDP stablecoin on Mezo. Per its own
docs it is "based on Threshold USD, which is a fork of Liquity" — so
the trove model, the Stability Pool, the redemption-driven price
floor and the Default Pool redistribution fallback are Liquity v1's,
inherited wholesale.

What is *not* Liquity is where the interesting risk sits: MUSD adds
per-trove fixed simple interest, removes the Stability Pool
incentive token, seeds the Stability Pool with a protocol bootstrap
loan, and ships upgradable contracts.

## Walkthrough / mechanics

### Opening a trove

```
BorrowerOperations.openTrove   -> BTC in, MUSD out, BTC parked in ActivePool
BorrowerOperations.withdrawColl-> partial collateral withdrawal
BorrowerOperations.closeTrove  -> repay debt, reclaim BTC
```

Collateral stays in `ActivePool` until the borrower withdraws, closes,
is redeemed against (`TroveManager.redeemCollateral`) or is liquidated
(`TroveManager.liquidate`).

### The peg band

| Bound | Value | Enforced by |
|---|---|---|
| Floor | $1.00 | redeem MUSD for $1 of BTC at any time |
| Ceiling | ~$1.10 | the 110% minimum collateral ratio |

Floor arbitrage, using the docs' own worked example at BTC = $100k:
buy 1000 MUSD for $800 at a depegged $0.80, redeem for 0.01 BTC
($1000), sell. That buys and burns MUSD, pushing the price up, and
stays profitable until $1.

Ceiling arbitrage: at MUSD = $1.20, buy 1 BTC, open a trove against
it, mint the maximum 90,909 MUSD (110% MCR), sell for $109,091. That
mints and sells MUSD, pushing the price down, and stays profitable
until $1.10.

### Fees

| Fee | Rate | Notes |
|---|---|---|
| Borrowing | 0.1% | governable; added as trove debt, minted to governance |
| Redemption | 0.75% | governable; taken in BTC from the redeemer's payout |
| Refinancing | — | behaves like the borrowing rate |
| Interest | global governable rate | **simple, not compounding** |

The interest design is the sharp edge. A global governable rate
applies to troves *at open*; later changes to that global rate do not
touch existing troves. A borrower may refinance to the current global
rate at any time, paying the refinancing fee. So the protocol's
average interest income is a function of when its troves were opened,
and borrowers hold a free option to refinance downward.

Simple interest means $10,000 at 3% owes $300 after a year, $600
after two, and so on — no compounding.

### Liquidation

Any trove below **110%** is liquidatable by anyone, with no MUSD
balance required; the liquidator only pays gas. The liquidator
receives **$200 MUSD gas compensation plus 0.5% of the trove's
collateral**. Single troves go through `TroveManager.liquidate`,
batches through `TroveManager.batchLiquidateTroves`.

Three cases, in priority order:

1. **Stability Pool covers it.** The pool burns MUSD equal to the
   debt and takes the remaining 99.5% of collateral. Burn and
   collateral claim are proportional across all pool deposits.
2. **Stability Pool partially covers it.** The pool absorbs what it
   can; the remaining debt *and* collateral are redistributed
   proportionally across surviving troves via `DefaultPool`.
3. **Stability Pool empty.** All debt and collateral redistribute
   through `DefaultPool`.

Redistribution is not immediately harmful — because liquidations
happen at 110%, a borrower receives $1.10 of BTC for every $1 of debt
added — but it lowers their collateral ratio, raises their debt, and
forces them to source more MUSD to close. Pending redistributed debt
does **not** accrue interest until applied, and application happens
automatically on the borrower's next trove interaction.

### Stability Pool economics

Liquity v1 paid LQTY to Stability Pool depositors, which is why its
pool was deep. MUSD pays nothing: "there are no direct incentives for
depositing into the Stability Pool", and the docs state outright that
depositors are not expected. Depositor yield is purely the discount
captured on liquidations.

Instead the pool is seeded with a **protocol bootstrap loan**: MUSD
is minted to the PCV (Protocol Controlled Value) contract at
deployment and deposited into the Stability Pool. MUSD withdrawn from
the pool back to PCV is first applied to repaying that loan, so the
bootstrap loan cannot be extracted from the protocol.

Fee routing is governed by `feeSplitPercentage` on the PCV contract:

- `100` — all fees to the MUSD Savings Rate vault via
  `receiveProtocolYield`, none to loan repayment.
- `< 100` — the remainder burns down the bootstrap loan while it is
  outstanding, and is deposited into the Stability Pool once repaid.

BTC handling in PCV is deliberately split. Redemption-fee BTC is
distributed via `distributeBTC()` to `btcRecipient`. Stability Pool
liquidation-gain BTC (`stabilityPoolBTC`) is **not** touched by
`distributeBTC()` and must be pulled explicitly with
`withdrawStabilityBTC()` or `withdrawFromStabilityPool()`. A
`distributor` role can be granted to a bot to call `distributeBTC()`
alongside owner, council and treasury.

### Upgradability

Unlike Liquity v1, MUSD contracts are **upgradable**. The docs frame
this as a temporary measure — substantial changes are meant to ship
as a new contract set added to the MUSD token's mintlist and
burnlist, so users opt in by migrating, and upgradability is to be
removed "when the protocol has been battle tested in production".
Until that happens, MUSD is a governance-mutable stablecoin.

## Pitfalls / gotchas

- **110% MCR is thin.** There is very little buffer between solvent
  and liquidatable in a fast BTC drawdown, and the Stability Pool has
  no incentive attracting third-party depositors to absorb it.
- **Redistribution is the realistic path**, not pool absorption, if
  the bootstrap loan is the pool's main liquidity. Model your trove
  assuming you may inherit someone else's debt.
- **Interest rate is sticky at open.** Do not quote "the MUSD rate"
  as a single number; every trove carries the rate from its own open,
  and only refinancing moves it.
- **Upgradable contracts** — the peg, the fees and the liquidation
  rules are all governance-reachable today.
- **Underlying chain is Proof-of-Authority.** Mezo's `x/poa` module
  gates validator entry on module-owner approval and carries an
  Emergency Team role with chain lockdown levels. The stablecoin's
  liveness assumptions are the chain's.
- **Collateral is bridged BTC**, arriving over tBTC, so tBTC's
  threshold-signer assumptions sit under everything above.

## References

- `mezo-org/musd` — `docs/README.md` (fetched 15 September 2026).
- `mezo-org/mezod` — README, `x/poa/spec/README.md`,
  `docs/bridge-observability.md`.
- Liquity v1: https://github.com/liquity/dev
- Threshold USD: https://github.com/Threshold-USD/dev
- Mezo H1 2026 review: https://mezo.org/blog/mezo-h1-2026
