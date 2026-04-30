# Ordinal Numbering Scheme - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/metaprotocols/ordinals`.
> Canonical source: https://docs.ordinals.com/overview.html
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/metaprotocols/ordinals/SKILL.md

## Concept

Ordinal theory assigns every satoshi a deterministic integer in the
order it was mined. Bitcoin consensus knows nothing about this number;
it is a client-side "interpretation" that any indexer must compute by
replaying the chain. Two clients that follow the spec agree on the
ordinal of every sat. The numbering enables sat-tracking, rare sat
collecting, and the entire inscription / BRC-20 / Runes ecosystem.

## Walkthrough / mechanics

Two rules define the scheme:

1. **Mining order**. The first sat of block N has the lowest unused
   number. The coinbase output of block 0 contains sats 0 through
   4_999_999_999 (50 BTC = 5_000_000_000 sats).
2. **FIFO transfer**. When a transaction has inputs `[I0, I1, ...]`
   and non-OP_RETURN outputs `[O0, O1, ...]`, sats are appended to
   outputs in order, draining each input fully before moving on.

To find the first sat of block N you sum the subsidy of all earlier
blocks. Subsidy halves every 210_000 blocks:

```
first_sat(N) = sum_{h=0..N-1} subsidy(h)
subsidy(h)   = floor(50e8 / 2^(h // 210_000))   sats
```

For block 1 the answer is 5_000_000_000 (the next sat after the
genesis subsidy). For block 209_999 the answer is 1_049_995_000_000_000
(close to 10.5 M BTC, which is exactly half the eventual supply).

## Worked example  (encoded data, real txs, configs)

Compute the first sat of block 840_000 (the 2024 halving):

```
era 0 (blocks      0..209_999):   210_000 * 50e8 = 1_050_000_000_000_000
era 1 (blocks 210_000..419_999):  210_000 * 25e8 =   525_000_000_000_000
era 2 (blocks 420_000..629_999):  210_000 * 12.5e8 = 262_500_000_000_000
era 3 (blocks 630_000..839_999):  210_000 * 6.25e8 = 131_250_000_000_000
                                            sum =   1_968_750_000_000_000
```

So sat 1_968_750_000_000_000 is the first sat of block 840_000 (a Rare
sat under Casey's hierarchy, since 840_000 is also the start of a new
difficulty epoch 840_000 / 2016 = 416.6... actually first sat of an
epoch occurs only at multiples of 2016; 840_000 is divisible by 2016
so this sat is also Rare, and being a halving boundary makes it Epic).

Decimal-form ordinal notation: an ordinal is sometimes written as
`block.offset_in_block`. Sat 1_968_750_000_000_000 is therefore
`840000.0`. The first 4-byte name for this sat under the alphabetical
naming scheme is also computable; the ord client exposes both.

To trace a sat through a transaction:

```
inputs:  I0 = [3000..3010), I1 = [7000..7050)   # 10 + 50 = 60 sats
outputs: O0 = 25 sats, O1 = 35 sats
```

O0 receives sats 3000..3010 (all of I0) followed by 7000..7014 (first
15 of I1). O1 receives 7015..7049. Fees are sats placed beyond the
last output and are credited to the coinbase of the mining block, so
they get re-numbered into the next coinbase output.

## Trade-offs / pitfalls

- The ord index is large: a recent full ord index sits above 1 TB and
  takes days to build. Light clients cannot compute ordinals locally.
- Different clients must implement FIFO identically. The spec is
  unambiguous, but historically there were bugs in early indexers
  around fee handling and sat-count for non-standard outputs.
- Bitcoin consensus does not enforce any ordinal property. Hard forks
  to consensus rules (e.g. dust limit changes) do not affect ordinals,
  but a soft fork that merged subsidy with fees could.
- Treating "rare" sats as economically valuable is a social
  convention; nothing prevents a wallet from spending a Mythic sat as
  fee.

## References

- Ordinal theory handbook: https://docs.ordinals.com/
- ord source: https://github.com/ordinals/ord
- BIP draft (informational only): https://github.com/ordinals/ord/blob/master/bip.mediawiki
