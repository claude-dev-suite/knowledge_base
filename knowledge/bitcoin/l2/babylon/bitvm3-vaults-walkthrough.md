# Babylon BitVM3 Vaults Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/babylon`.
> Canonical source: https://docs.babylonlabs.io/trustless-bitcoin-vault/
> (the old docs.babylonchain.io domain no longer resolves, checked
> 16 Sep 2026)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/babylon/SKILL.md

> **Superseded design — read the status note first.** This article
> describes the BitVM3-framed vault sketch from Babylon's 6 August 2025
> announcement. The Trustless Bitcoin Vaults (TBV) that actually reached
> public testnet use a different proof system (BABE) and serve a
> different purpose (Ethereum DeFi collateral, not staking vaults). See
> [Status: what actually shipped](#status-what-actually-shipped-as-of-september-2026)
> before relying on the mechanics below.

## Concept

BitVM3 (a Babylon team specification building on Linus's BitVM2) was the
proposed framework for **Bitcoin-secured vaults** in Babylon's
Genesis-era staking. It used general-purpose Bitcoin script computation
to enforce slashing, withdrawal delays, and yield-distribution rules
without trusting any covenant emulator.

While BitVM2 supports two-party fraud proofs, BitVM3 generalizes to
N-party vault commitments with optimistic withdrawal + slashing.

## Walkthrough / mechanics

### Goal

Replace covenant emulator quorum (used in current Babylon staking) with
a fully on-chain fraud-proof mechanism, bringing BTC staking closer to
fully trustless.

### Components

- **Vault output**: Taproot UTXO holding staked BTC.
- **Operator**: optimistic claim-maker who provides withdrawal liquidity
  to stakers and reclaims via fraud-proof challenge.
- **Watcher**: any party who can challenge incorrect operator claims.

### Vault script structure

```
P2TR(
  internal_key = staker_pk + operator_pk MuSig2,
  scripts:
    leaf A: optimistic exit (operator_sig + claim_commitment)
    leaf B: BitVM3 challenge -> bisection -> single-step verification
    leaf C: timelock refund to staker (long delay)
)
```

### Withdrawal flow (optimistic)

1. Staker initiates unstake on Babylon chain.
2. Operator front-pays staker the BTC value.
3. Operator commits a BitVM3 claim: a cryptographic commitment to the
   complete fraud-proof transcript including:
   - The unstaking tx hash on Babylon.
   - The expected vault UTXO.
   - The claim amount.
4. Challenge window: any watcher can dispute via on-chain bisection.
5. If unchallenged: operator spends vault via leaf A, recovering capital.
6. If challenged: BitVM3 protocol drives convergence to a single
   computational step, verified on Bitcoin script.

### Slashing flow

If a finality provider misbehaves:

1. Watcher detects equivocation, extracts FP secret (EOTS).
2. Watcher constructs slashing tx claim using BitVM3.
3. Watcher commits to: "this FP signed two conflicting blocks at height h".
4. The challenge protocol evaluates the proof on Bitcoin.
5. On success: slashing tx finalizes, burning a fraction of staked BTC.

### How BitVM3 differs from BitVM2

- **N-party support**: vault has many stakers; BitVM3 protocol
  shards challenges per vault entry.
- **Larger commitment overhead** but smaller worst-case challenge depth
  (logarithmic bisection over states).
- **Stateless verification** of slashing rules in single Bitcoin tx.

### Status: what actually shipped (as of September 2026)

The staking protocol on Babylon Genesis mainnet still uses the covenant
emulator quorum; BitVM3-based staking vaults were never deployed. What
Babylon shipped under the "Trustless Bitcoin Vaults" name is a different
product, on public testnet as of 16 September 2026:

- **Networks**: Bitcoin signet + Ethereum testnet. Borrowed assets are
  mock tokens on Sepolia. No mainnet; the 7 October 2025 vault-first
  roadmap post targeted vault mainnet for "early next year" and that
  has not happened.
- **Purpose**: native BTC as collateral for Ethereum DeFi, not
  liquidity for BTC-secured PoS. The only registered application is the
  Aave v4 borrowing adapter.
- **Proof system**: **BABE**, not BitVM3. The paper — "BABE: Verifying
  Proofs on Bitcoin Made 1000x Cheaper" (Garg, Kolonelos, Sergeevitch,
  Sridhar, Tse; UC Berkeley / Babylon Labs / Byzantine Research /
  Stanford, January 2026) — keeps BitVM3's on-chain cost but cuts
  off-chain storage and setup by three orders of magnitude, against
  BitVM3's ~42 GiB per garbled circuit. BABE combines witness
  encryption for linear pairing relations with a garbled circuit for EC
  scalar multiplication, making Groth16 verification practical in
  existing Bitcoin script with no fork.
- **Roles**: depositor, one Vault Provider per vault, App Keepers,
  Universal Challengers, and a *transitional* Security Council. None
  custodies BTC; every claim is challengeable and the depositor is
  always an authorized claimer.
- **Redemption**: an SP1 proof of the Ethereum redemption event is
  verified on Bitcoin via BABE, then a challenge window of 432 BTC
  blocks (~3 days) runs before payout; the claimer gets 108 BTC blocks
  (~18 hours) to disprove a challenge.
- **Depositor fallbacks**: self-claim with a Winternitz one-time
  signature (WOTS) committed at peg-in, and a relative-CSV refund on
  the Pre-PegIn output (2016 BTC blocks, ~14 days, for the current
  parameter version) if activation never completes.
- **Testnet caps**: 0.01–0.4 BTC per vault, 0.4 BTC per position and
  per address, 10 BTC total across the Aave v4 application, 78 %
  collateral factor.

Source: https://docs.babylonlabs.io/trustless-bitcoin-vault/ (fetched
16 September 2026).

