# LND Close Confirmation and Reorg Safety - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/lnd`.
> Canonical source: https://github.com/lightningnetwork/lnd/blob/master/lnwallet/confscale.go
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/lnd/SKILL.md

## Concept

Until `v0.21.0-beta` (2026-06-05), LND treated a channel close as final
the moment the funding-output spend was detected on chain - effectively
one confirmation, and in the coop-close path a literal hardcoded `1`.
Forgetting a channel means discarding its revocation secrets, so after
a one-block reorg the counterparty could publish *any* revoked
commitment for that channel and LND would not publish a penalty
transaction, because it no longer believed the channel existed. Up to
the full channel balance was at risk.

BOLT 5 has always recommended waiting for confirmations before
discarding channel state. `v0.21.0-beta` implements that as
**capacity-scaled confirmation requirements**: the larger the channel,
the deeper the close has to bury before LND forgets it. The trade is
explicit - a small channel closes fast, a wumbo channel closes safely.

Bastien Teinturier posted the responsible disclosure to Delving
Bitcoin on 2026-08-13, stating that the bug "was fixed version 20.0
(shipped in February 2026)"; Bitcoin Optech summarised it in
Newsletter #419 (2026-08-21) as affecting "LND versions before 0.20.0,
which fixed it in February 2026", noting that to Teinturier's
knowledge nobody was affected. Both are wrong about the version:
`lnwallet/confscale.go` and `lnwallet/confscale_prod.go` do not exist
at the `v0.20.4-beta` tag (HTTP 404 on raw.githubusercontent.com as of
September 2026) and first appear at `v0.21.0-beta`. PR #10331, which
both sources name as the fix, is listed in
`docs/release-notes/release-notes-0.21.0.md` and in none of the
`release-notes-0.20.x.md` files. **v0.21.0-beta is the minimum safe
version.**

Optech's summary is inaccurate on a second point: it says the patch
makes a node wait "at least six" confirmations, following BOLT 5. The
code waits 3 to 6, scaled by capacity (below), so a small channel is
still forgotten at 3 - short of the floor Teinturier's own post argues
for ("should not let node operators use values lower than 6").

## Walkthrough / mechanics

### The scaling function

`lnwallet.ScaleNumConfs` is a linear interpolation over channel value,
with `maxChannelSize` fixed at `MaxBtcFundingAmount` = 16,777,215 sat
(0.16777215 BTC):

```go
const (
        minRequiredConfs = 1
        maxRequiredConfs = 6
        maxChannelSize   = 16777215
)

func ScaleNumConfs(chanAmt btcutil.Amount,
        pushAmt lnwire.MilliSatoshi) uint16 {

        if chanAmt > maxChannelSize {
                return maxRequiredConfs
        }
        stake := lnwire.NewMSatFromSatoshis(chanAmt) + pushAmt
        conf := uint64(maxRequiredConfs) * uint64(stake) /
                uint64(lnwire.NewMSatFromSatoshis(maxChannelSize))
        // clamped to [minRequiredConfs, maxRequiredConfs]
}
```

Two callers, two floors:

| Caller | Wrapper | Floor | Ceiling |
|--------|---------|-------|---------|
| Funding | `FundingConfsForAmounts(chanAmt, pushAmt)` | 1 | 6 |
| Close | `CloseConfsForCapacity(capacity)` | 3 | 6 |

`CloseConfsForCapacity` passes `pushAmt = 0` (a close has no push
amount) and then raises the result to a hard `minCloseConfs = 3`:

```go
func CloseConfsForCapacity(capacity btcutil.Amount) uint32 {
        scaledConfs := uint32(ScaleNumConfs(capacity, 0))
        const minCloseConfs = 3
        if scaledConfs < minCloseConfs {
                return minCloseConfs
        }
        return scaledConfs
}
```

Wumbo channels (`chanAmt > 16,777,215` sat) short-circuit to 6.

The function lives in `lnwallet/confscale_prod.go` behind
`//go:build !integration`. The `integration` build returns a flat `1`
so the itest suite stays fast - which means **an `integration`-tagged
binary has the pre-fix behaviour**. There is also a `chanCloseConfs`
override on the `chainWatcher` config, documented as dev/integration
builds only; it is not exposed as an `lnd.conf` option.

### Where it is enforced

- `contractcourt/chain_watcher.go`: `requiredConfsForSpend()` resolves
  the override, else `CloseConfsForCapacity(chanState.Capacity)`. This
  gates processing of the funding-output spend.
