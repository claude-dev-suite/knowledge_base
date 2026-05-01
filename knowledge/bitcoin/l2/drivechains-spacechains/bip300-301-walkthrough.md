# Drivechains BIP-300 / BIP-301 Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/drivechains-spacechains`.
> Canonical source: https://github.com/bitcoin/bips/blob/master/bip-0300.mediawiki
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/drivechains-spacechains/SKILL.md

## Concept

Drivechains (BIP-300 + BIP-301, Paul Sztorc) propose a soft-fork
mechanism that lets a Bitcoin sidechain ("drivechain") inherit Bitcoin's
hashrate via merge mining and let users move BTC into and out of the
sidechain via a Bitcoin-native two-way peg. The peg uses a
"hashrate vote" system: Bitcoin miners vote on whether sidechain
withdrawals are honest.

BIP-300 specifies the peg mechanism (sidechain registration + withdrawal
voting). BIP-301 specifies blind merge-mining (BMM), letting Bitcoin
miners commit to sidechain block hashes without running sidechain nodes.

## Walkthrough / mechanics

### Sidechain registration (BIP-300)

A new drivechain is created by:

1. A miner inserting a special **propose-sidechain** OP_RETURN message
   in their coinbase, naming the sidechain (slot number 0..255).
2. Other miners signal approval over a "Sidechain Activation Window"
   (~6 months of blocks).
3. After 90 % approval, the sidechain is **active**: a slot is reserved
   for it, and BTC can be deposited.

### Two-way peg

**Deposit (BTC -> drivechain)**:

1. User sends BTC to the **sidechain deposit address** (a script bound
   to the slot).
2. Sidechain miners observe the deposit, mint sBTC on the sidechain.

**Withdrawal (drivechain -> BTC)**:

1. User sends sBTC to a "withdraw" address on the sidechain.
2. Sidechain miners aggregate withdrawals into a **Withdrawal Bundle (WB)**.
3. The WB is proposed on Bitcoin via a special OP_RETURN.
4. Bitcoin miners vote over the next ~3 months (13 150 blocks): each
   block can contain an "ACK" or "NACK" for any pending WB. After
   ACK/NACK threshold (>= 13 150 ACK), the WB is approved and BTC is
   released according to its outputs.

### Why miner votes?

The drivechain proponent's argument is that Bitcoin miners have aligned
interests with Bitcoin holders (they earn BTC and don't want to debase
it). Their hashrate vote is a censorship/approval signal.

Critics argue that miners are not aligned (they could collude with
sidechain operators to steal funds via false-positive votes), and that
this introduces a new form of miner trust into Bitcoin.

### BIP-301 Blind Merge Mining

A miner can earn drivechain fees without running a drivechain node by:

1. Sidechain users / nodes pay miners to include a sidechain block hash
   in the coinbase.
2. Miners include the hash; if the underlying block is valid (per
   sidechain consensus), the fee is collected on the sidechain.

Bitcoin miners thus only see a fixed-size commitment, not the full
sidechain blocks. This mirrors AuxPoW (RSK / Namecoin) but is
"blind" (miner trusts the bidder rather than verifying).

### Withdrawal vote semantics

Each Bitcoin block votes 1 of 3:

- **ACK**: miner acknowledges this Withdrawal Bundle is honest.
- **NACK**: miner rejects this WB.
- **Abstain** (no opcode emitted).

Threshold tallies are compared to fixed ratios (e.g. 13 150 / 26 300).
Pending WBs that don't reach ACK quorum within 26 300 blocks are
**failed**.

## Worked example

A drivechain "BMM-coin" is registered at slot 0.

```
1. Alice deposits 1 BTC to slot 0 deposit address.
2. Sidechain (BMM-coin) credits Alice with 1 sBTC after detection.
3. Alice trades sBTC on sidechain DEX for 6 months.
4. Alice initiates withdrawal: 0.5 sBTC -> Bob's BTC address.
5. Sidechain miners aggregate 100 pending withdrawals into one WB.
6. WB is proposed on Bitcoin block height H.
7. Bitcoin miners vote across 13 150 blocks; suppose 14 000 ACK.
8. WB approved; BTC released according to its outputs at height H + 26 300
   (~6 months after proposal).
9. Bob receives 0.5 BTC.
```

## Common pitfalls

- **Soft fork status**: BIP-300/-301 are not activated on Bitcoin
  mainnet (as of 2026). Drivechains exist as testnet experiments only.
  Multiple Bitcoin Knots and parallel implementations support BIP-300
  but Bitcoin Core does not.
- **Miner-collusion theft**: a >= 51 % miner can ACK fraudulent WBs that
  steal sidechain funds. The 6-month voting window is a safety margin
  for users to monitor and complain, but ultimately security is miner
  trust.
- **Withdrawal latency**: ~6 months is the worst-case wait. Liquidity
  providers (LP) can offer instant exits at a premium.
- **Slot scarcity**: only 256 slots; BIP-300 limits how many drivechains
  can run simultaneously.
- **Bridge UX**: deposits are fast, withdrawals are slow; users must
  understand asymmetry.

## References

- BIP-300 (Hashrate-Escrow Voting).
- BIP-301 (Blind Merge Mining).
- Sztorc. "The Drivechain Proposition" 2017.
- Bitcoin Knots BIP-300 implementation.
