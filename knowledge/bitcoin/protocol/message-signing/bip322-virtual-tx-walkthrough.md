# BIP322 Virtual Transaction Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/message-signing`.
> Canonical source: BIP322
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/message-signing/SKILL.md

## Concept

BIP322 reduces "sign a message with a Bitcoin address" to "sign a
virtual transaction that spends a virtual output". The two virtual
transactions, `to_spend` and `to_sign`, are constructed
deterministically from the address and message; they are never
broadcast. The witness/scriptSig of `to_sign` is the signature.

This article walks the byte-level construction of both txs, the
sighash computation that the signer actually performs, and the
verifier's reconstruction algorithm with concrete fixtures.

## Walkthrough / mechanics

**Step 1: tagged hash of message.**

```
msg_hash = TaggedHash("BIP0322-signed-message", message)
         = SHA256(SHA256("BIP0322-signed-message") || SHA256("BIP0322-signed-message") || message)
```

**Step 2: build to_spend.**

```
to_spend.version    = 0
to_spend.locktime   = 0
to_spend.inputs[0]:
  prevout.txid    = 0x0000000000000000000000000000000000000000000000000000000000000000
  prevout.vout    = 0xffffffff                # null outpoint convention
  scriptSig       = OP_0 || OP_PUSHBYTES_32 || msg_hash
  sequence        = 0
to_spend.outputs[0]:
  value           = 0
  scriptPubKey    = scriptPubKey of the signing address
```

`to_spend` has scriptSig referencing the message hash; this is what
makes the message commit through the script execution.

**Step 3: build to_sign.**

```
to_sign.version     = 0       # later revisions may use 2
to_sign.locktime    = 0
to_sign.inputs[0]:
  prevout.txid    = txid(to_spend)            # 32 bytes
  prevout.vout    = 0
  scriptSig       = (will be filled by signer)
  sequence        = 0
to_sign.outputs[0]:
  value           = 0
  scriptPubKey    = OP_RETURN
```

**Step 4: sign to_sign as if spending to_spend.outputs[0].**

For SegWit/Taproot, use the appropriate sighash:

```
P2WPKH:  BIP143 sighash with SIGHASH_ALL (or others)
P2TR:    BIP341 sighash with SIGHASH_DEFAULT
```

For P2PKH, use legacy sighash. The signer fills in
`scriptSig` (legacy) or `witness` (segwit/taproot) of `to_sign`.

**Step 5: encode signature.**

- **Simple form (preferred for SegWit/Taproot):** base64 of the
  serialized witness stack only.
  ```
  encoded = base64(varint(num_witnesses) || for each w: varint(len) || w)
  ```
- **Full form (required for P2SH or non-witness):** base64 of the
  whole `to_sign` transaction.

## Worked example

**Sign with P2WPKH `bc1qw508d6qejxtdg4y5r3zarvary0c5xw7kv8f3t4`:**

Address decodes to scriptPubKey `0014751e76e8199196d454941c45d1b3a323f1433bd6`.
Message: `"hello world"`.

```python
import hashlib, base64

def tagged(tag, msg):
    h = hashlib.sha256(tag.encode()).digest()
    return hashlib.sha256(h + h + msg).digest()

message = b"hello world"
msg_hash = tagged("BIP0322-signed-message", message)
# msg_hash = f7c121...  (32 bytes)

# to_spend.scriptSig = 00 20 <msg_hash>
to_spend_scriptsig = b"\x00\x20" + msg_hash

# build to_spend serialization
to_spend = bytes.fromhex("00000000")        # version 0
to_spend += b"\x01"                         # 1 input
to_spend += b"\x00" * 32                    # null prevout txid
to_spend += b"\xff\xff\xff\xff"             # vout = 0xffffffff
to_spend += bytes([len(to_spend_scriptsig)]) + to_spend_scriptsig
to_spend += b"\x00\x00\x00\x00"             # sequence
to_spend += b"\x01"                         # 1 output
to_spend += b"\x00" * 8                     # value = 0
spk = bytes.fromhex("0014751e76e8199196d454941c45d1b3a323f1433bd6")
to_spend += bytes([len(spk)]) + spk
to_spend += b"\x00\x00\x00\x00"             # locktime

txid_to_spend = hashlib.sha256(hashlib.sha256(to_spend).digest()).digest()[::-1]
```

Now build `to_sign`:

```python
to_sign = bytes.fromhex("00000000")         # version
to_sign += b"\x01"                          # 1 input
to_sign += txid_to_spend[::-1]              # internal byte order
to_sign += b"\x00" * 4                      # vout = 0
to_sign += b"\x00"                          # empty scriptSig
to_sign += b"\x00" * 4                      # sequence
to_sign += b"\x01"                          # 1 output
to_sign += b"\x00" * 8                      # value
to_sign += b"\x01\x6a"                      # scriptPubKey OP_RETURN
to_sign += b"\x00" * 4                      # locktime
```

Compute BIP143 sighash for input 0:

```python
sighash = bip143_sighash(
    to_sign,
    input_index=0,
    spent_amount=0,
    spent_script=spk,                       # scriptPubKey of to_spend.outputs[0]
    sighash_type=0x01,                       # SIGHASH_ALL
)
sig = ecdsa_sign(privkey, sighash) + b"\x01"
pubkey = privkey.pubkey                     # 33-byte compressed
witness = [sig, pubkey]

simple = b""
simple += bytes([len(witness)])
for w in witness:
    simple += compactsize(len(w)) + w
encoded = base64.b64encode(simple).decode()
```

`encoded` is the BIP322 simple signature, ~140 chars base64.

## Common bugs / pitfalls

1. **Forgetting to use TaggedHash.** Plain SHA256 of message produces
   a different hash and thus a different `to_spend`. Verifiers using
   the spec construction will reject.
2. **scriptSig OP_0 vs OP_PUSHBYTES_0.** `OP_0` is the byte `0x00`,
   which is also OP_PUSHBYTES_0 in some parsers. Both are equivalent
   in script semantics. The serialization is one byte either way.
3. **Wrong sighash for Taproot.** Use BIP341 with SIGHASH_DEFAULT
   (0x00) for 64-byte sigs. Using BIP143 sighash for a Taproot input
   produces a wrong digest.
4. **Spending value confusion.** `to_spend.outputs[0].value = 0`.
   When computing BIP143 sighash for `to_sign`, the spent_amount
   must also be 0. If you reuse a non-zero amount, sighash mismatches.
5. **Witness vs PSBT serialization.** Simple form is the witness
   stack alone. Some libraries serialize a fully signed transaction
   into the simple base64; this is incorrect.
6. **Multisig signing flow.** Each cosigner produces a partial
   witness. Combining is similar to PSBT but ad-hoc; BIP322 itself
   doesn't define the combiner format. Use full form with PSBT
   for multisig.
7. **Verifier accepting simple form for non-witness addresses.**
   P2PKH cannot use simple form (no witness stack). Verifiers must
   reject simple form for legacy addresses.

## References

- BIP322: https://github.com/bitcoin/bips/blob/master/bip-0322.mediawiki
- bip322-js: https://github.com/ACken2/bip322-js
- rust-bitcoin: https://docs.rs/bitcoin
- BIP143: https://github.com/bitcoin/bips/blob/master/bip-0143.mediawiki
- BIP341: https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki
