# Babylon BitVM3 Vaults Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/babylon`.
> Canonical source: https://docs.babylonchain.io/ (BitVM3 / Genesis vaults)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/babylon/SKILL.md

## Concept

BitVM3 (a Babylon team specification building on Linus's BitVM2) is the
proposed framework for **Bitcoin-secured vaults** in Babylon's
Genesis-era staking. It uses general-purpose Bitcoin script computation
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

### Status (as of 2026)

BitVM3 is a research roadmap item; production Babylon uses covenant
emulator. Genesis-era vaults aim to deploy BitVM3 in 2026 testnet.

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
- "EOTS for finality providers" research note.
