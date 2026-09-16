# Bisq Trade Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/privacy/p2p-exchanges`.
> Canonical source: https://bisq.wiki/Trading
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/privacy/p2p-exchanges/SKILL.md

## Concept

This article describes **Bisq 1** (the `bisq-network/bisq` desktop app).
Bisq 2's Bisq Easy protocol is a separate codebase with no multisig
escrow - see the skill.

Bisq is a **decentralized P2P fiat <-> BTC exchange** built on Tor and a
2-of-2 + arbitrator multisig scheme. There is no custody and no server:
order books propagate via gossip across Tor, and trades are escrowed on
chain in a Bitcoin multisig. Disputes are resolved by an arbitrator pool;
otherwise the protocol is fully peer-to-peer.

## Walkthrough / mechanics

### Roles

- **Maker**: posts a buy/sell offer in the gossiped order book.
- **Taker**: accepts the offer, kicks off the trade.
- **Arbitrator / Mediator**: human dispute resolver, keys held in
  Bisq DAO-elected accounts.

### Security deposits

Both parties lock a **security deposit** (typically 15 % of trade value)
into the multisig along with the trade amount. Lost-by-default if a party
acts in bad faith.

### Trade limits

`TradeLimits.MAX_TRADE_AMOUNT` is a hard network cap that clamps the DAO
`MAX_TRADE_LIMIT` parameter (default 2 BTC); per-payment-method limits are
derived from the clamped value, and a local `userDefinedTradeLimit`
(default 0.1 BTC) can only lower it further. The constant is new in the
1.10 line - v1.9.21 has no `MAX_TRADE_AMOUNT` at all, and before it the
ceiling came from the DAO parameter alone. It was introduced at
**0.125 BTC** in v1.10.0 as part of the May 2026 incident response and
raised to **0.250 BTC** in v1.10.1, where it still stands in v1.10.7
(25 August 2026). Buyers paying with a chargeback-risk fiat method in a
mature-market currency are additionally held to 0.002 BTC until their
account witness is signed; selling BTC is never reduced by signing state.

### 1 May 2026 exploit and hardening

The taker defines the miner fee for the trade transactions and passes
that value to the maker. It was not validated against negative numbers,
so an attacker could make the maker compute an incorrect multisig output
value. **11.59104 BTC was lost across 10 users**, all on altcoin trades -
fiat trades were protected by the account-age witness signing system.
Refund Angels advanced 10.98538295 BTC (~890,387 USD at the reference
rate of 81,052 USD/BTC for 15 May 2026) to reimburse victims. Repayment
is *proposed*, not ratified: part of the BTC fees that would otherwise go
to Burning Men would be redirected to the Refund Angels via the Filter
object, phased in over successive DAO cycles at 25 % / 50 % / 75 %. The
post calls a 12-18 month settlement horizon realistic and states that no
fee parameter change is finalized by it (Bisq blog, 16 June 2026); each
change still has to pass the normal DAO proposal and vote.

`v1.10.0` added validation across deposit, payout, delayed-payout and
mediated-payout transactions, trade amounts and prices, maker/taker fees,
miner fees, multisig public keys, raw inputs and canonical transaction
structure; capped acceptable price deviation at 25 %; and disabled XMR
auto-confirmation by network filter pending a security audit. Bisq 2 /
Bisq Easy were unaffected.

### Trade tx structure

Two on-chain transactions:

1. **Deposit Tx**: 2-of-2 multisig output `(M_buyer, M_seller)`. Inputs:
   buyer deposit + buyer payment + seller deposit. Output: single multisig
   UTXO holding `2 * deposit + amount_BTC`.
2. **Payout Tx**: spends the multisig. Outputs: refund seller deposit,
   refund buyer deposit, send `amount_BTC` to buyer (or back to seller in
   cancel/dispute).

### Tor addresses + onion-only matching

Both peers identify only by Tor v3 onion addresses. The matching protocol:

1. Maker signs offer with onion-bound key, broadcasts.
2. Taker connects via Tor to maker's onion, sends "take offer" message.
3. Both peers run BSQ (Bisq's coloured-coin token) verification to confirm
   on-chain account age and reputation (limits Sybil risk).
4. Maker constructs deposit tx PSBT, taker co-signs, deposit broadcast.
5. Off-chain fiat: maker sends bank wire / SEPA / cash deposit per the
   offer. Buyer confirms in-app once received.
6. Both sign payout tx releasing BTC to buyer + deposits back to both.

### Arbitration path

If buyer claims "no fiat received", trade flips to mediator. If mediation
fails, arbitrator unilaterally signs a payout tx using a 1-of-1 escape key
broadcast as a delayed-payout tx (timelocked roughly 20 days from deposit).

## Worked example

Alice (maker) sells 0.05 BTC for EUR via SEPA. Deposit 15 %.

```
Inputs (Deposit Tx):
  Alice  : 0.05 + 0.0075 (deposit) + fee = 0.058
  Bob    : 0.0075 (deposit) + fee
Output:
  multisig (Alice_pk, Bob_pk)  0.065 BTC
```

Bob wires EUR to Alice's IBAN. Alice marks "received". Both wallets
co-sign Payout Tx:

```
Output 1: Alice deposit return  0.0075 - fee/2
Output 2: Bob deposit return    0.0075 - fee/2
Output 3: Bob receives          0.0500 - fee/2
```

If Bob never wires, Alice raises mediation. Mediator signs payout tx
returning Alice's deposit + her 0.05 BTC, applying a slashing penalty to
Bob's deposit (split between mediator pool and Alice).

## Common pitfalls

- **No KYC** but a **bank wire still has KYC**: Alice's SEPA counterparty
  links a real-name bank account to the Bisq trade. Privacy gain is
  on-chain, not off-chain.
- **Deposit amount is large** for small trades: 15 % means a 100 EUR
  trade locks ~115 EUR for hours. Pricing is calibrated to discourage
  small abusive trades but discourages small honest ones too.
- **Tor reachability**: clients must be reachable as onion services for
  trades to start. NAT'd users without Tor hidden service config see
  "trades stuck".
- **DAO upgrade governance**: arbitrator and mediator keys are tied to
  the BSQ DAO; participating in upgrades is a soft requirement to
  trust the dispute system.
- **Deterministic deposit funding**: do not reuse the same UTXO across
  cancelled trades; cancel attempts can be linked.

## References

- Bisq wiki (https://bisq.wiki).
- Bisq whitepaper v2.0.
- bisq-network/bisq GitHub: trade-protocol.proto, docs/trade-limits.md,
  release-notes/1.10.0/notable-changes.md.
- Bisq blog: security incident post-mortem, and "Where Bisq stands after
  the security incident" (16 June 2026).
