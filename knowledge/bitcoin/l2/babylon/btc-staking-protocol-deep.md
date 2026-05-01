# Babylon BTC Staking Protocol Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/babylon`.
> Canonical source: https://docs.babylonchain.io/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/babylon/SKILL.md

## Concept

Babylon's BTC staking protocol lets BTC holders **stake BTC to provide
security** to PoS chains without bridging BTC anywhere. The BTC stays on
Bitcoin in a self-custodial Taproot output; if the staker double-signs on
the destination PoS chain, a slashing transaction takes a portion of their
BTC. The protocol simulates a "covenant" via pre-signed transactions and
a covenant emulator quorum.

This is sometimes called **trustless staking** because the staker never
hands BTC to a custodian.

## Walkthrough / mechanics

### Staking output script

A BTC staker locks `V` BTC into a Taproot output with three script paths:

1. **Unbonding path**: relative timelock `T_u` -> staker can take their
   BTC back after `T_u` blocks (e.g. 10 000 BTC blocks ≈ 70 days).
2. **Slashing path**: spendable by covenant quorum + finality provider on
   provision of slashing evidence.
3. **Withdrawal path**: spendable by staker after `T_w` (longer than
   `T_u`) for refund without unbonding tx.

Schematic:

```
P2TR(
  internal_key = MuSig2(staker_pk, finality_provider_pk),
  scripts:
    leaf 1 (timelock_unbond): T_u OP_CSV staker_pk OP_CHECKSIG
    leaf 2 (slashing): SlashingScript(covenant_quorum, finality_provider_pk)
    leaf 3 (withdraw_after_max): T_w OP_CSV staker_pk OP_CHECKSIG
)
```

### Slashing path detail

The slashing leaf is parameterised so that the **only valid spend** sends
`fraction * V` to a fixed `slashing_address` (burn or insurance fund) and
the rest to the staker. This means a slashing tx cannot redirect BTC to
the slasher's pocket; only the protocol-defined burn/insurance address
receives.

### Covenant emulator

Bitcoin lacks a real covenant opcode. Babylon uses a **covenant emulator
quorum** (a pre-set group of signers) to enforce that the slashing tx is
correctly formed. They jointly hold a `covenant_quorum` MuSig2 key; their
signature is required only for the slashing path. The emulator keys are
pre-published; their signature is non-discretionary (only sign valid
slash txs).

To prevent emulator collusion against honest stakers, the slashing leaf
script also verifies a **finality provider signature** on the slashing
tx — meaning the finality provider must be slashable for the slash to go
through. This cryptographically binds the slash to a misbehaving party.

### Finality providers

Finality providers are PoS validators on the consumer chain (e.g.
Cosmos Hub, Babylon-secured rollup) who run an additional signing duty:
they sign each PoS block they vote for, using a key derived from the
secret embedded in the staking output.

If they double-sign two conflicting blocks at the same height, the two
signatures expose their secret -> anyone can extract `sk_finality_provider`
and use it on the slashing path to take their BTC stake.

This is **EOTS (Extractable One-Time Signatures)**, similar to Schnorr
nonce reuse: any honest party can extract the secret if FP misbehaves.

### Lifecycle

1. Staker creates Taproot staking output funded with `V` BTC.
2. Pre-signs unbonding tx + slashing tx + withdraw tx; sends to Babylon
   chain for indexing.
3. Babylon registers stake; staker delegates voting power to FP_X.
4. FP_X signs PoS blocks using EOTS.
5. If FP_X equivocates: anyone extracts secret, builds slashing tx
   (with covenant quorum sigs), submits to Bitcoin.
6. If staker wants to exit: broadcasts pre-signed unbonding tx; after
   `T_u`, BTC returns.

## Worked example

Alice stakes 1 BTC to FP "Jacqueline" on Babylon-secured Cosmos chain.

```
Bitcoin tx: 1 BTC -> P2TR staking output (above script).
Babylon registers: Alice's stake, delegated to Jacqueline.
```

Six months later, Jacqueline double-signs at height 1234 to attempt fork:

```
1. Watcher detects two BLS sigs on height 1234 from FP Jacqueline.
2. Watcher extracts FP secret via EOTS.
3. Watcher constructs slashing tx:
     Input: Alice's staking output (script-path leaf 2)
     Witness: covenant_quorum_sig + FP_secret + slashing_script
     Output 0: 0.05 BTC -> burn address (5% slash)
     Output 1: 0.95 BTC -> Alice
4. Watcher submits to Bitcoin.
```

Alice loses 5 % of her stake but keeps the rest — she did not misbehave;
her FP did, and consequently her stake was slashed proportionally.

## Common pitfalls

- **Pre-signed transactions** must cover all valid script paths; missing
  a pre-signed tx means user is locked.
- **Covenant emulator centralization**: covenant quorum is finite (e.g.
  9-of-15). Compromise enables fake slashes.
- **EOTS implementation correctness**: any nonce reuse outside the
  protocol leaks the FP key; FPs must use deterministic nonce schemes.
- **Front-running slash submission**: the watcher who first submits a
  valid slash earns a small reward. Multiple watchers may race.
- **Reorg of slash tx**: 6 BTC confs before considering slash final.

## References

- Babylon documentation (docs.babylonchain.io).
- "Bitcoin-secured PoS" — Babylon team papers 2023.
- BIP340/BIP341 (Schnorr / Taproot scripts).
- EOTS construction (research note 2022).
