# Bitlayer BTC Staking Architecture - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/bitlayer`.
> Canonical source: https://docs.bitlayer.org/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/bitlayer/SKILL.md

## Concept

Bitlayer is an EVM-compatible Bitcoin L2 with a unique **BTC staking**
mechanism: Bitcoin holders can lock BTC under a Taproot script with a
covenant-emulated structure that binds slashing rules to L2 behaviour.
Stakers earn BTR (Bitlayer's gas token) yield while their BTC remains on
Bitcoin in a self-custodial timelocked output.

The architecture combines BitVM2-style bridges, EVM execution, and a
Babylon-inspired staking module.

## Walkthrough / mechanics

### Staking script

A staker locks BTC into a Taproot output with two script paths:

1. **Unbonding path**: a relative-timelocked path returning BTC to staker
   after `T_unbond` blocks (e.g. 1000 BTC blocks).
2. **Slashing path**: spendable by a covenant emulator (multisig of
   Bitlayer validators) only on submission of evidence that the staker
   misbehaved on L2.

Concretely:

```
P2TR (
  internal_key = staker_pk + provider_pk_aggregate,   // MuSig2
  scripts:
    leaf 1: <T_unbond> OP_CSV OP_DROP <staker_pk> OP_CHECKSIG
    leaf 2: <slashing_address> covenant + <slashing_pubkey> OP_CHECKSIG
)
```

`slashing_address` is a fixed P2TR controlled by the validator set; a
slashing event burns or redirects part of the staked BTC.

### EVM L2 stack

- Sequencer + sequencer rotation.
- Prover (zk-SNARK / TEE-attested execution depending on phase).
- Standard EVM JSON-RPC.
- ERC-20 BTC representation: `BBTC` or similar.

### Validator set

L2 validators (Bitlayer's name varies: "guardians" or "rollers") are
selected by **stake-weighted lottery** using BTC stake. A validator
processes L2 transactions, signs state commitments, and may be slashed
if they sign a fraudulent state.

### Bridging via BitVM2

Like Citrea/Strata, peg-in deposits to operator-multisig with a refund
script-path. Peg-out involves BitVM2-secured operator claims with
challenge windows.

### Yield

Stakers earn BTR (or future BTC native yield) proportional to:

- Stake size.
- Stake duration commitment (longer locks earn more).
- Validator role (running validator vs delegating to one).

Yield is paid on L2 in BTR; users can swap to BTC via the bridge.

## Worked example

Alice locks 1 BTC for staking:

```
Bitcoin tx:
  Output: P2TR (
            internal = Alice_pk + ValidatorSet_aggregate,
            leaf 1 (unbonding): 1000 OP_CSV ...
            leaf 2 (slashing):  ... slashing_address ...
          )
          1.0 BTC
```

After lock confs, Bitlayer L2 mints staking voucher to Alice's L2 address.
Alice picks validator V_5 to delegate; V_5's vote weight increases by 1.

Yield accrues over time. Suppose 6 % APR -> 0.06 BTC equivalent over a
year, distributed in BTR; Alice claims and unstakes:

```
Bitcoin tx (after 1000-block unbond delay):
  Input: staking output (taken via leaf 1)
  Witness: <Alice_sig>
  Output: 1.0 BTC -> Alice's wallet
```

If Alice misbehaved (e.g. delegated to a validator that signed a
fraudulent state and operators publish proof), the slashing path
spends:

```
Bitcoin tx:
  Input: staking output (taken via leaf 2)
  Witness: <covenant_proof, validator_sig>
  Output: <slash_amount> -> burn or insurance fund
          <remainder>     -> Alice
```

## Common pitfalls

- **Covenant emulation**: Bitcoin lacks a real covenant primitive. The
  slashing path uses pre-signed transactions or a federation of
  covenant-emulator signers; security depends on signer set integrity.
- **Slash conditions**: must be precisely specified in protocol;
  ambiguous fraud-proof rules invite disputes.
- **Long unbond**: 1000 BTC blocks ≈ 7 days; users with sudden
  liquidity needs cannot exit instantly.
- **Validator collusion**: if validator set colludes with covenant
  emulators, they could falsely slash honest stakers. Use widely-
  distributed validators + multiple covenant emulator pools.
- **MEV on L2**: standard EVM L2 MEV applies; BTC stake doesn't
  protect against frontrunning at the rollup layer.

## References

- Bitlayer documentation (docs.bitlayer.org).
- Babylon staking spec (similar staking primitives).
- BitVM2 paper.
