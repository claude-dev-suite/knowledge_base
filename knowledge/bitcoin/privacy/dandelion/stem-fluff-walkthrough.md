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
graph. In Dandelion++, each peer randomly chooses 2 of its *outbound*
peers as its stem-relay edges (`DANDELION_MAX_DESTINATIONS = 2` in the
BIP156 reference implementation) and maps each inbound peer to one of
them. The graph is rotated every \~10 min ("epoch"). A tx received over
the stem graph follows the stem out-edge its inbound peer is mapped to;
a tx received over the gossip graph fluffs normally.

### State machine for a transaction

```
            broadcast()              receive(stem,inv)
   [STATE_NULL] ----> [STEM:tx, fwd-out=stem-edge, timer=Exp(rate)]
                                |             timer fires OR fluff-flag set
                                v
                       [FLUFF: inv to all peers]
```

BIP156 makes the fluff decision **per transmission**: before relaying a
Dandelion tx the node flips a biased coin and SHOULD fluff with
probability `q = 10%`. The Dandelion++ paper instead makes it **per node,
per epoch** - a node is a diffuser or a relayer for *all* transactions in
that epoch, decided from a hash of its own identity and the epoch number,
so the outcome does not depend on the transaction. Either way the
expected stem length is `1/q`; the paper's mainnet experiments used
`q = 0.1` or `q = 0.2`, i.e. 10 or 5 hops.

### Embargo timer

Even if no node fluffs, an upper bound is enforced. Each node sets an
"embargo" timer when it receives the tx on the stem; if the timer fires
before it has seen the tx arrive as a normal transaction, it fluffs
itself. This prevents black-hole stems. BIP156 only requires the timer be
random and gives no value; its reference implementation uses a 10 s
minimum plus an exponential draw with a 20 s mean
(`DANDELION_EMBARGO_MINIMUM = 10`, `DANDELION_EMBARGO_AVG_ADD = 20`).
Grin instead ships a *fixed* 180 s default (`DANDELION_EMBARGO_SECS`) and
its own docs flag the consequence: with identical timers everywhere, the
node whose embargo expires first under a black-hole attack is usually the
originator, which unmasks it.

### Anti-deanonymisation property

The guarantee is stated as the **precision and recall** of the
adversary's transaction->IP map, not as a per-hop probability. For a spy
fraction `p`, no spreading protocol can achieve expected recall below `p`
or expected precision below `p²`. Dandelion++ aims at both: the paper
states that for certain classes of Byzantine adversaries it achieves the
recall bound and comes within a *logarithmic* factor of the precision
bound (§3.2, shown in §4.2). Thm. 1 is a separate statement - on a random
4-regular anonymity graph whose topology the adversary does not know, the
cheap first-spy estimator is within a constant factor of the *optimal*
estimator (`D_OPT <= 8*D_FS + 6p² + O(p³)`), which is why the paper's
simulated first-spy curves count as near-precision-optimal. Two practical
corollaries from the same paper: a *larger* `q` raises the adversary's
precision, and keeping `q <= 0.2` caps the increase in average precision
at about 0.1 even when spies open outbound edges to every honest node.
What blunts repeated observations is pseudorandom one-to-one
forwarding - transactions from one source retrace the same path within an
epoch, so extra samples buy the adversary little.

## Worked example

Alice broadcasts tx `T`:

```
Alice -> N1 (STEM)
N1 is a relayer this epoch; Alice's inbound edge maps to N4. Forward.
N4 drew diffuser mode this epoch (probability q=0.1). FLUFF.
N4 -> { N7, N9, N12, ... } via inv.
```

A first-spy adversary observing `N7,N9,N12` simultaneously concludes "N4
sent it" but cannot tell whether N4 originated `T` or simply fluffed it on
behalf of an upstream stem - the first-spy estimate is wrong by two hops.
Because N4 stays a diffuser and N1 keeps the same route until the epoch
rolls over, further transactions from Alice retrace this same path rather
than handing the adversary an independent sample.

## Common pitfalls

