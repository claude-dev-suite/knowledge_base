# Multisig Patterns - Bare, P2SH, P2WSH, Tapscript, MuSig2

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/scripts`.
> Canonical source: https://github.com/bitcoin/bips/blob/master/bip-0011.mediawiki and BIP141, BIP342, BIP327
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/scripts/SKILL.md

## Concept

"Multisig" in Bitcoin spans five distinct on-chain shapes, each with different size, privacy, and limits. Choosing wrong inflates fees, leaks signer count, or breaks limits at signing time. This article compares them side-by-side at the byte level.

## Walkthrough / mechanics

### 1. Bare multisig (BIP11) - obsolete

```
scriptPubKey: OP_M <P1> <P2> ... <PN> OP_N OP_CHECKMULTISIG
```

- Output is large and visible (entire pubkey set on chain at funding time and at spend).
- Standardness: `M, N <= 3` for relay; consensus allows up to N=20 (SCRIPT_NUM limits).
- Spend: `OP_0 <sig1> <sig2> ... <sigM>` (the leading OP_0 is the OP_CHECKMULTISIG off-by-one bug, immortalized).

Best avoided.

### 2. P2SH-multisig (BIP16, address starts with `3`)

```
scriptPubKey: OP_HASH160 <RIPEMD160(SHA256(redeemScript))> OP_EQUAL
redeemScript: OP_M <P1> ... <PN> OP_N OP_CHECKMULTISIG
spend (scriptSig): OP_0 <sig1> ... <sigM> <push redeemScript>
```

- `M, N <= 15` (limited by `MAX_SCRIPT_ELEMENT_SIZE = 520` bytes for the redeemScript push).
- Output is small (23 bytes) and identical-looking for any P2SH content - hides the multisig structure until spend.
- Sigs and redeemScript are in scriptSig - increases legacy weight.

### 3. P2WSH-multisig (BIP141, native segwit, `bc1q...`)

```
scriptPubKey: OP_0 <SHA256(witnessScript)>          (34 bytes)
witnessScript: OP_M <P1> ... <PN> OP_N OP_CHECKMULTISIG
witness: <empty> <sig1> ... <sigM> <witnessScript>
```

- `M, N <= 67` approximately, bounded only by 520-byte witness item; in practice 15-of-15 is comfortable.
- Witness discount: 1 weight unit per byte instead of 4 - significant fee savings.
- Same opcode set as P2SH; uses BIP143 sighash with amount commitment.

### 4. P2SH-P2WSH-multisig

```
scriptPubKey: OP_HASH160 <RIPEMD160(SHA256( OP_0 <SHA256(witnessScript)> ))> OP_EQUAL
scriptSig:    push of "OP_0 <SHA256(witnessScript)>"
witness:      same as native P2WSH
```

Wraps native segwit inside P2SH for legacy-only senders. Pays 23 extra scriptSig bytes per input.

### 5. Tapscript multisig (BIP342) and MuSig2 (BIP327)

Tapscript drops `OP_CHECKMULTISIG`; instead use `OP_CHECKSIGADD` chained:

```
witnessScript:
  <P1> OP_CHECKSIG
  <P2> OP_CHECKSIGADD
  ...
  <PN> OP_CHECKSIGADD
  <M> OP_NUMEQUAL
```

Behavior: each sig either adds 1 or 0 to the running total. Witness must place sigs in pubkey order, with empty bytes for missing sigs (e.g. `<sigP1> <> <sigP3> ...`).

```
witness: <emptyOrSigPN> ... <emptyOrSigP1> <leaf script> <control block>
```

For full M-of-N this is one signature per signer, no MULTISIG quirks; sigops budget consumes 50 per CHECKSIG/CHECKSIGADD evaluated.

**MuSig2** (off-chain key aggregation):
```
P_agg = MuSig2.KeyAgg([P1, ..., PN])
scriptPubKey: OP_1 <taptweak(P_agg, R)>            (P2TR)
witness:      <single 64-byte Schnorr signature>
```

- 1 on-chain signature regardless of N.
- Signers run a 2-round interactive protocol off-chain to produce the aggregate.
- Indistinguishable from single-sig on-chain - maximum privacy.
- Falls back to taproot script-path (multisig leaf) on signer absence.

### Size comparison (3-of-5)

| Scheme | Output size | Spend witness/scriptSig weight |
|--------|-------------|-------------------------------|
| Bare multisig | ~211 B | non-standard |
| P2SH | 23 B | ~265 B in scriptSig (4x WU) |
| P2WSH | 34 B | ~265 B in witness (1x WU each) |
| P2SH-P2WSH | 23 B | scriptSig 35 B + witness 265 |
| Tapscript script-path | 34 B | ~290 B + 33+32x depth control block |
| Taproot key-path (MuSig2) | 34 B | 64 B witness |

## Worked example

5-of-7 P2WSH:

```
witnessScript = 55 21 <P1> 21 <P2> 21 <P3> 21 <P4> 21 <P5> 21 <P6> 21 <P7> 57 ae
                ^M=5                                                       ^N=7 ^OP_CHECKMULTISIG
length = 1 + 7*(1+33) + 1 + 1 = 241 bytes
SHA256 -> 32 byte program -> scriptPubKey = 0020 <hash>  (34 bytes)
```

Spend:
```
witness = [
  "",              # CHECKMULTISIG dummy
  <sig_1>,         # 71-72 bytes (DER + sighash byte)
  <sig_2>,
  <sig_3>,
  <sig_4>,
  <sig_5>,
  <witnessScript>  # 241 bytes
]
```

## Common bugs / anti-patterns

- Using `OP_CHECKMULTISIG` in Tapscript: it is `OP_SUCCESS` there; tx becomes anyone-can-spend (script-path).
- Forgetting the `OP_0` dummy in CHECKMULTISIG spends; no error message - script just fails.
- Sorting public keys after generating address but signing with original order; signatures become invalid.
- Hitting the 520-byte witness item limit for N around 16-of-16 in P2WSH; switch to taproot.
- Using bare multisig today; relayed only with private mining.
- Mixing compressed and uncompressed pubkeys: address (and witness) must use exactly the same encoding used at funding time.
- For tapscript M-of-N: forgetting the per-signer empty-byte placeholder, putting them in wrong order.
- Treating MuSig2 like classical multisig: NEVER reuse nonces; an attacker can compute the private key from two signatures sharing a nonce.

## References

- BIP11 (M-of-N): https://github.com/bitcoin/bips/blob/master/bip-0011.mediawiki
- BIP16 (P2SH): https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki
- BIP141 (witness): https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki
- BIP342 (Tapscript): https://github.com/bitcoin/bips/blob/master/bip-0342.mediawiki
- BIP327 (MuSig2): https://github.com/bitcoin/bips/blob/master/bip-0327.mediawiki
