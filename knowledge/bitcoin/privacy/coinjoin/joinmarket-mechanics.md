# JoinMarket Mechanics - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/privacy/coinjoin`.
> Canonical source: https://github.com/JoinMarket-Org/joinmarket-clientserver
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/privacy/coinjoin/SKILL.md

## Concept

JoinMarket is a maker/taker CoinJoin marketplace with no central
coordinator. Makers ("yield generators") publish offer prices over
IRC and Tor onion services; takers ("snickers") pay makers a small
fee for liquidity in their CoinJoin. The market mechanism replaces
trust in a coordinator with economic incentives: a maker who refuses
to sign loses fee revenue and is shadow-banned via reputation. The
trade-off is smaller anonymity sets (typical: 3-9 participants) and
longer rounds (minutes to hours) compared to centralised coordinator
designs.

## Walkthrough / mechanics

Makers run `yg-privacyenhanced` continuously:

```
1. Connect to all configured IRC servers (over Tor onion).
2. Publish an !ABSORDER offer:
       order_id, min_size, max_size, txfee, cjfee_abs, cjfee_rel
3. Wait for takers to query / fill.
```

Takers run `sendpayment.py` or `tumbler.py`:

```
1. Connect, fetch offerbook, choose N makers based on
      cost = makers' fees + on-chain fee share.
2. !FILL each maker with the desired CoinJoin amount.
3. Each maker responds with input commitments + nonces.
4. Sender constructs the multi-input PoDLE-validated CoinJoin.
5. Each maker provides their input UTXOs and one output of the
   negotiated equal-amount denomination.
6. Taker assembles tx, all parties sign, broadcasts.
```

PoDLE (Proof of Discrete Log Equivalence) is a Sybil-defence: each
input is committed to a non-malleable identifier so the same UTXO
cannot be used to enter multiple parallel rounds. A maker who tries
to be in two CoinJoins simultaneously with the same UTXO is detected
and banned.

Output denominations: equal among all participants. A 4-maker join
of 0.1 BTC produces 5 outputs of 0.1 BTC (taker + 4 makers), each
maker getting their fee revenue back to a separate change output.

## Worked example  (encoded data, real txs, configs)

Sample taker run:

```
$ python sendpayment.py -N 4 -m 4 wallet.dat 0.5 \
        bc1q<recipient>
Selecting 4 makers from offerbook...
Selected: J55Ufq, J5tD0d, J5XYZ, J5GHt
Total fees: 12000 sat (3000 each maker average) + 4000 sat tx fee
Initiating round...
Round complete in 142s.
Tx broadcast: bb05...e1f
```

Resulting tx:

```
inputs:  taker UTXO 0.6, maker1 0.12, maker2 0.4, maker3 0.55, maker4 0.31
outputs: 0.5 to recipient
         0.5 to maker1 mix output
         0.5 to maker2 mix output
         0.5 to maker3 mix output
         0.5 to maker4 mix output
         0.0997 taker change
         0.0098 maker1 change (incl fee)
         ...
fee: ~4_000 sat at 30 sat/vB
```

Anonymity-set per output: 5 (taker + 4 makers). Considerably smaller
than Wasabi rounds (50-150) but distributed: each round draws from a
different pool of makers, so over many rounds the effective set grows.

Tumbler mode `tumbler.py` automates 4-12 sequential rounds with
randomised delays (1 minute to 24 hours per stage), splitting the
amount across 3-5 destination addresses.

## Trade-offs / pitfalls

- **Anonymity set size**. 4-9 participants per round is small. The
  Tumbler aggregates rounds but timing analysis can still re-cluster.
- **Maker concentration**. Half the offered liquidity often comes
  from a few large makers. A taker who picks the cheapest 4 makers
  may end up with 4 colluding identities and an effective set of 1.
  Defence: random selection across all eligible offers, not just
  cheapest.
- **PoDLE collisions**. Honest takers may share a UTXO with a
  maker by coincidence (very rare). The protocol auto-retries.
- **Fee market**. Makers compete on fee, race to bottom; very low
  fees mean very few makers, harming privacy. Yields fluctuate
  60-200 sat/BTC per round.
- **IRC liveness**. JoinMarket relies on a few onion-hosted IRC
  servers. When two of three were briefly down in 2023, rounds
  ground to a halt. Multi-IRC config is essential.
- **Wallet-cluster leakage**. JoinMarket wallets use 5 internal
  "mixdepths" to segregate funds. Spending from a deeper mixdepth
  alongside a fresher one re-clusters them; the GUI does not
  prevent this.
- **No equal-output guarantee for taker**. The taker gets the
  recipient amount AND change to multiple addresses. Change outputs
  are non-uniform and easy to identify as "the taker's change" in
  retrospect.
- **Legal note**. Running a maker (yg) is essentially providing
  liquidity to anonymous CoinJoins; jurisdictional treatment varies.

## References

- JoinMarket repo: https://github.com/JoinMarket-Org/joinmarket-clientserver
- PoDLE: https://github.com/AdamISZ/PoDLE
- Tumbler guide: https://github.com/JoinMarket-Org/joinmarket-clientserver/blob/master/docs/tumblerguide.md
