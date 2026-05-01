# BOB (Build-on-Bitcoin) OP Stack Hybrid Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/bob`.
> Canonical source: https://docs.gobob.xyz/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/bob/SKILL.md

## Concept

BOB ("Build on Bitcoin") is a hybrid L2 that lives on **Optimism's OP
Stack** (Ethereum L2 framework) while integrating Bitcoin-native
primitives. BOB runs as an OP rollup posting state to Ethereum L1 for
settlement and DA, but its smart contracts can verify Bitcoin SPV proofs,
read Ordinals/Runes, and trigger BTC transfers via a BitVM-aware bridge.

## Walkthrough / mechanics

### Why "hybrid"

Pure Bitcoin L2 architectures struggle with limited scripting expressiveness.
Pure Ethereum L2 lacks Bitcoin-native UX. BOB chooses Ethereum's OP-Stack
for execution stability (mature OP Stack + battle-tested Ethereum
settlement) but provides:

- BTC light client smart contracts (verifies Bitcoin headers via SPV).
- Ordinals/BRC-20 indexers integrated as precompiles.
- Native BTC bridge (via partners like Threshold tBTC + future BitVM2
  variants).

### Architecture

```
+-----------------------------------------+
| BOB Rollup (OP Stack, EVM, Geth fork)    |
+-----------------------------------------+
| Settlement: Ethereum L1 (Cancun/Dencun)  |
+-----------------------------------------+
| DA: Ethereum blob space (EIP-4844)       |
+-----------------------------------------+
| BTC light client (smart contracts)       |
+-----------------------------------------+
| BTC bridge: tBTC v2 + BitVM2 (planned)   |
+-----------------------------------------+
```

### Bitcoin light client

A Solidity contract on BOB stores Bitcoin headers. Anyone can submit a
new header that:

1. Has correct PoW (matches difficulty target).
2. Chains to a previously-stored header.
3. Difficulty adjustment per epoch is consistent.

Once stored, contracts can verify SPV proofs of Bitcoin transactions
(e.g. Ordinals inscriptions) by checking Merkle inclusion against a
stored header.

### Bridging

- **tBTC v2**: existing Threshold-network tBTC contract is deployed on
  BOB; users mint tBTC by depositing BTC to the Threshold federation,
  receive tBTC on BOB.
- **Native BOB BTC bridge** (in development): BitVM2-style operator
  bridge with fraud-proof challenge.

### OP-Stack settlement

BOB inherits Optimism's settlement model:
- Sequencer batches L2 txs, posts to L1 every ~2 minutes.
- 7-day fault-proof window before L2 -> L1 withdrawals finalize.
- Forced inclusion via L1 transaction (censorship resistance).

### Bitcoin-aware smart contract pattern

```solidity
function redeemBRC20(bytes calldata proof, uint256 blockHeight) external {
    require(BTCLightClient.verifySPV(proof, blockHeight), "bad SPV");
    OrdinalsParser.Inscription memory ins = OrdinalsParser.parse(proof);
    require(ins.protocol == "brc-20", "not BRC-20");
    // ... mint equivalent ERC-20 representation
}
```

## Worked example

A user wants to bring 100 ORDI (BRC-20) from Bitcoin to BOB.

```
1. User finds existing ORDI inscription tx on Bitcoin.
2. User submits SPV proof of inscription tx + Merkle path + block header
   to BOB's BRC-20 wrapper contract.
3. Contract verifies SPV via BTCLightClient.
4. Contract checks the inscription content equals "{p:'brc-20',op:'transfer',
   tick:'ORDI',amt:100}".
5. Contract mints 100 wrapped-ORDI ERC-20 to user.
```

Withdrawal of wBTC (tBTC) back to Bitcoin:

```
1. User burns tBTC on BOB.
2. User waits OP-Stack 7-day window for withdrawal finalization on Ethereum.
3. tBTC contract on Ethereum receives finalized state, queues redemption.
4. Threshold network signs Bitcoin payout tx; user receives BTC.
```

## Common pitfalls

- **OP-Stack 7-day delay**: users withdrawing from BOB to Ethereum (or
  through to BTC via tBTC) must wait 7 days unless they use third-party
  fast-withdrawal liquidity providers.
- **SPV proof gas cost**: validating a Bitcoin SPV proof on EVM is
  ~80k gas + Merkle verification gas; can be expensive at high gas
  prices.
- **Light client liveness**: someone must keep submitting Bitcoin
  headers; protocol incentivizes via small reward per accepted header.
- **Ordinals reorgs**: a Bitcoin reorg of 6 blocks could invalidate a
  recently-bridged inscription. Require deeper confs before accepting
  SPV.
- **Chain ID confusion**: BOB has its own chain ID; users sometimes
  confuse with Ethereum mainnet, sending funds to the wrong network.

## References

- BOB documentation (docs.gobob.xyz).
- OP Stack specs (specs.optimism.io).
- Threshold tBTC v2 documentation.
- BIP-141 (SegWit / Merkle proofs).
