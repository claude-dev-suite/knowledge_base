# Channel-On-Pay Mechanics - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/lsp`.
> Canonical source: BLIP-51 + ACINQ splice-on-demand white-paper
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/lsp/SKILL.md

## Concept

Channel-on-pay generalizes JIT channels: instead of opening only when
inbound capacity is zero, the LSP opportunistically opens or splices
channels whenever an incoming payment would exceed the client's current
inbound capacity. Phoenix + ACINQ pioneered this with splice-on-demand,
where existing channels are "topped up" rather than opening new ones,
amortizing on-chain fees over many payments.

## Walkthrough / mechanics

Splice-on-demand vs new-channel-on-demand:

- **New channel**: each top-up is a fresh funding tx. Simple, but burns
  on-chain fees per top-up and forces the wallet to track many small
  channels.
- **Splice**: the LSP modifies the existing channel's funding output via
  a splice tx (BOLT 2 splice extension). State is preserved, on-chain
  fees amortized, channel count stays at 1.

For splice: the funding tx is replaced via a coordinated 2-of-2 spend
that spends the old funding output and creates a new one at the higher
capacity. Both parties sign. The new commitment immediately reflects
the additional inbound liquidity.

Mid-payment timeline (splice variant):

```
T+0:   Incoming HTLC reaches LSP. Amount > client_inbound_capacity.
T+0:   LSP holds HTLC, prepares splice tx:
         input: existing channel funding_outpoint
         output: new funding output (multisig 2-of-2)
                 new_capacity = old_capacity + delta
T+0.5: LSP and client exchange splice signatures (BOLT 2 update_*).
T+0.7: Splice tx broadcast, optionally accepted as 0-conf for HTLC fwd.
T+1.0: LSP forwards HTLC to client via the (now larger) channel.
T+1.2: Client releases preimage; LSP settles upstream and collects fees.
```

Fee taxonomy:

- **Splicing fee**: one-time on-chain mining fee, split per LSP policy
  (often LSP pays first M splices then user pays).
- **Service fee**: an msat-denominated proportional fee (e.g. 0.4% of
  inbound capacity added).
- **Liquidity fee**: per-block holding fee for the additional inbound
  while it remains unused.

## Worked example

Phoenix + ACINQ flow for a user already having a 200k sat channel:

```
User has channel with ACINQ:
  capacity = 200_000 sat
  local_balance = 50_000 sat
  inbound_capacity = 150_000 sat

Incoming payment: 500_000 sat.
ACINQ inspects -> exceeds inbound by 350k sat.
ACINQ initiates splice: add 400k sat (rounded up for headroom + fee margin).

Splice flow:
  ACINQ proposes splice_init { contribution: 400000, feerate: 8 sat/vB }.
  Client signs commit_signed for new state.
  ACINQ broadcasts splice tx.
  ACINQ forwards 0-conf 500k sat HTLC over the new state.

Resulting channel (after splice confirms):
  capacity = 600_000 sat
  local_balance = 550_000 sat (50k existing + 500k just received)
  inbound_capacity = 50_000 sat
  fee charged: 0.4% of 400k = 1600 sat (deducted from incoming).
```

For new-channel-on-pay (BLIP-51 style without splice), the LSP opens a
brand-new channel each time, which is simpler but accumulates UTXO
proliferation. BLIP-52 JIT is the open-spec version of this pattern.

## Common bugs / pitfalls

- Fee surprise: if the user's incoming payment is small but capacity must
  be added, the splice fee ratio can exceed the payment value. LSPs must
  cap or refuse such operations and surface a clear error.
- Splice race: a second incoming HTLC arrives mid-splice. Implementations
  must either queue or handle multi-HTLC splice (BOLT 2 splice supports it).
- Reserve drift: post-splice, the 1% reserve may bring channel below
  HTLC dust limits if not recomputed. Always recalculate reserves after
  splice.
- Failure between broadcast and confirm: if the splice tx is replaced by
  RBF or evicted, the channel state is ambiguous. Mitigated by TRUC v3
  + ephemeral anchors.
- Wallet that doesn't support splice (`option_splice` feature bit) cannot
  participate; LSP must fall back to a new channel or refuse the JIT
  payment.

## References

- BLIP-51: https://github.com/lightning/blips/blob/master/blip-0051.md
- BLIP-52: https://github.com/lightning/blips/blob/master/blip-0052.md
- ACINQ splice-on-demand white paper (https://acinq.co)
- BOLT 2 splice extension draft
