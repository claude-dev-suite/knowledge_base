# RSK PowPeg Architecture - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/rootstock-rsk`.
> Canonical source: https://dev.rootstock.io/rsk/architecture/powpeg/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/rootstock-rsk/SKILL.md

## Concept

The PowPeg is the federated 2-way peg between Bitcoin and the Rootstock
(RSK) sidechain. It moves BTC into the RSK chain as **rBTC** (1:1) and
back. Custody is held by a federation of \~10–15 functionaries, each
running a tamper-evident **HSM** (PowHSM, custom-firmware Ledger Nano S
devices). PowHSMs sign only against on-chain proofs of Bitcoin and RSK
state, removing trust in functionary individuals.

## Walkthrough / mechanics

### Federation members

Each functionary runs:

- A full Bitcoin Core node.
- A full RSK node.
- A PowHSM device storing one key share of the federation Bitcoin
  multisig (P2SH-multisig: e.g. 6-of-9 historically, growing as
  members join).

### Bridge contract

On RSK, the **Bridge** is a precompiled contract at address
`0x0000...01000006`. It exposes:

- `getFederationAddress` — current peg-in BTC address.
- `registerBtcTransaction(tx, height, partialMerkleTree)` — provides an
  SPV proof of a peg-in tx. Bridge verifies block header + Merkle, then
  credits the recipient with rBTC.
- `releaseBtc(amount)` — peg-out request burning rBTC.
- `addSignature(pubkey, sigs, txHash)` — functionaries submit BTC
  multisig signatures for outbound peg-out txs.

### PowHSM signing rule

A PowHSM will **only** sign a peg-out BTC tx if:

1. The transaction is referenced from a confirmed RSK block.
2. The RSK block has accumulated enough cumulative work (typical
   threshold: 4000 RSK blocks ≈ 1.5 days).
3. Cumulative work on RSK exceeds a stored "watermark"; the watermark
   only moves forward, preventing rollback attacks.

The HSM verifies all this internally; if rules fail, no signature emits.

### Merge mining

RSK is **merge-mined** with Bitcoin: Bitcoin miners include an RSK block
hash in their coinbase OP_RETURN (or a dedicated tag) and submit the
Bitcoin block header as a proof of work for the RSK block. This means
~75 % of Bitcoin hashrate currently secures RSK at zero marginal
cost to miners.

### Federation rotation

New federation members join via an RSK governance contract election. Old
multisig is **migrated** with a single Bitcoin tx that spends the existing
peg UTXOs to the new federation script. Members publish proof-of-key
ceremonies before rotation.

## Worked example

Alice deposits 1 BTC for rBTC.

```
Bitcoin tx:
  Input : Alice UTXO 1.001
  Output: P2SH-multisig (federation address)  1 BTC
  Output: Alice change 0.0009 BTC
```

After 100 BTC confs, Alice (or anyone) calls
`Bridge.registerBtcTransaction(serialised_tx, height, MMR_proof)` on RSK.
Bridge verifies SPV, then mints 1 rBTC to her RSK address.

Peg-out: Alice calls `Bridge.releaseBtc()` from her RSK contract sending
0.5 rBTC. Bridge:

1. Burns 0.5 rBTC.
2. Stores a release request in chain state.
3. Functionaries' HSMs gradually contribute signatures across following
   RSK blocks.
4. After 6-of-9 sigs collected, Bridge serialises the BTC tx and
   broadcasts via federation members' BTC nodes.

## Common pitfalls

- **Peg-out latency**: 6-of-9 collection can take several hours. Wallets
  showing "pending" must surface this clearly to users.
- **HSM availability**: a single firmware bug can make functionary HSMs
  refuse to sign. PowHSM is open-source and audited; updates are
  governance-controlled.
- **SPV proof gas**: `registerBtcTransaction` is expensive (megaGas).
  Batching multiple deposits in one tx reduces per-deposit gas.
- **51 %-attacker sidechain reorg**: because RSK is merge-mined, a
  Bitcoin majority miner could simultaneously reorg both chains. PowHSM
  watermarks mitigate by refusing to sign on shallow chains.
- **Federation collusion**: 6-of-9 (or current threshold) can in theory
  steal peg balance. Mitigation: HSMs enforce the rules; key extraction
  requires a successful supply-chain attack on the firmware.

## References

- RSK PowPeg whitepaper (2020).
- powhsm.io firmware docs.
- rsk-architecture/powpeg-spec on dev.rootstock.io.
