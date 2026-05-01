# Spacechains Mechanics - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/drivechains-spacechains`.
> Canonical source: https://github.com/RubenSomsen/Spacechains-WP/blob/main/spacechains.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/drivechains-spacechains/SKILL.md

## Concept

Spacechains (Ruben Somsen 2018) are an alternative to drivechains: a
sidechain-like architecture that uses **PermaPoW** ("non-burnable"
proof-of-work via blind merge mining) **without** any Bitcoin native peg
mechanism. Spacechains have no two-way peg with Bitcoin; their native
asset is independent. They piggyback Bitcoin's hashrate via blind merge
mining (a la BIP-301) but make zero changes to Bitcoin's consensus.

## Walkthrough / mechanics

### Architecture summary

- **Spacechain**: a sidechain blockchain with its own native coin
  (e.g. SPACE).
- **Bitcoin**: provides ordering / hashrate via BMM.
- **No peg**: SPACE coins cannot be exchanged 1:1 with BTC via any
  protocol-level mechanism. They're a separate asset, like any altcoin.

### Why no peg?

Drivechain peg requires miners to vote on withdrawals (BIP-300), which
introduces new trust assumptions and arguably miner power Bitcoin
holders dislike. Spacechains skip the peg entirely; SPACE is a
separate-coin asset users can buy/sell on exchanges. The only Bitcoin
connection is the blind-merge-mined block hash.

### Blind merge mining flow

1. Spacechain user / sequencer creates a Spacechain block, computes hash
   `H_sc`.
2. They post a **blind-bid** Bitcoin transaction: `OP_RETURN <H_sc>` +
   miner fee (in BTC).
3. Bitcoin miner includes the bid (collects BTC fee). Bid is committed
   to the Spacechain block by virtue of the OP_RETURN.
4. Once the Bitcoin tx confirms, Spacechain block is canonical.

Multiple bids in the same Bitcoin block conflict; only one wins (per
spec, the highest fee). This creates an **auction** for Spacechain
blockspace at each Bitcoin block.

### Properties

- **Permissionless**: anyone can propose a Spacechain block by paying
  Bitcoin fees.
- **Spam-resistant**: high BTC bids deter low-value Spacechain blocks.
- **No two-way peg**: SPACE is not BTC; cannot be redeemed for BTC.
- **Bitcoin neutrality**: zero soft-fork required.

### Why this is desirable

Bitcoin maximalists who oppose drivechain pegs but want experiment-friendly
sidechains favor Spacechains: they don't add miner-vote trust to Bitcoin,
they don't enable new BTC-issuance attacks, but they let useful sidechain
features (smart contracts, better privacy, etc.) develop.

### Atomic Swaps for transfers

Without a native peg, BTC <-> SPACE exchange happens via on-chain or
off-chain atomic swaps (HTLC, PTLC, or scriptless adaptor signatures).
Same mechanism as cross-chain DEXes today.

## Worked example

Spacechain "AltSpace" with privacy-focused features (e.g. native
CoinJoin). User Alice wants to use AltSpace.

```
1. Alice buys 100 SPACE on a centralized exchange (or earns SPACE).
2. Alice runs an AltSpace node, queries balance.
3. Alice produces an AltSpace block (or pays a sequencer to do it).
4. Alice or sequencer constructs Bitcoin BMM bid:
     OP_RETURN <H_altspace_block>  + 5000 sat fee
5. Bitcoin miner includes Alice's bid in next block. AltSpace block
   confirmed.
6. AltSpace transactions in that block are now canonical.
```

To convert SPACE to BTC, Alice runs an HTLC swap with another user
holding BTC. No protocol-level peg involved.

## Common pitfalls

- **No Bitcoin-native exit**: SPACE != BTC. Users must buy/sell SPACE
  on markets. Highly experimental sidechains may have low liquidity.
- **Bid-auction MEV**: high-fee bids win. Spacechain ordering is then
  determined by who can pay highest BTC fees, which has economic
  implications for Spacechain users.
- **Reorg sensitivity**: a Bitcoin reorg invalidates Spacechain block
  confirmation. Users wait for deep BTC confs before considering
  Spacechain finality.
- **Block production cost**: each Spacechain block costs a Bitcoin fee.
  At low Spacechain volume, fees per tx may be very high.
- **Status as research**: no production Spacechain exists (as of 2026).
  Multiple proposals (Spacechain-WP, Bitcoin-staked alt-chains) are at
  experimental stage.

## References

- Somsen. "Spacechains: A Whitepaper" 2018.
- BIP-301 — Blind Merge Mining (compatible primitive).
- Bitcoin atomic swaps via Schnorr adaptor signatures (Poelstra 2017).
