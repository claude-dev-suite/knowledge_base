# Rare Sat Classes - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/metaprotocols/ordinals`.
> Canonical source: https://docs.ordinals.com/overview.html#rarity
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/metaprotocols/ordinals/SKILL.md

## Concept

Casey Rodarmor's rarity hierarchy is a social convention layered on
top of ordinal numbering. It picks specific sats whose ordinal numbers
align with chain milestones (blocks, difficulty epochs, halvings,
cycles). Collectors trade these like numismatic coins. Beyond the
six base classes there are ad-hoc "named" categories such as Pizza
sats, Block 9 sats, Vintage, Palindromes, and Uncommon-but-named.

## Walkthrough / mechanics

```
Common      every sat that is not in any class below
Uncommon    first sat of every block             (~889 K eligible / day class)
Rare        first sat of every 2016-block epoch  (one per ~2 weeks)
Epic        first sat of every halving           (one per ~4 years)
Legendary   first sat of every cycle (6 halvings, ~24 years)
Mythic      sat 0 only (Satoshi's coinbase)
```

The classes are nested by construction: every Rare sat is also
Uncommon, every Epic is Rare, etc. Conventionally a sat is reported
under its highest class.

To classify a sat number `s`, find its block via the cumulative-
subsidy table, then check:

```
offset = s - first_sat(block)
if offset > 0: Common
elif block % 210_000 * 6 == 0: Legendary
elif block %  210_000     == 0: Epic
elif block %    2016      == 0: Rare
elif block != 0:                Uncommon
else:                           Mythic
```

The "cycle" period 6 * 210_000 = 1_260_000 blocks aligns halvings
with the difficulty epoch boundary (LCM of 210_000 and 2016 is
1_260_000). The first Legendary sat after Mythic appears at block
1_260_000, expected around 2032.

## Worked example  (encoded data, real txs, configs)

Identify the rarity of sat 1_968_750_000_000_000 (block 840_000):

```
block       = 840_000
offset      = 0                                 -> at least Uncommon
840_000 % 2016    = 0                           -> at least Rare
840_000 % 210_000 = 0                           -> at least Epic
840_000 % 1_260_000 = 840_000                   -> not Legendary
classification: Epic
```

That sat sits in the coinbase of the 2024 halving block. The Epic
sat from the previous halving (block 630_000, mined April 2020) sold
for 33.3 BTC at auction in 2023 according to Ordswap.

The named "Pizza sat" set comes from block 57_043, which contains the
sats Hanyecz spent on the famous 10_000-BTC pizza purchase. Any sat in
that input range is "Pizza-class" by convention. There are 9_989_991
such sats; tracking them requires walking every spend of that
coinbase.

ord client lookup:

```
$ ord wallet sats
+--------------------+----------+
| sat                | rarity   |
+--------------------+----------+
| 1968750000000000   | epic     |
| 623160000000000    | uncommon |
+--------------------+----------+
```

## Trade-offs / pitfalls

- Rare sats get destroyed accidentally when wallets ignore ordinal
  awareness. A standard wallet that spends an Epic sat as a normal
  payment input does not "lose" the sat (it still has the same
  number) but an indexer-unaware recipient may mix it with thousands
  of common sats and effectively burn its provenance for collectors.
- Coin selection in ord wallet must protect rare sats; bitcoin core
  wallet cannot do this without an external index.
- The rarity hierarchy is purely social. Bitcoin consensus does not
  treat sat 0 differently from any other sat; if Satoshi ever spent
  block 0 outputs, sat 0 would move just like any other.
- Premiums for rare sats fluctuate wildly with inscription cycles.
  Treat them as collectibles, not investments.
- "First sat of block" must account for empty / OP_RETURN-heavy
  blocks correctly: the offset is into the coinbase output, not the
  block itself.

## References

- Rarity table: https://docs.ordinals.com/inscriptions.html#rarity
- Pizza sat tracking: https://github.com/ordinals/ord/issues/1471
- ord wallet command: https://github.com/ordinals/ord/blob/master/docs/src/wallet.md
