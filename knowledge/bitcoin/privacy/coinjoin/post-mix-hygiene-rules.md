# Post-Mix Hygiene Rules - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/privacy/coinjoin`.
> Canonical source: https://en.bitcoin.it/wiki/Privacy
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/privacy/coinjoin/SKILL.md

## Concept

A CoinJoin round provides "anonymity-at-the-moment-of-mixing".
Maintaining that privacy across subsequent spending requires
disciplined coin handling. The chain-analyst's primary re-attack is
re-clustering: detecting that two CoinJoined outputs are spent
together and inferring they belong to the same wallet. Without
hygiene, chained CoinJoins produce no real gain - all the work and
fees evaporate within a single careless transaction.

## Walkthrough / mechanics

The five rules that experienced operators follow:

1. **Never combine CoinJoined outputs with non-CoinJoined coins**.
   A single tx with one mixed input and one unmixed input creates a
   common-input cluster that links your KYC purchase wallet to your
   mixed coins permanently.

2. **Never combine outputs from different rounds in the same tx**.
   Two outputs of the same value from different rounds spent together
   prove they share a wallet, halving the effective anonymity set.
   Many wallet UIs auto-consolidate inputs to minimise fee; this
   behaviour must be disabled.

3. **No address reuse**. Receiving a payment to an already-used
   address links the new sender to the previous one. Fresh address
   per inbound payment.

4. **Wait between mix and spend**. Immediate spending after a
   CoinJoin allows timing analysis even if amounts and clusters
   are clean. Wait at least one block, ideally hours; randomise
   the delay.

5. **Mind the change**. CoinJoin produces "toxic change": non-
   standard amounts that cannot be mixed again because they break
   the equal-output property. Spending change with a fresh receive
   address pollutes that receive address with toxic provenance.
   Treat change as a separate, unmixed wallet branch.

Coin control discipline:

```
mixed:  outputs labelled "anonymity_set: 50, round_id: r1"
toxic:  change outputs from rounds, labelled
unmixed: deposits from KYC exchange, labelled

Spending rules:
  pay_recipient_from_mixed: select inputs ONLY from mixed pool
  consolidate_dust: NEVER do this (re-clusters by definition)
  remix_toxic: must go through a new CoinJoin first
```

Wasabi 2.0 implements this via "labels" and refuses to combine
inputs of different cluster IDs unless explicitly forced.

## Worked example  (encoded data, real txs, configs)

Bad scenario:

```
Wallet has:
  0.05 BTC mixed (anonymity_set 50)
  0.03 BTC mixed (anonymity_set 50, different round)
  0.02 BTC unmixed (from Coinbase deposit, KYC-tagged)

User wants to pay 0.07 BTC.
Naive coin-select picks all 3 inputs.
Tx: inputs 0.05 + 0.03 + 0.02, output 0.07 + change 0.0299

Analyst: input cluster -> 3 UTXOs from same wallet
       -> mixed coins now linked to KYC deposit
       -> historical CoinJoin gain destroyed
```

Good scenario:

```
Pay 0.04 BTC from mixed pool only:
  inputs: 0.05 mixed
  outputs: 0.04 to recipient, 0.0099 toxic change to fresh "post-mix change" wallet
  fee: 100 sat
The 0.03 mixed and 0.02 unmixed remain untouched, separate clusters.
Toxic change re-mixed in next CoinJoin round before further use.
```

Quantifying the leak: assume each round gives anonymity-set 50.
Mixing twice and never combining: effective set ~50 (best per round).
Combining two outputs from set-50 rounds: set drops to 1 against
their common owner.

Whirlpool (Samourai, now defunct) enforced "post-mix wallet
separation" by giving the user a distinct seed/account for post-mix
funds. Wasabi 2.0 enforces label-aware coin selection. Sparrow
manual labelling is available but user-driven.

## Trade-offs / pitfalls

- **Toxic change is unavoidable**. Any CoinJoin round produces
  change for participants whose input is not exactly a multiple of
  a standard denomination. Some wallets re-mix change automatically;
  others leave it as an explicit foot-gun.
- **Re-mixing has diminishing returns**. After 4-6 rounds, marginal
  privacy gain is small; chain-analysts assume the wallet "is mixed"
  and apply heuristics targeting the post-mix flow instead.
- **Fee correlations**. Using identical fee rates across many spends
  creates a soft fingerprint. Randomise fee rates within reason.
- **Receiver surveillance**. Even a perfectly clean CoinJoin output
  paid to a KYC-ed exchange links the mixed coins to identity. The
  exchange now knows your post-mix history and can cluster forward.
- **Wallet bugs**. Some wallets show "mixed" labels but still
  consolidate at internal change selection. Verify the actual tx
  before signing; coin-control UI is not always honest.
- **Lightning channel funding**. Funding an LN channel from mixed
  coins anchors that channel to the mix; on-chain force-close
  reveals the mixed origin. Use channel funding splices via PayJoin
  to break the link.
- **Block-explorer doxxing**. Sharing a tx ID in a public discussion
  ("see my CoinJoin here") instantly defeats the round - your
  identity is now glued to that tx graph.

## References

- Privacy heuristics catalogue: https://en.bitcoin.it/wiki/Privacy
- Wasabi labelling: https://docs.wasabiwallet.io/using-wasabi/Labels.html
- Toxic change discussion: https://gist.github.com/RubenSomsen/c9f0a92493e06b0e29acced59ca1f4fa
