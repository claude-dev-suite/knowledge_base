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
has been overtaken in volume by Ordinals/Inscriptions and Taproot Assets. The reference
implementation is still under active development -- Counterparty Core v11.3.0
(16 August 2026) is the current release, and 2026 brought protocol-level additions
including constant-product AMM liquidity pools and indefinite DEX orders (mainnet gate
block 952,800).

## Walkthrough / mechanics

### Message embedding modes (chronological)

1. **Multisig encoding (2014 - 2018)**: Counterparty data was split across the public
   keys of a 1-of-3 bare multisig output. Stored ~64 bytes per output, no OP_RETURN size
   limit, but inflated UTXO set permanently.
2. **OP_RETURN single-output (2014 onward, primary mode after 2017)**: Up to 80-byte
   payload in an unprunable OP_RETURN.
3. **OP_RETURN concatenation across outputs**: For long messages, multiple OP_RETURNs.
4. **Taproot envelope (2025 onward)**: Counterparty Core v11.0.0 (27 May 2025) added
   Taproot envelope data encoding at activation block 902,000 (mined 20 June 2025),
   explicitly to cut fees on larger messages, and removed P2SH data encoding at the
   same time. The envelope script is Ordinals-compatible, so an issuance / fairminter /
   broadcast can carry an inscription.

With `encoding: "auto"` (the compose default in v11.3.0) the composer still picks
`opreturn` when the payload plus the `CNTRPRTY` prefix fits `OP_RETURN_MAX_SIZE = 80`
bytes and falls back to `multisig` otherwise; `taproot` must be requested explicitly and
is rejected for UTXO-attached sources, non-segwit sources, transactions carrying a
destination output, and `detach`.

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
- **OP_RETURN size limit**: historically Bitcoin's relay policy allowed a single
  OP_RETURN output carrying 80 data bytes (`-datacarriersize=83`), so longer messages
  had to chunk across outputs, increasing fees. Bitcoin Core v30.0 (10 October 2025)
  raised the default `-datacarriersize` to 100,000 and now relays and mines multiple
  OP_RETURN outputs per transaction, the limit applying to their aggregate scriptPubKey
  size; the standard transaction size binds first, so the policy cap is effectively
  gone. This is relay policy, not consensus, and `-datacarrier` / `-datacarriersize`
  stay configurable -- Core v31.1 (8 July 2026) still sets `MAX_OP_RETURN_RELAY` to
  `MAX_STANDARD_TX_WEIGHT / WITNESS_SCALE_FACTOR`, i.e. 100,000 vbytes. Bitcoin Knots
  keeps the old default, `MAX_OP_RETURN_RELAY = 83` as of the v29.4.1.knots20260508
  release (2 September 2026), so the real constraint is now which node software sits on
  the relay path rather than one network-wide number. Counterparty Core itself has not
  followed: `config.OP_RETURN_MAX_SIZE` is still `80` in v11.3.0, so its composer keeps
  chunking or switching encoding above that.
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
- counterparty-core release notes - https://github.com/CounterpartyXCP/counterparty-core/releases
- Bitcoin Core 30.0 release notes (OP_RETURN policy) - https://github.com/bitcoin/bitcoin/blob/v30.0/doc/release-notes.md
- Original whitepaper (2014) - https://counterparty.io/files/CounterpartyWhitepaper.pdf