- **Bitcoin Core never adopted Dandelion.** BIP156 is listed with status
  **Closed** in the BIPs repo README (checked September 2026). The mainline
  transaction relay still uses straight `inv` flooding, with announcements
  trickled on a randomised exponential timer - an average 5 s for inbound
  peers (shared per network) and 2 s per outbound peer
  (`INBOUND_INVENTORY_BROADCAST_INTERVAL` and
  `OUTBOUND_INVENTORY_BROADCAST_INTERVAL` in `net_processing.cpp` on
  master, September 2026). That delay is not a BIP35 feature - BIP35 is
  the `mempool` message and nothing else. The only Bitcoin implementation
  of Dandelion was the BIP156 reference fork of Core
  (`dandelion-org/bitcoin`), never merged upstream. Outside Bitcoin,
  **Grin** and **Monero** both implement Dandelion++ natively (present on
  master, checked September 2026); `btcsuite/btcd` carries no Dandelion
  code, commits or issues (checked September 2026).
- **Embargo too short** -> fluff dominates and stem privacy is lost.
  Embargo too long -> tx propagation latency increases substantially
  (tens of seconds under the BIP156 reference timer, minutes under Grin's
  180 s default).
- **Single-edge graph** is brittle: an adversary that controls a node's
  unique stem-out edge **always** sees the tx. Dandelion++ moved to **two**
  out-edges to mitigate.
- **Reorg-resistant stem**: a tx that leaves the stem and is later evicted
  from mempool must NOT re-enter as stem; that breaks the once-per-epoch
  invariant and gives the adversary repeated samples.
- **Replacement / RBF**: bumping a stem-phase tx with RBF re-broadcasts
  with a new wtxid. Wallets must explicitly re-stem for the bump to gain
  the same privacy.

## What Bitcoin Core shipped instead

Core 31.0 (19 April 2026) added `-privatebroadcast`, a boolean node option
(default `false`, checked on master September 2026). It addresses the same
first-spy threat by a different route: transactions submitted via
`sendrawtransaction` are broadcast over **short-lived connections through
the Tor or I2P networks only**, and are not placed in the local mempool
first. Transactions submitted through the wallet are unaffected.

Per the 31.0 release notes this buys two properties:

1. The originator's IP address is never known to the recipients.
2. Two otherwise unrelated transactions from the same originator are not
   linkable, because a **separate connection is used per transaction**.

Note the contrast with Dandelion++: there is no stem graph, no epoch
rotation and no embargo timer. Unlinkability comes from connection
isolation over an anonymity network rather than from routing confusion
among honest peers, so — unlike Dandelion++ — it does not depend on other
nodes running the same code.

Two RPCs landed alongside it in 31.0:

- `getprivatebroadcastinfo` — reports transactions currently being
  privately broadcast.
- `abortprivatebroadcast` — removes matching transactions from the private
  broadcast queue.

Operational constraints, from `init.cpp` on master (September 2026):

- Startup fails if neither Tor nor I2P is reachable.
- Incompatible with `-connect`, since private broadcast must open its own
  connections to randomly chosen Tor/I2P peers. Use `-maxconnections=0
  -addnode=...` instead.
- Warns when `-proxyrandomize=0`: Tor circuits for private broadcast
  connections may then be correlated with other Tor connections.
- Capped at 64 concurrent private broadcast connections
  (`MAX_PRIVATE_BROADCAST_CONNECTIONS`); beyond that, new ones pause.

**Do not use the 31.0 build for this.** 31.0 leaked the originator's IP
address: under certain circumstances connections were made over clearnet
rather than the enabled privacy network. Fixed in **31.1 (July 2026)**.

## References

- Bojja Venkatakrishnan, Fanti, Viswanath. "Dandelion: Redesigning the
  Bitcoin Network for Anonymity" SIGMETRICS 2017.
- Fanti et al. "Dandelion++: Lightweight Cryptocurrency Networking with
  Formal Anonymity Guarantees" SIGMETRICS 2018.
- BIP156 (header reads `Assigned: 2017-06-09`; status **Closed** in the
  BIPs repo README as of September 2026, never merged into Bitcoin Core).
- BIP156 reference implementation, `dandelion-org/bitcoin` branch
  `dandelion` (`DANDELION_*` constants in `src/net.h`).
- Grin, `doc/dandelion/dandelion.md` - Dandelion++ in Grin; source of the
  `q`, epoch and `DANDELION_EMBARGO_SECS` defaults quoted above (checked
  September 2026).
- Bitcoin Core 31.0 release notes, "P2P and network changes"
  (`-privatebroadcast`, PR #29415; RPCs, PR #34329).
- Bitcoin Core 31.1 release notes, "PrivateBroadcast" (clearnet IP leak fix).
