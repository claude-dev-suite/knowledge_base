# Merlin Chain TVL Architecture - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/merlin`.
> Canonical source: https://docs.merlinchain.io/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/merlin/SKILL.md

## Concept

Merlin Chain is a Bitcoin L2 built on **Polygon CDK** (zk-rollup
construction toolkit). It targets BRC-20, ordinals, and inscriptions
ecosystems by providing an EVM-compatible execution layer where BTC,
BRC-20s, and Bitcoin-native NFTs can interact via a federated bridge.
Its competitive edge has been TVL via BTC-yield products (locked-LP
strategies + staking integrations).

## Walkthrough / mechanics

### Stack

- **Execution**: EVM-equivalent built on Polygon CDK Erigon fork.
- **Sequencer**: federated, with planned decentralisation.
- **Prover**: zk-SNARK using Polygon CDK's prover (Plonky2-style).
- **DA**: hybrid — primary on Celestia, with optional Bitcoin
  inscription anchoring.
- **Settlement**: Bitcoin via a federated multisig bridge (Cobo MPC +
  partner custodians). Roadmap: BitVM2 trust-minimisation.

### Bridge model

Phase 1 (mainnet launch):

- 8-of-15 (varies) MuSig2 federation across Cobo, Sinohope, and
  partner custody firms.
- BTC, BRC-20, and Ordinals can all be wrapped to ERC-20/ERC-721
  representations on Merlin.
- Withdrawals require federation co-signing; typically processed in
  batches every BTC block.

Phase 2 (TBD):

- BitVM2 fraud-proof bridge replacing federation trust.
- ZK validity proofs anchored to Bitcoin via Taproot inscriptions.

### Asset support

- **m-BTC**: ERC-20 1:1 BTC peg.
- **BRC-20 wrappers**: each BRC-20 ticker minted on Merlin as an
  ERC-20 controlled by a wrapper contract.
- **Ordinals**: ERC-721 with metadata pointing to inscription content
  hash and block index.
- **Runes**: similar ERC-20 wrap for rune balances.

### Yield mechanism

Merlin pioneered **"BTC2"**: a public liquidity-mining program where
users lock BTC into the bridge, then deploy m-BTC across DEX LPs and
yield protocols. Rewards combine:

- Trading fees.
- MERL token emissions.
- Partner protocol incentive layers.

This drove TVL > $3B at peak (early 2024), ranking Merlin among top L2s
by deposit volume.

### EVM compatibility

Standard tooling works: Foundry, Hardhat, viem, Metamask. Chain ID
4200 (Merlin Mainnet). Block time \~3 s. Gas paid in BTC (1 sat = 10^10
gas-wei equivalent), simplifying UX vs. ETH-fee chains.

## Worked example

Bridge BTC to Merlin and yield-farm:

```
1. User sends 0.5 BTC to bridge address (federation P2WSH).
2. Federation indexers detect deposit at confs >= 6.
3. Federation signs L2 mint tx, depositing 0.5 m-BTC to user's EVM addr.
4. User adds 0.5 m-BTC + 25 000 USDT to a Merlin DEX LP.
5. User stakes LP token in protocol vault.
6. Vault aggregates trading fees + MERL incentives + sub-yield from
   collateral lent to a Bitcoin-secured stable-asset protocol.
```

Withdraw:

```
1. User unstakes LP, removes from DEX, swaps back to m-BTC.
2. User burns m-BTC via bridge contract.
3. Federation observes burn, includes user's withdrawal in next batch
   payout tx on Bitcoin.
4. After ~30 min, user receives BTC at their wallet.
```

## Common pitfalls

- **Federation custody risk**: until BitVM2 migration, withdrawals
  require federation honesty + liveness. ~$3B TVL = extremely high
  reward for attacker if signer set is compromised.
- **BRC-20 wrapping correctness**: BRC-20 inscription accounting
  depends on indexer software; bridge must use consensus indexer.
- **MERL token volatility**: yield denominated in MERL is highly
  volatile; users frequently exit yield programs at top, leaving
  later participants with lower returns.
- **DA centralisation**: relying on Celestia for DA + federation for
  validity means Merlin is closer to a sidechain than a true rollup
  in current phase.
- **Bridge UX delays**: peak times see 1-2 hour withdrawal delays
  because federation batches are throttled.

## References

- Merlin Chain documentation (docs.merlinchain.io).
- Polygon CDK technical paper.
- Cobo MPC custody whitepaper.
