# Dandelion Stem-Fluff Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/privacy/dandelion`.
> Canonical source: https://arxiv.org/abs/1701.04439 (Dandelion); https://arxiv.org/abs/1805.11060 (Dandelion++)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/privacy/dandelion/SKILL.md

## Concept

Dandelion (and the strengthened Dandelion++) is a two-phase relay protocol
to **defeat first-spy IP deanonymisation** of transaction sources. A new
tx enters a "stem" phase where it is forwarded along a single random
out-edge (looks like private one-to-one routing). After a random delay or
hop count it transitions to "fluff" — the standard `inv` flooding that
bitcoind already uses. An adversary doing first-spy clustering must guess
which fluff source was the stem origin; the confusion grows with stem depth.

## Walkthrough / mechanics

### Topology

Each node maintains an **anonymity graph** independent of the gossip
graph. In Dandelion++, each peer randomly chooses 2 out-edges from its
inbound set as its stem-relay edges. The graph is rotated every \~10 min
("epoch"). A tx received over the stem graph follows a stem out-edge of
the receiving node; a tx received over the gossip graph fluffs normally.

### State machine for a transaction

```
            broadcast()              receive(stem,inv)
   [STATE_NULL] ----> [STEM:tx, fwd-out=stem-edge, timer=Exp(rate)]
                                |             timer fires OR fluff-flag set
                                v
                       [FLUFF: inv to all peers]
```

Each stem hop independently rolls a Bernoulli `p_fluff` (e.g. 0.1) per
forward. If `1` -> fluff at this node. Else forward to its stem-out edge.
This gives a geometric distribution of stem length with mean `1/p_fluff`.

### Embargo timer

Even if no node fluffs, an upper bound is enforced. Each node sets a
random "embargo" timer (e.g. \~20 s exponential) when it receives the tx
on the stem; if the timer fires before it has seen the tx fluff back, it
fluffs itself. This prevents black-hole stems.

### Anti-deanonymisation property

If the adversary controls a subset of nodes, the probability that one of
them is the **fluff source** is `1 / (stem_depth)`. With `p_fluff=0.1`,
stem depth ≈ 10 hops, so a passive 10 % adversary needs ~10x more
observations to localize the source.

## Worked example

Alice broadcasts tx `T`:

```
Alice -> N1 (STEM)
N1 picks stem-out=N4. Coin flip: 0.1 fluff? no. Forward to N4.
N4 picks stem-out=N9. Coin flip: 0.1 fluff? yes. FLUFF.
N4 -> { N7, N9, N12, ... } via inv.
```

A first-spy adversary observing `N7,N9,N12` simultaneously concludes "N4
sent it" but cannot tell whether N4 originated `T` or simply fluffed it on
behalf of an upstream stem. Stem depth was 2 -> deanon weight 1/2.

## Common pitfalls

- **Bitcoin Core has not adopted Dandelion**. The mainline transaction
  relay still uses straight `inv` flooding (with the BIP35 INV trickle and
  per-peer poisson delay). Forks like btcd had experimental support; some
  nodes (Grin, Monero) implement the protocol natively.
- **Embargo too short** -> fluff dominates and stem privacy is lost.
  Embargo too long -> tx propagation latency increases substantially
  (~20 s).
- **Single-edge graph** is brittle: an adversary that controls a node's
  unique stem-out edge **always** sees the tx. Dandelion++ moved to **two**
  out-edges to mitigate.
- **Reorg-resistant stem**: a tx that leaves the stem and is later evicted
  from mempool must NOT re-enter as stem; that breaks the once-per-epoch
  invariant and gives the adversary repeated samples.
- **Replacement / RBF**: bumping a stem-phase tx with RBF re-broadcasts
  with a new wtxid. Wallets must explicitly re-stem for the bump to gain
  the same privacy.

## References

- Bojja Venkatakrishnan, Fanti, Viswanath. "Dandelion: Redesigning the
  Bitcoin Network for Anonymity" SIGMETRICS 2017.
- Fanti et al. "Dandelion++: Lightweight Cryptocurrency Networking with
  Formal Anonymity Guarantees" SIGMETRICS 2018.
- BIP156 (proposed but not merged).