- `peer/brontide.go`: `WaitForChanToClose` takes a `numConfs` argument
  and passes it to `RegisterConfirmationsNtfn`. Before the fix this
  argument did not exist and the registration was hardcoded to 1.

### The event-ordering wrinkle

Making the close multi-confirmation broke a behaviour integrators
depended on. `SubscribeChannelEvents` stopped emitting `CLOSED_CHANNEL`
until full depth was reached, so UIs that keyed off it hung for up to
six blocks. PR #10794, also in `v0.21.0-beta`, restores the v0.20.1
timing: the chain watcher fires an early `CLOSED_CHANNEL` over the
channel notifier as soon as the coop-close spend lands, and the channel
arbitrator suppresses the duplicate that `MarkChannelClosed` would
otherwise emit at final depth.

So as of `v0.21.0-beta` there are two distinct moments:

1. Spend seen on chain -> early `CLOSED_CHANNEL` event. **Not final.**
2. Required depth reached -> `MarkChannelClosed`, state discarded.

`PendingChannels` gained two fields on `WaitingCloseChannel` (#10509)
so callers can tell them apart: `close_height` (height at which the
closing transaction first confirmed) and `blocks_til_close_confirmed`
(remaining confirmations).

## Worked example

Required close confirmations for representative capacities, from the
formula above:

| Capacity (sat) | `ScaleNumConfs` | Close confs |
|----------------|-----------------|-------------|
| 1,000,000 | 0 -> clamped to 1 | 3 |
| 5,000,000 | 1 | 3 |
| 8,388,607 | 2 | 3 |
| 11,184,810 | 4 | 4 |
| 13,981,013 | 5 | 5 |
| 16,777,215 | 6 | 6 |
| 50,000,000 (wumbo) | 6 (short-circuit) | 6 |

The 3-confirmation floor dominates everything up to ~0.1118 BTC, so in
practice most channels on the network close at exactly 3 confirmations
and only large ones scale past it.

An integrator polling a 1 M sat coop close:

```bash
lncli pendingchannels | jq '.waiting_close_channels[] |
  {chan: .channel.channel_point,
   close_height,
   blocks_til_close_confirmed}'
```

`close_height` populates at confirmation 1 (and the early
`CLOSED_CHANNEL` event fires there too), then
`blocks_til_close_confirmed` counts 2, 1, 0 over the next two blocks.
Only at 0 is the channel actually forgotten.

## Common pitfalls

- **Treating v0.20.x as patched.** The disclosure post says "v20.0" and
  Optech says "0.20.0". The code and the release notes say otherwise.
  Check for `lnwallet/confscale.go` in the tree you are running.
- **Treating the first confirmation as done.** The early
  `CLOSED_CHANNEL` event is deliberately re-added for UX; it is not a
  finality signal. Key settlement/accounting off
  `blocks_til_close_confirmed == 0`, not off the event.
- **Alerting on "stuck" closes.** A close sitting at 1 of 3 confs for
  20 minutes is normal on v0.21.0+. Monitoring tuned against v0.20
  will page for it.
- **Shipping an `integration`-tagged build.** `CloseConfsForCapacity`
  returns 1 under that build tag. Release binaries do not use it -
  `make release` is the safe path.
- **Assuming force closes are exempt.** The comment on
  `CloseConfsForCapacity` states it is used for both cooperative and
  force closes.
- **Missing the breaking change that rode along in #10331.**
  `MinCLTVDelta` rose from 18 to 24 in `v0.21.0-beta`, for a larger
  safety margin over `DefaultFinalCltvRejectDelta` (19 blocks).
  Invoices built with a custom `cltv_expiry_delta` of 18-23 are now
  rejected. The default of 80 is unaffected, and invoices created
  before the upgrade keep working.

## References

- lnd `lnwallet/confscale.go`, `confscale_prod.go`,
  `confscale_integration.go` at tag `v0.21.0-beta`.
- lnd `contractcourt/chain_watcher.go` (`requiredConfsForSpend`),
  `peer/brontide.go` (`WaitForChanToClose`).
- lnd `docs/release-notes/release-notes-0.21.0.md` (#10331, #10794,
  #10509).
- Delving Bitcoin, "Disclosure: LND doesn't wait for enough
  confirmations when closing channels" (t-bast, 2026-08-13).
- Bitcoin Optech Newsletter #419 (2026-08-21) - summary; inaccurate on
  the fixed version and the confirmation count.
- BOLT 5 - recommendations on waiting for confirmations before
  discarding channel state.
