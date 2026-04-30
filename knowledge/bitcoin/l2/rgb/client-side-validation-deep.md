# RGB Client-Side Validation - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/rgb`.
> Canonical source: https://docs.rgb.info/ and RGB Working Group specs
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/rgb/SKILL.md

## Concept

RGB's foundational property is **client-side validation (CSV)** -- contract state lives
in user wallets, not on the Bitcoin chain. The chain is used only for *single-use seals*
(SUSes): each Bitcoin output represents a one-shot commitment slot. Spending the output
"breaks the seal" and reveals the next state in a contract via a signed off-chain
message that any verifier can cross-check against the on-chain spend. This is the
opposite of EVM-style "world-replicates state" and gives RGB extreme privacy + scaling
at the cost of out-of-band proof distribution.

## Walkthrough / mechanics

### Single-use seal model

```
Bitcoin UTXO  =  one slot for one future commitment
Spend the UTXO  =  reveal the contracted-to commitment
```

The Bitcoin output is a *seal*: any honest verifier sees that this output, when spent,
revealed exactly one off-chain message (the new state).

In RGB, the seal is a **Tapret** commitment (Taproot-tweaked output) where the tweak
encodes:

```
TaprootKey = TweakedKey(InternalKey, OP_RETURN-equivalent script committing to
                        H(state_transition_witness))
```

Tapret keeps the on-chain footprint a normal-looking Taproot output. The tweak commits
to a 32-byte hash; the actual transition data lives off-chain.

### State transitions

A contract state transition is a Strict-encoded data structure:

```
StateTransition {
  prev_outputs:  [seal_a, seal_b, ...],      // inputs being spent
  assignments:   [
    AssignmentTo(new_seal, new_state_data),
    ...
  ],
  globals:       updated contract globals,
  redeem_proof:  AluVM execution proof or signature
}
```

The transition is a JSON-CBOR blob hashed to commit-32 bytes that gets put into the
Tapret tweak of the spending tx.

### Validation walk

When Alice receives a token, her wallet performs:

```
For each candidate input UTXO chain back to genesis:
  1. Fetch the on-chain tx that produced it.
  2. Verify the Tapret commitment matches the off-chain transition's hash.
  3. Verify the transition's AluVM script ran successfully.
  4. Verify state conservation (no inflation in fungible class).
  5. Recurse on each input of the transition.
At genesis: verify the issuer signature and contract schema.
```

Validation can fail at any step; failure means the asset is not authentic from the
sender's perspective.

### Proof distribution

Since proofs aren't on-chain, RGB defines:
- **Bifrost**: a peer-to-peer proof exchange protocol.
- **Storm**: a content-addressed proof storage layer (DHT-like).
- **Universe**-style indexers (similar concept to TAP universes but RGB-specific).

The receiver of a payment must obtain the full proof chain from genesis to current
state. Sender hands it over directly; if lost, the receiver can attempt to fetch from
public RGB universe / Storm nodes.

## Worked example

Issuance and transfer of an RGB20 fungible asset:

```
Genesis (Tether issues USDT-RGB on a Bitcoin tx):
  contract_id = hash(genesis_data || schema_hash)
  schema      = RGB20 v1
  ticker      = USDT-RGB
  total_issue = 1,000,000,000
  issuance seal = Tapret commitment in genesis_tx output 0
  Issuer signs genesis with their private key.

Transfer 1: Tether -> Alice 1,000 USDT-RGB
  Tether spends genesis_tx:0.
  StateTransition_1 {
    prev: [genesis_tx:0]
    assignments: [
      Alice_seal:  Tapret commitment in transfer1_tx:0  (1,000 tokens)
      Tether_seal: Tapret commitment in transfer1_tx:1  (999,999,000 tokens)
    ]
    redeem_proof: Tether's signature
  }
  transfer1_tx is broadcast on Bitcoin with Tapret tweak = H(StateTransition_1).
  Tether sends StateTransition_1 + proof bundle to Alice off-chain.

Transfer 2: Alice -> Bob 100 USDT-RGB
  Alice spends transfer1_tx:0.
  StateTransition_2 {
    prev: [transfer1_tx:0]
    assignments: [
      Bob_seal:   transfer2_tx:0  (100 tokens)
      Alice_seal: transfer2_tx:1  (900 tokens)
    ]
    redeem_proof: Alice's signature
  }
  Alice sends Bob the proof bundle (genesis -> transition_1 -> transition_2).

Bob's validation:
  - Fetches transfer2_tx, transfer1_tx, genesis_tx.
  - Verifies each Tapret tweak matches the off-chain transition's hash.
  - Verifies issuer sig at genesis (against known Tether key from registry).
  - Verifies amount conservation: 1B -> {1k, 999_999k} -> {100, 900}+rest.
  - Accepts: Bob owns 100 USDT-RGB.
```

## Trade-offs and security

- **Proof loss = funds loss**: RGB has no on-chain history of asset state. A wallet
  that loses its proof bundle can never spend the asset again, even if the seed phrase
  is intact. Storm and Bifrost mitigate but don't eliminate.
- **Privacy max**: chain observers see only Tapret outputs (indistinguishable from any
  Taproot UTXO). Asset id, amount, recipient -- all hidden.
- **Sender must reveal full history**: the receiver, by virtue of validation, learns
  *everything* about the asset's prior owners back to genesis. This is "private from
  third parties, fully transparent to the recipient" -- different model than EVM
  ZK-roll-ups.
- **Storage burden**: long proof chains (many transfers) accumulate. RGB v0.11+ adds
  proof aggregation / pruning via "round-trips" with the issuer.
- **AluVM bugs**: contract logic runs client-side. A buggy schema is dangerous --
  attackers can submit transitions that pass on a buggy validator but fail on others,
  causing wallet inconsistency.
- **Compared to TAP's MS-SMT**: TAP has slightly less privacy (anchor outputs leak
  asset_id metadata via universe queries) but simpler validation (sum-tree balance
  proofs vs full transition replay).

## References

- RGB-WG GitHub - https://github.com/RGB-WG
- RGB protocol docs - https://docs.rgb.info/
- "RGB Black Paper" - Maxim Orlovsky
- LNP/BP Standards - https://www.lnp-bp.org/