## Worked example

Alice stakes 1 BTC. She unstakes; operator front-pays 0.99 BTC
immediately.

```
1. Operator broadcasts BitVM3 claim tx with commitment to:
   - Unstaking tx hash on Babylon: H_b
   - Expected vault UTXO: outpoint X
   - Amount: 1 BTC.
2. Challenge window 14 days.
3. No challenge -> operator spends X to recover 1 BTC, less fee.
```

Slashing example: FP Jacqueline equivocates at height 1234.

```
1. Watcher submits BitVM3 slashing claim.
2. Operator (or any watcher) provides witnesses for FP equivocation
   and EOTS-extracted secret.
3. Bisection rounds verify fraud: was sig_1 valid? was sig_2 valid?
   are they on conflicting blocks at same height?
4. Final single-step on-chain verification confirms misbehavior.
5. Slash tx finalizes, sending fraction of stake to slashing burn addr.
```

## Common pitfalls

- **Implementation complexity**: BitVM3 requires sophisticated state
  commitments and bisection logic; bugs may freeze funds.
- **Challenge data size**: bisection rounds can require multi-KB witness
  data; high fees during congestion.
- **Capital lockup**: operators must hold large reserve to front-pay
  unstakes; capital efficiency is poor compared to true smart contracts.
- **Watcher incentive**: if no watcher challenges, fraudulent operator
  claims succeed; watcher reward must exceed challenge cost.
- **Compatibility with EOTS**: Schnorr-extractable secrets must be
  cleanly verifiable in BitVM3 instructions; complicates the
  state-transition function.

## References

- Linus, Aumayr, Maffei. "BitVM2: Permissionless Verification" 2024.
- Babylon BitVM3 design notes.
- Garg, Kolonelos, Sergeevitch, Sridhar, Tse. "BABE: Verifying Proofs on
  Bitcoin Made 1000x Cheaper", January 2026 — the construction TBV
  actually shipped on:
  https://docs.babylonlabs.io/trustless-bitcoin-vault/research/babe_verification/
- "EOTS for finality providers" research note.
