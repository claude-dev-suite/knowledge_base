# BIP329 Label Format - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/wallets/labels`.
> Canonical source: https://github.com/bitcoin/bips/blob/master/bip-0329.mediawiki
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/wallets/labels/SKILL.md

## Concept

BIP329 standardizes how wallets export and import user-supplied labels using JSON Lines (NDJSON / JSONL). One JSON object per line, each describing a label attached to one of six entity types: `tx`, `addr`, `pubkey`, `xpub`, `input`, `output`. The format is line-oriented so it streams cleanly through pipes, is diffable in version control, and can be appended to without rewriting the file. There is no schema beyond the per-line object, no envelope, and no required ordering. Wallets MAY add extra fields like `origin` and `spendable`; consumers MUST ignore unknown fields rather than reject the file.

## Walkthrough / mechanics

Per-line schema (TypeScript-style):

```ts
type Label =
  | { type: "tx";      ref: Hex64;  label: string; origin?: string }
  | { type: "addr";    ref: Address; label: string; origin?: string }
  | { type: "pubkey";  ref: HexPub; label: string; origin?: string }
  | { type: "xpub";    ref: Xpub;   label: string; origin?: string }
  | { type: "input";   ref: `${Hex64}:${number}`; label: string; origin?: string }
  | { type: "output";  ref: `${Hex64}:${number}`;
      label: string; origin?: string; spendable?: boolean };
```

`ref` semantics:

| `type`   | `ref` value                                       |
|----------|----------------------------------------------------|
| `tx`     | 64-hex txid (display order, big-endian)            |
| `addr`   | encoded address (Bech32/Bech32m/Base58)            |
| `pubkey` | 33-byte hex compressed pubkey                      |
| `xpub`   | base58check xpub/ypub/zpub/Ypub/Zpub               |
| `input`  | `<txid>:<vin_index>` decimal index                 |
| `output` | `<txid>:<vout_index>` decimal index                |

UTF-8 is mandatory; the file SHOULD be saved without BOM. Each line ends with `\n` (LF only). Empty lines and `#`-prefixed comments are NOT part of the spec; some wallets tolerate them but consumers MUST NOT rely on them.

Round-trip rules:

1. Unknown `type` values: skip silently.
2. Unknown extra fields: preserve on re-export to avoid lossy round-trips when possible.
3. Duplicate `(type, ref)` keys: last write wins; a stricter consumer MAY warn.
4. Strings: NFC-normalized Unicode, no length cap (wallets often display the first 50 chars).
5. `spendable: false` on an `output`: the consumer SHOULD exclude that UTXO from automatic coin selection but allow manual coin control.

Filename convention: `<wallet>-labels.jsonl`. MIME type `application/jsonl` or `application/x-ndjson`.

## Worked example

A real-shape export from a multi-account Sparrow wallet (anonymized):

```jsonl
{"type":"xpub","ref":"zpub6rFR7y4Q2AijBEqTUquhVz398htDFrtymD9xYYfG1m4wAcvPhXNfE3EfH1r1ADqtfSdVCToUG868RvUUkgDKf31mGDtKsAYz2oz2AGutZYs","label":"Spending account"}
{"type":"addr","ref":"bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh","label":"Coffee shop tip jar"}
{"type":"addr","ref":"bc1q7cyrfmck2ffu2ud3rn5l5a8yv6f0chkp0zpemf","label":"Friend Alice"}
{"type":"tx","ref":"f91d0a8a78462bc59398f2c5d7a84fcff491c26ba54c4833478b202796c8aafd","label":"Coffee on 2026-04-12"}
{"type":"output","ref":"f91d0a8a78462bc59398f2c5d7a84fcff491c26ba54c4833478b202796c8aafd:0","label":"Sent to Alice","spendable":false}
{"type":"input","ref":"a32fd4b0e4d2f1c97e4e1f4c2a1b3d8e7f5b9c6e1a8d4c2e1f9b3a5d6e1c2b9f:1","label":"Spent from cold storage"}
{"type":"pubkey","ref":"02f7e2b6e4f1c5e2a8b3d4e5f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1","label":"Cosigner Bob (hardware)"}
{"type":"xpub","ref":"zpub6rFR7y4Q2AijBfsVQ8RGd6X5L3RqU8VqZ3Y2C2k6E4fB1MQ8N7p9d2R9c1c8fE","label":"Vault account","origin":"sparrow-vault-export"}
```

Importing this into Specter Desktop:

```
$ specter labels import sparrow-labels.jsonl --strategy merge
parsed 8 labels (5 known refs, 3 new) — wrote to ~/.specter/labels.json
```

Round-trip verification:

```bash
$ specter labels export specter-labels.jsonl
$ diff <(sort sparrow-labels.jsonl) <(sort specter-labels.jsonl)
# expected: empty diff if loss-free
```

Encrypted variant (age):

```bash
age -r age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p \
    sparrow-labels.jsonl > sparrow-labels.jsonl.age
```

## Common pitfalls

- Using JSON arrays instead of JSON Lines. A `[ {...}, {...} ]` file is NOT BIP329; consumers expecting line-oriented input will fail on the first character.
- CRLF line endings on Windows exporters. Strict parsers reject `\r\n`-terminated lines because the `\r` becomes part of the JSON parse buffer.
- Mixing addresses and outputs by stuffing `addr` labels into `output` records. The address-vs-output distinction matters: an address label survives across UTXOs at that address, while an output label is bound to one UTXO.
- Treating xpub labels as per-address. The xpub label applies to the entire account; individual addresses derived from it should each have their own `addr` row.
- Stripping non-ASCII in transit. Labels with emojis or CJK characters MUST round-trip byte-for-byte; fixing this almost always means switching the file pipeline to UTF-8 end-to-end.
- Leaking the file. Labels are highly correlative (they contain counterparty names, amounts, intent). Encrypt at rest with age/GPG/sops or refuse to export plaintext on shared filesystems.
- Forgetting that `spendable: false` is advisory. A determined user can still spend the UTXO via coin control; the field is a hint to coin selection, not a lock.

## References

- BIP329: https://github.com/bitcoin/bips/blob/master/bip-0329.mediawiki
- jsonlines.org: https://jsonlines.org/
- Sparrow labels source: https://github.com/sparrowwallet/sparrow/tree/master/src/main/java/com/sparrowwallet/sparrow/io/bip329
- Specter Desktop label support: https://github.com/cryptoadvance/specter-desktop
- BlueWallet partial impl: https://github.com/BlueWallet/BlueWallet
