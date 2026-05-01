# MAP Protocol Interop Architecture - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/map-protocol`.
> Canonical source: https://docs.maplabs.io/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/map-protocol/SKILL.md

## Concept

MAP Protocol (Multichain Asset Protocol) is a Bitcoin-anchored
**interoperability layer** that bridges Bitcoin assets (BTC, BRC-20,
Runes, Ordinals) to multiple EVM chains. It uses on-chain light clients
on each connected chain plus zk-SNARKs to verify Bitcoin block headers,
allowing trust-minimized cross-chain operations without a centralized
bridge.

## Walkthrough / mechanics

### Components

- **MAP Relay Chain**: an EVM-compatible chain that serves as the
  routing hub. Maintains light clients of every connected chain (Bitcoin,
  Ethereum, BNB, Polygon, etc.).
- **Light client contracts**: deployed on each connected chain, verifying
  block headers from MAP Relay (and vice versa).
- **Maintainers**: stake-bonded relayers submitting block headers and
  proofs across chains.
- **MOS (MAP Omnichain Service)**: standardized interface for
  cross-chain message passing.

### Bitcoin light client

A Solidity contract on the MAP Relay Chain implements:

1. Bitcoin header storage with PoW validation.
2. Difficulty adjustment per 2016-block epoch.
3. SPV proof verification for transactions.

Uniquely, MAP uses **zk-SNARKs** to compress block-header validation:
maintainers submit a SNARK proving "I added headers H1..Hn correctly
chained to H0 with valid PoW", reducing on-chain gas dramatically vs
header-by-header submission.

### Cross-chain flow

User wants to send BRC-20 ORDI from Bitcoin to BNB Chain.

1. User inscribes a BRC-20 transfer on Bitcoin to MAP's bridge address
   with embedded destination metadata.
2. MAP maintainer detects inscription, submits SPV + ZK proof to MAP
   Relay light client.
3. MAP Relay contract recognizes inscription, mints wrapped-ORDI on MAP
   Relay.
4. MAP Relay sends MOS message to BNB Chain via BNB-side light client
   verifying MAP Relay's state.
5. BNB-side wrapper contract mints user's wrapped-ORDI on BNB.

### MAP -> Bitcoin (peg-out)

Burning wrapped assets on a connected chain triggers an MOS message to
MAP Relay, which authorizes a Bitcoin signing federation (currently
multi-sig) to release BTC/BRC-20 inscription transfer to the user.
Bitcoin-side trust is currently federated; ZK-SP1 proof-of-reserve
mechanism is on the roadmap.

### Maintainer economics

- Maintainers stake MAP tokens.
- Earn fees per cross-chain message (gas surcharge + bridge fee).
- Slashed if they submit invalid proofs.
- Compete on speed (first valid proof wins reward).

## Worked example

A user bridges 0.1 BTC -> wMBC (wrapped MAP BTC) on Polygon:

```
1. User funds the federation BTC address with 0.1 BTC.
2. Maintainer M1 watches Bitcoin chain, batches new headers + the user's
   tx, generates a ZK proof of validity.
3. M1 calls MAP Relay LightClient.submitBatch(zk_proof, headers).
4. MAP Relay verifies proof; advances Bitcoin head; processes deposit
   event.
5. MAP Relay -> Polygon LightClient via MOS message.
6. Polygon contract mints 0.1 wMBC to user's Polygon address.
```

User then trades wMBC on Polygon DEXes; later burns to redeem BTC.

## Common pitfalls

- **Light client liveness**: each connected chain must run an updated
  light client; if no maintainer submits headers, messages stall.
- **Proof generation cost**: ZK header batching has hardware costs;
  small chains may struggle to fund continuous maintenance.
- **Federation custodial phase**: peg-out from MAP to Bitcoin still uses
  federation multisig; trust assumption higher than peg-in.
- **Cross-chain message delay**: typical 5-30 minutes depending on
  source/destination chain block times.
- **Maintainer misconduct**: malicious maintainer submitting fake proofs
  is slashed, but if a quorum colludes, lookups can be temporarily
  poisoned. Multiple maintainers + on-chain verification mitigates.

## References

- MAP Protocol whitepaper.
- MAP docs (docs.maplabs.io).
- BIP-152 SPV (compact block) for relay efficiency.
