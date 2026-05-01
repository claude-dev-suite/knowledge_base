# UMA Multi-Currency Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/uma`.
> Canonical source: https://docs.uma.me/uma-protocol/sending-payments
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/uma/SKILL.md

## Concept

UMA addresses cross-currency Lightning payments natively: a sender in
USD can pay a receiver in EUR (or pesos, sats, etc.) and the protocol
negotiates the FX conversion in the LNURL-pay handshake. The receiver's
VASP advertises supported currencies with min/max amounts and live
conversion rates; the sender's VASP picks one and pays the resulting
BOLT11 invoice.

## Walkthrough / mechanics

### Currencies declaration

In the LNURLP response, the receiver VASP includes a `currencies` array:

```json
"currencies": [
  {
    "code": "USD",
    "name": "US Dollar",
    "symbol": "$",
    "multiplier": 1052.5,
    "decimals": 2,
    "convertible": {"min": 1, "max": 1000000}
  },
  {
    "code": "EUR",
    "name": "Euro",
    "symbol": "€",
    "multiplier": 998.4,
    "decimals": 2,
    "convertible": {"min": 1, "max": 1000000}
  },
  {
    "code": "BRL",
    "multiplier": 5240.0,
    ...
  },
  {
    "code": "SAT",
    "multiplier": 1.0,
    "decimals": 0
  }
]
```

`multiplier` is **msat per smallest unit of the currency**. So 1 USD =
1052.5 msat (illustrative example: actual values follow live BTC/USD
rate). For EUR with 2 decimals: 1 cent = 998.4 msat -> 1 EUR = 99 840
msat.

### Selection by sender

Sender VASP picks the receiver currency based on user preference. It
shows the user the quoted price ("$50 USD ≈ 47 EUR"). On confirmation,
sender VASP sends a `payreq` with both:

- `amount` (msat — final payment value)
- `currencyCode` (e.g. "USD")

The `amount` is the receiver's native amount times their multiplier:

```
amount_msat = user_amount_in_currency * multiplier
            = 50.00 USD * 100 cents/USD * 1052.5 msat/cent
            = 50 * 100 * 1052.5
            = 5,262,500 msat
```

### Receiver invoice

Receiver VASP returns a BOLT11 invoice for `amount_msat`. They also
declare `paymentInfo`:

```json
{
  "currencyCode": "USD",
  "multiplier": 1052.5,
  "exchangeFeesMillisatoshi": 5000,
  "decimals": 2
}
```

Sender VASP can verify the multiplier hasn't changed unexpectedly between
requests (replay-resistance via timestamp).

### Sender currency

UMA optionally supports **sender currency**: sender pays $50 USD; their
VASP debits USD account, converts internally, and pays the LN invoice
in msat. The protocol doesn't mandate sender-side conversion details
(VASP-internal).

### Rate freshness

Multiplier is timestamped. UMA spec recommends max 5-minute freshness;
older quotes must be re-fetched. Receiver VASP signs the LNURLP response
including timestamp, so freshness is auditable.

### Receiver fees

Receiver VASP may charge an FX spread by inflating the multiplier
(charging more msat per fiat unit). `exchangeFeesMillisatoshi` is a
flat fee on top, declared separately for transparency.

### MPP / multi-path

For large amounts, sender VASP may MPP the BOLT11 invoice. The exchange
rate applies once (at quote time); MPP just splits the msat payment
across paths.

## Worked example

Bob in Brazil pays Alice in Germany €25:

```
LNURLP response:
  EUR: multiplier=998.4 msat/cent, decimals=2
  BRL: multiplier=5240.0 msat/cent, decimals=2
  SAT: multiplier=1.0 msat/sat, decimals=0

payreq:
  amount = 25.00 EUR = 25 * 100 * 998.4 = 2,496,000 msat
  currencyCode = EUR

invoice = lnbc24960u1p... (24,960 sat)

Sender VASP debits Bob's BRL account at internal rate ~5300 BRL/EUR
=> 132.50 BRL (with internal spread).

Sender VASP pays invoice over LN; Alice's VASP credits 25.00 EUR to
Alice's account.
```

## Common pitfalls

- **Multiplier change mid-flow**: between LNURLP and payreq, BTC/USD
  could have moved 1-2 %. UMA spec requires re-quote if amount differs
  by more than threshold.
- **Decimals confusion**: USD has 2 decimals (cents), JPY has 0
  (yen). VASPs must use the right `decimals` field; bug-prone.
- **Trailing zero msat**: msat is finest granularity; if rounding leaves
  fractional msat, round down (favor sender).
- **Sanctions per currency**: some VASPs refuse certain corridors
  (e.g. RUB, IRR). Compliance check happens before quote.
- **Cold rates**: idle VASPs cache stale FX rates; users see surprising
  prices when hot price feed updates.

## References

- UMA documentation, "Currencies" section.
- BOLT11 Lightning invoice spec.
- LUD-06 LNURL-pay base spec.
