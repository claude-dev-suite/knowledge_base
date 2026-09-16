# Botanix Spiderchain Deep Dive (Post-Mortem)

> Phase B article. Companion to dev-suite skill `bitcoin/l2/botanix`.
> Canonical source: https://botanixlabs.com/
> Wind-down post: https://botanixlabs.com/blog/winding-down-botanix-what-we-built-and-why-we-re-stopping
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/botanix/SKILL.md

> **RETIRED - architecture study only.** The Botanix network ran on
> mainnet from 1 July 2025 until the wind-down announced 5 June 2026,
> staged from 1 July 2026 with final network shutdown scheduled no later
> than 1 August 2026. Everything below describes how the Spiderchain was
> designed and how it operated while live. None of these flows are
> usable today: there is no coordinator to request a deposit address
> from, and no L2 to mint against.

## Status: network retired

As of September 2026 the Botanix L2 is shut down.

- **5 June 2026** - Botanix Labs published "Winding Down Botanix: What
  We Built, and Why We're Stopping".
- **1 July 2026** - target wind-down date. The botanixlabs.com banner
  reads "Botanix is shutting down on July 1, 2026 ... Please withdraw
  your assets before this deadline."
- **15 July 2026** - two-week grace period per the official post.
- **1 August 2026** - "if needed, 2 weeks more until August 1 when the
  final shutdown of the network occurs". Afterwards "the federation
  will sweep the remaining Bitcoin and the company starts its dissolving
  process"; capital return was expected to take until October 2026.

Secondary coverage (Decrypt, crypto.news, cryptotimes.io) reported a
9 July 2026 withdrawal cutoff and that non-BTC tokens become permanently
unrecoverable. Neither appears in the official post; prefer the dates
above.

The Spiderchain itself did not fail. The post reports "a year of mainnet
operation with one hundred percent uptime and zero security incidents",
25 million transactions and 200,000 wallets across the mainnet year
(July 2025 - 2026), after nearly four years of building. The stated
reasons are demand-side: mistiming the Bitcoin community's appetite for
L2 utility, token-incentive bootstrapping no longer working, wrapped BTC
on an existing L2 such as Arbitrum being sufficient for most Bitcoin
DeFi demand, and on-chain growth consolidating toward centralized
venues.

Note that docs.botanixlabs.com was still serving present-tense live
network documentation with a forward roadmap as of September 2026. It is
stale vendor documentation, not evidence the chain is running.

## Concept

Botanix introduced the **Spiderchain**: a network of decentralized
multisigs that custody Bitcoin and back an EVM-equivalent L2. Instead of
a single static federation, the Spiderchain is a rotating set of
**orchestrators** chosen via PoS (PoB - Proof-of-Bitcoin) staking. Each
new BTC deposit is held by a fresh randomly-sampled multisig drawn from
the active orchestrator pool, dramatically reducing the attack surface
compared to a single federated wallet.

## Walkthrough / mechanics

### Orchestrators

- **Stake**: orchestrators lock collateral (initially BTC) to participate.
- **Rotation**: each "epoch" (e.g. 1000 BTC blocks), the active set is
  resampled from registered stakers based on stake weight.
- **Multisig role**: orchestrators sign Bitcoin transactions controlling
  peg UTXOs.

### Rotating multisigs ("spider legs")

When a user deposits BTC:

1. The deposit script picks `K` orchestrators from the current epoch's
   active set, e.g. K=11.
2. Of those, a (K, N) FROST threshold address is created — say 7-of-11.
3. The deposit lands in this fresh multisig.
4. Each subsequent epoch, the deposit may be **migrated** to a new
   multisig (re-randomized) to maintain low cumulative compromise risk.

### Risk amortization property

Compromising the chain requires breaking many independent multisigs
simultaneously, each with its own threshold. If an attacker compromises
1 multisig (say corrupting 7 of its 11 orchestrators), only the BTC in
THAT multisig is at risk — not the entire peg.

Statistically, with R rotation rounds and `f` fraction of malicious
orchestrators, the probability that any single multisig is fully
compromised (>=t out of K malicious) follows binomial distribution; the
compounded probability across all multisigs is the system's security
margin.

### EVM execution

- L2 EVM-equivalent client running standard JSON-RPC.
- Native gas token: BTC (1 sat = 10^10 wei).
- All Solidity dApps deploy unchanged.

### Peg-in flow

1. User requests deposit address from Spiderchain coordinator.
2. Coordinator selects random K orchestrators, returns Taproot multisig
   address for K-of-11 FROST.
3. User deposits BTC.
4. After confs, orchestrators sign L2 mint tx crediting user with
   m-BTC.

### Peg-out flow

1. User burns m-BTC on L2.
2. Spiderchain consensus selects which UTXO(s) cover the withdrawal.
3. Selected multisigs co-sign Bitcoin tx paying user.
4. After 1 BTC conf, withdrawal complete.

### Slashing

Orchestrators caught misbehaving (e.g. signing fraudulent payouts) lose
their staked BTC. Detection mechanisms include:

- On-chain proof of incorrect L2 state attestations.
- Signature on conflicting messages (equivocation).

## Worked example (historical)

Alice deposits 1 BTC to Botanix, while the network was live.

```
1. Alice -> coordinator: "deposit 1 BTC".
2. Coordinator: selects orchestrators {O3, O7, O11, O14, O18, O22, O25,
   O29, O32, O36, O40} (K=11 random).
3. FROST DKG over those 11 produces aggregated key Q.
4. Address P2TR(Q) returned to Alice.
5. Alice broadcasts BTC tx: 1 BTC -> P2TR(Q).
6. After confs, 7-of-11 of the orchestrators sign L2 mint tx ->
   Alice gets 1 m-BTC on Botanix L2.
```

Three months later, Alice's deposit has been rotated 3 times; current
custody is some new (K, N) tuple of fresh orchestrators.

## Common pitfalls

- **Coordinator availability**: deposit-address generation depends on
  coordinator. If it fails, deposits cannot start. Multiple coordinators
  required.
- **Rotation cost**: each migration is an on-chain Bitcoin tx
  consolidating UTXOs. Frequent rotations bloat fees; protocol must
  balance rotation cadence vs. cost.
- **Liveness**: K-of-11 threshold means up to (K-t) orchestrators can
  drop. Above threshold loss = funds locked. Insurance/slashing pools
  fund recovery.
- **Privacy**: each Taproot multisig is unique; chain analysis can map
  Alice's deposit to her L2 mint via timing/amount.
- **Stake reuse**: orchestrators reusing the same stake across multiple
  multisigs scales linear weight; protocol may cap per-multisig
  representation.

## References

- Botanix Labs documentation (stale; still describes a live network).
- "Winding Down Botanix: What We Built, and Why We're Stopping" -
  Botanix Labs blog, 5 June 2026.
- "Spiderchain: A Decentralized Bitcoin Sidechain" — Willem Schroé 2023.
- FROST threshold Schnorr signatures (Komlo & Goldberg 2020).
