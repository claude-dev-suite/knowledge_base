# RGB Lightning Flow - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/rgb`.
> Canonical source: https://github.com/RGB-Tools/rgb-lightning-node and Bitlight Labs docs
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/rgb/SKILL.md

## Concept

`rgb-lightning-node` (RLN) is an LDK fork that carries RGB asset state through Lightning
channels. Each commitment transaction includes an extra anchor output committing to the
post-update RGB state via Tapret. Asset transfers route via standard HTLCs whose payment
amounts are interpreted as asset units rather than (or in addition to) sats. As of March
2026, RLN is in production beta with USDT-on-RGB rolling out alongside Bitlight Labs's
testing infrastructure.

## Walkthrough / mechanics

### Channel funding with assets

```
Funding tx (multisig 2-of-2 between Alice, Bob):
  IN:  Alice's RGB UTXO (1000 USDT-RGB)
       Bob's BTC UTXO (channel reserve sats)
  OUT: 2-of-2 multisig output  (channel anchor)
        with Tapret commitment to initial RGB state:
        { Alice_balance: 1000 USDT, Bob_balance: 0 USDT }
```

The funding tx itself is an RGB transition. State data goes off-chain; on chain we see
a Taproot output that carries the Tapret tweak.

### Commitment txs

Each channel update produces a new asymmetric commitment tx (one per party). Compared to
vanilla LDK:

- Output 1: to_local (claimable by holder after timelock).
- Output 2: to_remote (immediate to other party).
- **Output 3 (RGB anchor)**: zero-sat dust output with Tapret tweak committing to the
  new RGB state assignment.
- HTLCs: standard, but the HTLC value carries asset units in the *RGB transition state*
  (not in the on-chain sat amount, which stays minimal).

### HTLC for asset transfer

```
Alice -> Bob HTLC for 50 USDT:
  channel_state.alice_usdt -= 50
  channel_state.bob_usdt   += 50  (pending HTLC)
  Anchor commitment encodes the new state.
  payment_hash = H, preimage = R as in vanilla LN.
```

When R is revealed:
- Both parties update commitment to a new state with the HTLC settled.
- Asset balance moves Alice -> Bob.

If the HTLC times out:
- Both parties update to a state that returns the asset to Alice.

### Multi-hop with RGB

Currently more constrained than TAP. RLN supports:
- Same-asset routing through nodes that all hold the same RGB schema.
- Atomic swap-style cross-asset: less mature than TAP's RFQ.

The expected production path (March 2026 update) involves Bitlight Labs and other
infrastructure-providers running RGB-aware liquidity hubs.

### On-chain close

Cooperative close:
- Final state is committed to a closing tx.
- Tapret tweak in the closing tx output(s) reflects final asset assignments.
- Both parties get on-chain UTXOs carrying their post-channel asset balance.

Unilateral close:
- The closing party broadcasts their commitment tx (with the RGB anchor output).
- Funds are revocable / claimable per LDK rules; assets follow the to_local / to_remote
  output's seal.

## Worked example

Alice and Bob open a channel; Alice loads 1000 USDT-RGB; Alice pays Bob 60 USDT in two
HTLCs:

```
Day 0  Funding tx:
       IN   Alice USDT UTXO (1000 USDT) + Alice 100k sat funding
       OUT  Channel multisig, value 100k sats, Tapret = H(state_v0)
            state_v0 = { alice: 1000 USDT, bob: 0 USDT }

Day 1  HTLC #1: Alice -> Bob 40 USDT
       Pending state: { alice: 960, bob: 0, htlc: 40@H1 }
       Bob settles via R1.  New state: { alice: 960, bob: 40 }.
       Both sides exchange revocation keys for prior state.

Day 2  HTLC #2: Alice -> Bob 20 USDT
       Pending: { alice: 940, bob: 40, htlc: 20@H2 }
       Bob settles. State: { alice: 940, bob: 60 }.

Day 30 Cooperative close.
       closing_tx outputs:
         OUT 1: 95k sat to Alice (sat reserve - fees)
         OUT 2: 0-sat-dust Tapret commit to alice_seal { 940 USDT }
         OUT 3: 0-sat-dust Tapret commit to bob_seal   { 60  USDT }
       Each party now has an on-chain RGB UTXO carrying their share.
```

## Trade-offs and security

- **Proof bundle size**: every channel update advances the RGB state graph. Closing
  parties must hand each other (and any future receiver) the full transition proof
  chain since channel open. Bundles can grow in long-lived channels.
- **State desync risk**: if RGB state and LDK state get out of sync (e.g., due to a
  buggy persist), a unilateral close may attempt to publish a state-prev commitment
  which the counterparty can punish. Critical to atomically persist both layers.
- **Watchtower coverage**: LDK watchtowers must be RGB-aware to detect equivocation in
  the RGB state. Adoption is still early.
- **Liquidity fragmentation**: each asset/schema combo is a separate liquidity surface.
  Cross-asset routing inside RGB is harder than in TAP today.
- **Wallet dependency on Bitlight infra**: in March 2026, RLN production deployments
  largely rely on Bitlight Labs's testing nodes. More implementations needed for
  decentralisation.
- **Compared to TAP/LND**: RLN is more flexible (full AluVM contracts) but less
  battle-tested; TAP is locked to fungible balance tracking but more mature.

## References

- rgb-lightning-node - https://github.com/RGB-Tools/rgb-lightning-node
- Bitlight Labs blog - https://bitlightlabs.com/blog
- RGB v0.11 release notes (March 2026) - https://github.com/RGB-WG/rgb/releases
- LDK - https://github.com/lightningdevkit/rust-lightning
