# Bisq Trade Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/privacy/p2p-exchanges`.
> Canonical source: https://bisq.wiki/Trading
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/privacy/p2p-exchanges/SKILL.md

## Concept

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
- bisq-network/bisq GitHub: trade-protocol.proto.
