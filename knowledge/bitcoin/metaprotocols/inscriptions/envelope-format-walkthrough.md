# Inscription Envelope Format - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/metaprotocols/inscriptions`.
> Canonical source: https://docs.ordinals.com/inscriptions.html
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/metaprotocols/inscriptions/SKILL.md

## Concept

An inscription is a sequence of fields embedded in a Taproot
script-path witness. The "envelope" is a Bitcoin script structured
so that pre-Taproot validators see only `OP_FALSE OP_IF ... OP_ENDIF`
which is unconditionally skipped, while ord-aware indexers parse the
pushed bytes to reconstruct content type, body, and other metadata.
The clever trick is that envelopes occupy witness data, which is
discounted 4x in fee accounting (1 weight unit per byte instead of
4), making large content economically viable.

## Walkthrough / mechanics

The envelope has the shape:

```
OP_FALSE
OP_IF
  OP_PUSH "ord"          ; 3-byte protocol tag
  OP_PUSH <tag>          ; even-numbered field tag
  OP_PUSH <value>        ; field value
  ... more tag/value pairs ...
  OP_PUSH 0              ; body tag
  OP_PUSH <chunk_1>      ; up to 520 bytes per push
  OP_PUSH <chunk_2>
  ...
OP_ENDIF
```

Field tags are single bytes. Even tags carry data; odd tags are
reserved for future use and indexers must ignore unknown odd tags
(forward-compatible). Defined fields:

```
0  body              raw content
1  content_type      ASCII MIME (e.g. "image/png")
2  pointer           varint sat offset within the receiving UTXO
3  parent            32-byte txid + varint output index of parent
5  metadata          CBOR-encoded structured metadata
6  delegate          inscription ID to delegate body to
7  metaprotocol      ASCII tag (e.g. "brc-20")
9  content_encoding  HTTP-style encoding (e.g. "br", "gzip")
```

Body chunks must be the trailing pushes; once tag 0 appears, every
subsequent push is concatenated until OP_ENDIF. Each push is bounded
by the 520-byte limit on standard script pushes.

The envelope is wrapped inside a Taproot leaf script:

```
<reveal_pubkey> OP_CHECKSIG
OP_FALSE OP_IF ... OP_ENDIF
```

The CHECKSIG forces the spender to provide a signature, ensuring the
output is owned (not spendable by anyone who knows the script).

## Worked example  (encoded data, real txs, configs)

Inscribe a 4-byte text "hi!\n" with content type `text/plain;
charset=utf-8`. The envelope script bytes:

```
00                                        ; OP_FALSE (0x00)
63                                        ; OP_IF
03 6f 72 64                               ; PUSH 3 bytes "ord"
01 01                                     ; PUSH 1 byte (tag 1 = content_type)
18 74 65 78 74 2f 70 6c 61 69 6e 3b 63 68
   61 72 73 65 74 3d 75 74 66 2d 38      ; PUSH 24 bytes "text/plain;charset=utf-8"
01 00                                     ; PUSH 1 byte (tag 0 = body)
04 68 69 21 0a                            ; PUSH 4 bytes "hi!\n"
68                                        ; OP_ENDIF
```

Total leaf script size: 41 bytes. After the reveal-key prefix (33 bytes
+ OP_CHECKSIG) the leaf is 75 bytes. Witness weight ~75 + control
block (33 bytes) + signature (64 bytes) = 172 weight units. At a
100 sat/vB fee that costs roughly 4_300 sats.

A 100 KB inscription needs ceil(100_000 / 520) = 193 body pushes.
Each push has a 1-byte length prefix for sizes up to 75 and 2 bytes
(OP_PUSHDATA1) for 76..255 and 3 bytes (OP_PUSHDATA2) for 256..520.
Total envelope is roughly 100_400 bytes; weight 100_400 wu;
virtual size 25_100 vB. At 50 sat/vB that is 1_255_000 sats
(~$700 at $55K BTC).

## Trade-offs / pitfalls

- Indexers must reject envelopes with unknown EVEN tags but accept
  unknown ODD tags. Getting this wrong creates indexer disagreement
  on edge-case inscriptions.
- Multiple envelopes in a single witness are valid; each becomes a
  separate inscription on consecutive sats. ord allocates them by
  spending order.
- Standardness: pre-v25 Bitcoin Core had a 100_000-byte witness limit
  per push that made > 400 KB inscriptions non-relayable. Modern
  policy permits up to the consensus block weight.
- A reveal that fails CHECKSIG (wrong sig) cannot be mined, so the
  commit output is stuck until the dust limit makes it unspendable.
- Bitcoin Knots and some other node implementations apply local
  filters that reject inscription-shaped txs from their mempool. They
  still relay if mined, since the pattern is consensus-valid.

## References

- ord envelope spec: https://docs.ordinals.com/inscriptions.html#fields
- BIP341 (Taproot): https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki
- Push opcodes: https://en.bitcoin.it/wiki/Script
