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
  partner custodians). Roadmap: the "Fraud Proofs Based on Bitcoin"
  module, still described in future tense in Merlin's own docs as of
  September 2026.

### Bridge model

Phase 1 (mainnet launch):

- 8-of-15 (varies) MuSig2 federation across Cobo, Sinohope, and
  partner custody firms.
- BTC, BRC-20, and Ordinals can all be wrapped to ERC-20/ERC-721
  representations on Merlin.
- Withdrawals require federation co-signing; typically processed in
  batches every BTC block.

Phase 2 (TBD):

- Bitcoin-verified fraud proofs replacing federation trust.
- ZK validity proofs anchored to Bitcoin via Taproot inscriptions.

Merlin names this module "Fraud Proofs Based on Bitcoin" and still
writes it in future tense ("will introduce") as of September 2026.
Its documented shape: prover and verifier pre-sign a series of
transactions to enable challenge-response; the program under
verification is compiled to a binary circuit of NAND gates; each
gate's leaf script goes into a Merkle tree whose root is committed to
a Taproot address; the verifier reads the root back and, on suspicion,
challenges the prover to produce the corresponding leaf scripts. The
construction is BitVM-shaped, but no Merlin primary source uses the
names BitVM or BitVM2 - do not attribute those to the project.

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

This drove the Merlin's Seal bridge vault to a peak of ~$2.65B of
locked BTC on 29 March 2024 (DefiLlama), ranking Merlin among top L2s
by deposit volume at the time.

The program did not hold. As of 15 September 2026 (DefiLlama):

| Metric | Value | Peak |
|---|---|---|
| Merlin's Seal vault (BTC locked) | ~$410M | ~$2.65B (29 Mar 2024) |
| Merlin chain DeFi TVL | ~$9.3M | ~$529M (5 May 2024) |

Note the ~40x gap between the two. These are separate measurements:
bridge-locked BTC is not value deployed in Merlin DeFi, and only a
small fraction of the bridged value shows up in the chain's own
protocols. On the same date Stacks (~$79.3M) and Rootstock (~$79.0M)
each carried roughly 8x Merlin's chain DeFi TVL, so the "largest
Bitcoin L2 by TVL" framing is a 2024 statement, not a current one.
The drawdown is sector-wide: Bitlayer fell from ~$361M (1 Jan 2025)
to ~$0.46M and BSquared from ~$89M to ~$2.6M over the same window.

### EVM compatibility

Standard tooling works: Foundry, Hardhat, viem, Metamask. Chain ID
4200 (Merlin Mainnet, confirmed via `eth_chainId` on 15 September
2026). Block time \~3 s by design; Blockscout reported a \~4.05 s
trailing average on 15 September 2026. Gas paid in BTC (1 sat = 10^10
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

- **Federation custody risk**: until the Bitcoin fraud-proof module
  ships, withdrawals require federation honesty + liveness. ~$410M
  locked in the vault on 15 September 2026 (down from ~$2.65B at the
  March 2024 peak) is still an extremely high reward for an attacker
  if the signer set is compromised — the drawdown shrank the
  prize, it did not remove it.
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
- Fraud Proofs Based on Bitcoin - https://docs.merlinchain.io/merlin-docs/about-merlin/key-modules/fraud-proofs-based-on-bitcoin
- Polygon CDK technical paper.
- Cobo MPC custody whitepaper.
