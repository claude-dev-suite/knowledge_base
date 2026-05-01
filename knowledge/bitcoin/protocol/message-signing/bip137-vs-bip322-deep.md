# BIP137 vs BIP322 - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/message-signing`.
> Canonical source: BIP137 (formalization), BIP322
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/message-signing/SKILL.md

## Concept

BIP137 and BIP322 both produce "I own this address" attestations but
use fundamentally different cryptographic constructions. The skill
sketches each; this article presents a side-by-side breakdown of the
prefix conventions, recovery semantics, encoding, and the
interoperability gap that still trips up exchange-withdraw flows in
2025-2026.

## Walkthrough / mechanics

**BIP137: recoverable ECDSA over a fixed prefix.**

```
prefix          = "Bitcoin Signed Message:\n"     # 24 bytes
preamble_bytes  = varint(len(prefix)) || prefix || varint(len(msg)) || msg
digest          = SHA256d(preamble_bytes)         # 32 bytes
sig             = ecdsa_sign_recoverable(privkey, digest)
                  -> (r, s, recid, addr_type_hint)
header_byte     = 27 + recid + (4 if compressed) + (8 if P2WPKH-uncompressed)
                  ...
signature       = base64(header_byte || r_32 || s_32)
                  -> 65 bytes raw -> ~88 chars base64
```

The header byte encodes:
- recid (0..3): which y-parity to recover
- compressed flag: pubkey was compressed
- "address type hint" (BIP137 extensions): bits to indicate the
  expected scriptPubKey type (P2PKH, P2SH-P2WPKH, P2WPKH, etc.)

Verification flow:

```
1. Decode base64 -> 65 bytes.
2. Recover pubkey from (r, s, recid, digest).
3. Derive address from pubkey using the address-type hint.
4. Compare to claimed address.
```

The original BIP137 only supported P2PKH; "Trezor flavor" extends
header bytes to encode SegWit address types. Bitcoin Core sticks to
strict P2PKH.

**BIP322: virtual-tx based.**

Construct two transactions:

```
to_spend (virtual; never broadcast):
  vin = [{
    prevout    = (txid=0xff..ff, vout=0xffffffff),  # null prev outpoint
    scriptSig  = OP_0 || PUSH32 hash_of_message,
    sequence   = 0,
  }]
  vout = [{
    value      = 0,
    scriptPubKey = scriptPubKey_of_address,         # the addr being signed for
  }]

to_sign (virtual; never broadcast):
  vin = [{
    prevout    = (txid=hash_of(to_spend), vout=0),
    scriptSig  = empty,
    sequence   = 0,
  }]
  vout = [{
    value      = 0,
    scriptPubKey = OP_RETURN,
  }]
```

Then SIGN to_sign as if its input were spending to_spend's output.
The signature is whatever scriptSig+witness would satisfy that script.

```
hash_of_message = TaggedHash("BIP0322-signed-message", message)
```

Verification flow:

```
1. Reconstruct to_spend, to_sign from address + message + signature.
2. Apply signature to to_sign's input.
3. Run script verification of to_sign.input[0] against to_spend.output[0].
4. Pass = ownership proven.
```

Output forms:
- **Simple BIP322**: base64 of just the witness stack. Works for
  SegWit addresses (P2WPKH, P2WSH, P2TR).
- **Full BIP322**: base64 of a PSBT-like blob. Required for
  legacy P2PKH/P2SH or anything with non-witness script.

## Worked example

**BIP137 signing P2PKH address `1...`:**

```python
import hashlib, base64
from ecdsa import SigningKey, SECP256k1
from ecdsa.util import sigencode_string

def bip137_sign(privkey_wif, message):
    sk = SigningKey.from_secret_exponent(decode_wif(privkey_wif), curve=SECP256k1)
    prefix = b"Bitcoin Signed Message:\n"
    preamble = compactsize(len(prefix)) + prefix + compactsize(len(message)) + message.encode()
    digest = hashlib.sha256(hashlib.sha256(preamble).digest()).digest()
    sig, recid = sk.sign_recoverable(digest)
    header = bytes([27 + recid + 4])    # +4 because compressed
    return base64.b64encode(header + sig).decode()
```

**BIP322 simple form for P2WPKH `bc1q...`:**

```python
def bip322_sign_simple(privkey, message, p2wpkh_addr):
    spk = address_to_scriptPubKey(p2wpkh_addr)            # OP_0 PUSH20 <h160>
    msg_hash = tagged_hash("BIP0322-signed-message", message.encode())
    to_spend = build_to_spend(spk, msg_hash)
    to_sign  = build_to_sign(to_spend.txid)

    # sign as if spending to_spend.output[0]
    digest = bip143_sighash(to_sign, 0, to_spend.outputs[0].value, spk, SIGHASH_ALL)
    sig = ecdsa_sign(privkey, digest) + b"\x01"           # SIGHASH_ALL byte
    pubkey = privkey.pubkey
    witness = [sig, pubkey.encode()]
    return base64.b64encode(serialize_witness(witness)).decode()
```

For Taproot, the digest uses BIP341 and signs with Schnorr (64-byte
sig + optional sighash byte).

**Verification of the Taproot case:**

```python
def bip322_verify_taproot(addr, message, sig_b64):
    spk = address_to_scriptPubKey(addr)                  # OP_1 PUSH32 <Q_x>
    msg_hash = tagged_hash("BIP0322-signed-message", message.encode())
    to_spend = build_to_spend(spk, msg_hash)
    to_sign  = build_to_sign(to_spend.txid)
    witness = parse_witness(base64.b64decode(sig_b64))

    digest = bip341_sighash(to_sign, 0, [(to_spend.outputs[0].value, spk)],
                            SIGHASH_DEFAULT)
    Q_x = spk[2:]
    return schnorr_verify(Q_x, digest, witness[0])
```

## Common bugs / pitfalls

1. **BIP137 prefix variants.** "Trezor variant" includes the prefix
   inside `varint(len(prefix))` count; some libraries omit. Sigs from
   one library don't verify with another.
2. **Header byte interpretation.** Header byte semantics vary across
   BIP137 dialects. Always verify with the exact wallet that signed.
3. **Using P2PKH key for SegWit address claim.** Some wallets derive
   a P2PKH from the same pubkey as the SegWit address and BIP137-sign.
   The claim is "I have the privkey" but doesn't authenticate the
   actual address.
4. **BIP322 sighash mismatch.** Simple form expects SIGHASH_ALL for
   non-Taproot, SIGHASH_DEFAULT for Taproot. Some libraries default
   to ALL (0x01) for Taproot too, producing 65-byte sigs that some
   verifiers reject.
5. **Empty message handling.** BIP322 hashes via tagged hash; an
   empty message still produces a deterministic 32-byte hash. BIP137
   uses `varint(0) || ""` which some libraries reject.
6. **Multisig BIP322.** Requires combining partial sigs from each
   cosigner using a custom flow (similar to PSBT). Most wallets don't
   yet implement.
7. **Address case.** Bech32 addresses are case-insensitive but the
   scriptPubKey is case-sensitive (lowercase canonical form). Verify
   with normalized lowercase address.

## References

- BIP137: https://github.com/bitcoin/bips/blob/master/bip-0137.mediawiki
- BIP322: https://github.com/bitcoin/bips/blob/master/bip-0322.mediawiki
- bitcoinjs-lib BIP322: https://github.com/ACken2/bip322-js
- rust-bitcoin signed-message: https://docs.rs/bitcoin
- Sparrow Wallet message signing UX
