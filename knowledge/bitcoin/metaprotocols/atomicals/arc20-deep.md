# ARC-20 (Atomicals) - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/metaprotocols/atomicals`.
> Canonical source: https://docs.atomicals.xyz/protocols/arc20-fungible-tokens
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/metaprotocols/atomicals/SKILL.md

## Concept

ARC-20 is the Atomicals protocol's fungible-token specification. Its
defining design choice: 1 ARC-20 token unit equals exactly 1 satoshi.
Token amounts are not abstract numbers tracked off-chain - they are
the underlying sat balances of the UTXOs themselves. This means
Bitcoin's UTXO accounting *is* the token accounting; an indexer is
needed only to map "which sats belong to which ticker", not to
compute balances. Sending sats moves tokens 1:1; spending below the
dust limit destroys both.

## Walkthrough / mechanics

The protocol uses Atomicals' "envelope" inscriptions (similar shape
to ord envelopes but with a different protocol tag). Operations:

- `dft` (deploy fungible token): registers a ticker with `max_supply`
  measured in sats, `mint_amount` (sats per mint event), and
  `mint_height` activation. Strictly first-deploy-wins per ticker.
- `dmt` (decentralized mint): a tx whose first input is a previously
  prepared "minting" UTXO and whose first output equals exactly
  `mint_amount` sats. Indexer credits those sats as the new ARC-20
  tokens of the ticker.
- `mod` / `evt`: optional metadata updates.
- Transfers are implicit: any standard tx that moves the token sats
  is a transfer. There is no special op.

Because tokens are sats, a wallet must *not* spend ARC-20 sats as
fee or general payment. Atomicals-aware wallets pin the carrying
UTXOs and exclude them from coin selection.

Identity model: each Atomical (FT or NFT) has a unique 36-byte ID =
`txid || vout` of the operation that created it. The ID stays
constant; the carrying UTXO chain follows the sats.

## Worked example  (encoded data, real txs, configs)

Deploy ticker `atom` with 21_000_000 token cap and 1000 sats per
mint event:

```json
{
  "args": {
    "request_ticker": "atom",
    "max_mints":      21000,
    "mint_amount":    1000,
    "mint_height":    830000,
    "mint_bitworkc":  "aabb"
  }
}
```

Wait - max_supply for ARC-20 = max_mints * mint_amount = 21M sats =
0.21 BTC of total token sats locked away from circulation. To deploy
a 21_000_000-token ARC-20 you must commit 0.21 BTC of dust UTXOs
across all minters' wallets combined.

Mint tx (decentralized):

```
input 0:  <prepared 1000-sat UTXO meeting bitwork>
output 0: 1000 sats to <minter address>
output 1: OP_RETURN containing dmt envelope
          payload: { "args": { "mint_ticker": "atom", "i": <index> } }
```

The `bitworkc` field requires the commit txid to start with hex
`aabb` (proof-of-work mint), preventing bot-only scenarios. Each
minter must grind nonces to find a valid commit txid.

Transfer 500 atom: a normal Bitcoin tx that includes a 500-sat-or-
greater portion of the ARC-20-tagged UTXO and routes 500 sats to
the recipient. The Atomicals indexer follows the FIFO sat flow (same
as ord ordinals) to determine which sats among the input-output mix
carry which token.

## Trade-offs / pitfalls

- **Capital lock**: tokens occupying sats means high-supply ARC-20s
  consume meaningful BTC. A 1B-token ARC-20 locks 1B sats = 10 BTC of
  dust. This is by design (no inflated supply illusions) but limits
  cap sizes to what users will lock.
- **Dust limit**: 1-token UTXOs are below Bitcoin's standard 546-sat
  dust limit and cannot be sent in standard tx. Practical minimum
  per-output token amount is 546.
- **Coin-selection footgun**: any non-Atomicals wallet that touches
  these UTXOs may consolidate them with other change, mixing token
  sats with non-token sats. The Atomicals indexer still tracks the
  exact sats but downstream apps may misreport balance.
- **Fee tax**: every move pays Bitcoin fees on the sats themselves,
  not abstract token amounts. Moving 1_000 atom = moving 1000 sats
  = a P2TR output. Real fee dwarfs the token notional in cheap-token
  scenarios.
- **PoW mint**: the `bitworkc` requirement forces grinding. Casual
  minters lose to specialised hashing scripts. Comparable to early
  BRC-20 mint races in unfairness.
- **Indexer adoption**: smaller than Ordinals/Runes. Most CEXes do
  not support ARC-20 deposits; secondary markets are limited to a
  few specialised platforms (Atomical Market, Wizz).
- **AVM not live**: the proposed Atomicals VM remains experimental;
  programmability features advertised in marketing material are not
  production today.

## References

- ARC-20 docs: https://docs.atomicals.xyz/protocols/arc20-fungible-tokens
- Atomicals reference: https://github.com/atomicals/atomicals-electrumx
- Bitwork PoW: https://docs.atomicals.xyz/specifications/bitwork
