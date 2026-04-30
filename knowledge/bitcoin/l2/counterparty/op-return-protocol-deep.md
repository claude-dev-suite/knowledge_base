# Counterparty OP_RETURN Protocol - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/counterparty`.
> Canonical source: https://docs.counterparty.io/docs/protocol/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/counterparty/SKILL.md

## Concept

Counterparty (XCP) is the original metaprotocol on Bitcoin: launched January 2014, it
embeds protocol messages in Bitcoin transactions and indexes them off-chain to derive
asset balances and contract state. Counterparty's "OP_RETURN protocol" is the encoding
scheme by which messages (issue, send, dividend, dispenser, etc.) are committed to
Bitcoin. Today Counterparty is most relevant historically (Rare Pepes, Spells of Genesis)
and as the underlying registry for some collectible communities; it remains active but
has been overtaken in volume by Ordinals/Inscriptions and Taproot Assets.

## Walkthrough / mechanics

### Message embedding modes (chronological)

1. **Multisig encoding (2014 - 2018)**: Counterparty data was split across the public
   keys of a 1-of-3 bare multisig output. Stored ~64 bytes per output, no OP_RETURN size
   limit, but inflated UTXO set permanently.
2. **OP_RETURN single-output (2014 onward, primary mode after 2017)**: Up to 80-byte
   payload in an unprunable OP_RETURN.
3. **OP_RETURN concatenation across outputs**: For long messages, multiple OP_RETURNs.
4. **Witness data (taproot era, niche)**: experimental newer encoding.

The dominant mode today is OP_RETURN.

### Message format

Each Counterparty message:

```
PREFIX  ("CNTRPRTY", 8 bytes)
TYPE_ID (4 bytes BE)
PAYLOAD (variable, type-specific)

Whole payload is RC4-encrypted with the first input's txid as the key
(historical artifact -- not security, but obfuscation against trivial parsers).
```

After RC4 decryption:

```
TYPE_ID = 0  -> Send
TYPE_ID = 1  -> Order
TYPE_ID = 10 -> Issuance (asset creation / supply change)
TYPE_ID = 50 -> Dividend
TYPE_ID = 21 -> Btcpay (settle a BTC-leg of an order)
... etc.
```

### Issuance

```
Issuance message {
  asset_id:        u64 (numeric or named via separate registration)
  quantity:        i64 (positive issues, negative locks supply)
  divisible:       bool
  description:     ascii string up to 41 bytes
}

Source = first input's address.  This address is recorded as the issuer.
A "named asset" requires burning 0.5 XCP at issuance for the privilege of a
non-numeric ticker.
```

### Send

```
Send message {
  asset_id:        u64
  quantity:        u64
}

Source = first input's address.
Destination = first non-OP_RETURN, non-fee output's address.
```

### Indexer model

Counterparty itself doesn't have its own chain. **counterparty-server** (formerly
counterparty-lib) is an indexer that:

1. Reads each Bitcoin block via bitcoind RPC.
2. Scans every tx for OP_RETURNs.
3. Decodes Counterparty messages.
4. Updates a SQLite/Postgres database of asset balances.
5. Exposes a JSON-RPC API for wallet apps.

There is no consensus among indexers beyond reading the same Bitcoin chain; differences
arise only from indexer bugs.

## Worked example

Issuance and transfer of "RAREPEPE" asset:

```
Step 1: Issuer Joe Looney burns 0.5 XCP for "RAREPEPE" name (separate tx).
        After confirm, "RAREPEPE" is reserved to Joe.

Step 2: Joe issues 5000 RAREPEPE
  Bitcoin tx:
    IN:  Joe's UTXO
    OUT: 0  -> OP_RETURN: rc4(CNTRPRTYIssuance{asset:RAREPEPE, qty:5000, div:false, desc:""})
    OUT: 1  -> 546 sat to Joe (dust, recipient = source)
    OUT: 2  -> change
  After bitcoind confirm:
    counterparty-server decodes -> credits Joe with 5000 RAREPEPE.

Step 3: Joe sends 1 RAREPEPE to Alice
  Bitcoin tx:
    IN:  Joe's UTXO
    OUT: 0 -> OP_RETURN: rc4(CNTRPRTYSend{asset:RAREPEPE, qty:1})
    OUT: 1 -> 546 sat to Alice  (this is the destination)
    OUT: 2 -> change to Joe
  After confirm:
    counterparty-server decodes -> debits Joe by 1, credits Alice by 1.

Step 4: Alice queries
  > xcp-cli get_balances --address alice_btc_addr
  [{ asset: "RAREPEPE", quantity: 1, divisible: false }]
```

## Trade-offs and security

- **Indexer trust**: balances are computed off-chain by indexers. Different indexers
  (counterparty-server, alternative implementations) must agree -- bugs cause
  consensus-relevant divergence visible in wallet UIs.
- **Reorg sensitivity**: an indexer must re-process on Bitcoin reorg. Most indexers
  wait 6 confs to display balances.
- **Bloat history**: pre-2018 multisig encoding permanently burned ~3 sats UTXOs into
  Bitcoin's UTXO set. That's an ongoing cost.
- **OP_RETURN size limit**: Bitcoin's policy limit (80 bytes by default) constrains
  message sizes; longer messages chunk across multiple outputs, increasing fees.
- **No smart contracts in modern sense**: Counterparty supports orders, dispensers, and
  dividends but not arbitrary state machines. CIP-141 added bet/contract logic that has
  been deprecated due to use issues.
- **Privacy**: messages reveal asset, amount, source, destination on-chain (encryption
  is RC4 with public key derivation -- effectively zero privacy).
- **Modern alternatives**: Ordinals/Inscriptions and Taproot Assets / RGB cover most
  use cases more efficiently. Counterparty lives on for legacy Rare Pepe and other
  collectibles communities.

## References

- Counterparty docs - https://docs.counterparty.io/docs/protocol/
- counterparty-server - https://github.com/CounterpartyXCP/counterparty-core
- "Counterparty 2.0" announcement (2024 indexer rewrite)
- Original whitepaper (2014) - https://counterparty.io/files/CounterpartyWhitepaper.pdf
