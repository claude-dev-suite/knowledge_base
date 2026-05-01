# Hemi Network Superfinality Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/hemi`.
> Canonical source: https://docs.hemi.xyz/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/hemi/SKILL.md

## Concept

Hemi is a Bitcoin/Ethereum hybrid network designed by the Maxwell-Garzik
team (former Bitcoin Core / Bloq leadership). Its central technical claim
is **superfinality**: Hemi blocks reach a finality stronger than either
Bitcoin or Ethereum alone, by anchoring to **both** chains. Each Hemi
block hash is committed to a Bitcoin block (via PoP - Proof-of-Proof) and
referenced via Ethereum settlement, requiring a simultaneous reorg of
both base layers to revert Hemi.

## Walkthrough / mechanics

### Layered consensus

```
+-------------------------------+
| Hemi VM (hVM): EVM + Bitcoin   |
+-------------------------------+
| Hemi sequencer + PoS validators |
+-------------------------------+
| PoP (Proof-of-Proof) miners     |
| (anchor Hemi -> Bitcoin)        |
+-------------------------------+
| Bitcoin DA + finality           |
+-------------------------------+
```

### hVM

The Hemi Virtual Machine is an **EVM with embedded Bitcoin node** (a
real `bitcoind` subprocess). hVM smart contracts can:

- Read Bitcoin block headers, transactions, UTXOs, mempool.
- Read Ordinals / Runes / BRC-20 indexer state.
- Issue queries against the Bitcoin chain like SPV but without SPV
  proofs (the EVM has direct read access).

This makes Hemi unusual: contracts trust the **node** rather than
verifying proofs. Hemi argues this is acceptable because each validator
runs the same Bitcoin client and consensus over hVM state implies
agreement on Bitcoin reads.

### PoP (Proof-of-Proof)

PoP miners take Hemi block hash and inscribe it onto Bitcoin via:

1. Construct a Bitcoin transaction with an OP_RETURN or Taproot
   inscription containing the Hemi block hash.
2. Pay Bitcoin fee.
3. After Bitcoin confirmation, Hemi consensus considers the block to
   have reached "Bitcoin finality" — a reorg now requires a Bitcoin
   reorg of equal depth.

PoP miners are rewarded by Hemi (in HEMI native token) for each
successful anchor.

### Superfinality

After ~9 Bitcoin confirmations of the PoP commitment, Hemi blocks are
considered economically irreversible: an attacker would need to reorg
Bitcoin AND Hemi simultaneously, a strictly stronger requirement than
attacking either chain alone.

### EVM compatibility

Standard Geth-fork client. Solidity dApps deploy unchanged. Native gas
token: HEMI. Fee market includes a "PoP gas surcharge" that pays PoP
miners.

### Bridging

- **hBTC**: 1:1 BTC representation on Hemi, secured by a hybrid
  multisig + tunneling protocol (currently federated; BitVM2 path on
  roadmap).
- **hETH**: 1:1 ETH representation via Ethereum settlement bridge.

## Worked example

A Hemi smart contract checking Bitcoin balance:

```solidity
function getBitcoinBalance(bytes20 btcAddrHash) external view returns (uint64) {
    // hVM precompile: direct read from local bitcoind
    return BitcoinPrecompile.getAddressBalance(btcAddrHash);
}
```

A user transfers 1 hBTC to another Hemi address. The block containing
this tx is at Hemi height 50 000.

```
PoP miner anchors block 50 000:
  Bitcoin tx with OP_RETURN <hemi_block_hash_50000>
  After 9 BTC confs, hemi block 50 000 finalized.
```

To reverse this transfer, an attacker must:
- Reorg Hemi to before 50 000 (needs majority Hemi PoS stake).
- Reorg Bitcoin past the PoP anchor (needs 51 % Bitcoin hashrate for 9
  blocks).

## Common pitfalls

- **Validator trust on Bitcoin reads**: hVM precompiles trust local
  bitcoind. A validator running a forked bitcoind could disagree with
  others; consensus failures appear as forks. Mitigated by requiring
  all validators run reference bitcoind versions.
- **PoP miner liveness**: if no PoP miner anchors a block, it never
  gets superfinality. Multiple PoP miners + economic incentives reduce
  this risk.
- **PoP gas spam**: low-value Hemi blocks anchoring cheaply increases
  Bitcoin OP_RETURN bloat.
- **Bridge custodial risk**: hBTC bridge is currently federated; full
  trust-minimisation on roadmap.
- **Reorg propagation**: if Hemi reorgs but PoP anchor stays in Bitcoin,
  the orphaned block's PoP is wasted; protocol does not pay rewards on
  orphans.

## References

- Hemi documentation (docs.hemi.xyz).
- Veriblock PoP protocol (predecessor; Maxwell, Garzik 2018).
- Garzik, Maxwell. "Hemi: Bitcoin-secured smart contracts" 2024.
