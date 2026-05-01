# Pool Auction Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/loop-pool-lit`.
> Canonical source: https://docs.lightning.engineering/lightning-network-tools/pool
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/loop-pool-lit/SKILL.md

## Concept

Lightning Labs **Pool** is a non-custodial marketplace for Lightning
channel liquidity. Sellers (with idle capital) lease channels to
buyers (with traffic but no inbound liquidity) via on-chain auction.
The auction settles every ~10 minutes via the Pool **auctioneer** (an
LL-run server) into a **batch transaction** that opens many channels
at once, amortising fees.

The protocol uses pre-signed transactions and **dual-funded** opens:
sellers commit liquidity that can only be paid back by ending the
lease.

## Walkthrough / mechanics

### Roles

- **Auctioneer**: runs the order book, computes auction matches,
  builds the batch tx.
- **Maker (seller)**: posts ask orders ("I'll lease X sat for N blocks
  at premium Y").
- **Taker (buyer)**: posts bid orders ("I'll pay premium Z for X sat
  inbound").

### Account setup

A user opens a Pool **account** by sending BTC to an on-chain output
controlled by `(user_key, auctioneer_key)` 2-of-2 multisig. This
account holds the user's reserve and earns/pays premiums.

### Order book

Orders specify:
- Direction: bid (buy inbound) or ask (sell outbound).
- Amount (sat).
- Lease duration (blocks).
- Premium rate (sat per block per million sat).
- Min/max channel size.
- Node info (to ensure peers can connect).

### Auction matching

Every batch interval (~10 min), auctioneer:

1. Sorts asks ascending by premium, bids descending.
2. Matches at the **uniform clearing price**: lowest accepted ask =
   highest accepted bid for marginal unit.
3. Generates pairs (maker, taker, amount, premium).

### Batch transaction

Auctioneer builds a single Bitcoin tx containing:

- Inputs: each maker's account UTXO + bidder's account UTXO.
- Outputs: per-pair lease channel opens (P2TR Taproot multisig
  channel funding output).
- Account residuals: each user's remaining balance back to their
  account.

All sigs are exchanged via the auctioneer (relay-only); the user's
key + auctioneer's key co-sign each input.

### Lease enforcement

The lease channel has a special **maker-key timelock**: maker cannot
cooperatively close the channel until the lease duration expires.
During the lease, maker's funds are locked in the channel; taker
benefits from the inbound liquidity.

After lease expires, channel can close normally. Maker reclaims their
funds + premium; taker keeps any earnings from routing or payments.

### Premium calculation

```
premium = amount * rate_per_block_per_msat * lease_blocks
```

Maker earns the premium on lease open (pre-paid by taker). Some
versions defer payment to lease end.

## Worked example

Maker posts ask: 0.5 BTC outbound liquidity, 4032 blocks (~28 days),
rate 0.000005 sat/block/sat. That's premium = 50,000,000 * 0.000005
* 4032 = 1,008,000 sat ≈ 0.01 BTC.

Taker posts bid: 0.5 BTC inbound, 4032 blocks, willing to pay up to
0.01 BTC.

Auctioneer matches at clearing price (suppose 0.01 BTC).

Batch tx:

```
Inputs:
  Maker_account_UTXO 0.6 BTC (key2of2)
  Taker_account_UTXO 0.05 BTC

Outputs:
  Channel funding: 0.5 BTC (Maker_key + Taker_key Taproot)
  Maker account residual: 0.6 + 0.01 (premium) - 0.5 = 0.11 BTC
  Taker account residual: 0.05 - 0.01 - fee_share = 0.04 BTC
```

After 1 BTC conf, the channel opens. Lease tracker stored on
auctioneer; both peers can use the channel.

## Common pitfalls

- **Auctioneer single point**: LL's auctioneer is centralized.
  Failure mode: auction halts, but accounts remain fully owned by
  user (auctioneer can't steal).
- **Lease enforcement**: relies on maker honoring CLTV / contract;
  auctioneer co-signs to prevent unilateral early close. If maker's
  node is compromised, no on-chain refund mechanism for taker.
- **Premium price discovery**: thin liquidity = wide bid-ask spread;
  small participants get worse prices.
- **Reorg of batch**: deep BTC reorg breaks the multisig accounts;
  Pool requires multiple confs before considering trade final.
- **Account custody risk**: 2-of-2 with auctioneer means if
  auctioneer disappears, user must use timelocked recovery path
  (unilateral after long delay).

## References

- Lightning Pool docs (lightning.engineering/pool).
- "Pool: Lightning Liquidity Marketplace" — Lightning Labs blog.
- BOLT-02 channel funding (referenced by Pool's leased-channel scheme).
