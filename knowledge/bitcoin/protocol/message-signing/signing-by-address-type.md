# Message Signing by Address Type - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/message-signing`.
> Canonical source: BIP137, BIP322
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/message-signing/SKILL.md

## Concept

Different Bitcoin address types support different signing schemes,
sighash conventions, and witness encodings. This article catalogs the
correct signing approach per address type, the witness/scriptSig
shape produced, and the verifier checks that distinguish a valid
signature from a malformed one.

The skill notes which schemes apply to which addresses; this article
goes byte-level into each.

## Walkthrough / mechanics

**Address type compatibility matrix:**

| Address | BIP137 | BIP322 simple | BIP322 full |
|---------|--------|---------------|-------------|
| P2PKH `1...` | yes (canonical) | no | yes |
| P2SH `3...` (script) | no | depends on inner | yes |
| P2SH-P2WPKH `3...` (wrapped segwit) | no | yes (inner is segwit) | yes |
| P2WPKH `bc1q...` 20-byte | extension dialect | yes | yes |
| P2WSH `bc1q...` 32-byte | no | yes (witness defined) | yes |
| P2TR `bc1p...` | no | yes (Schnorr witness) | yes |

For P2SH wrapping a non-witness script (rare, like multisig in P2SH),
only full form works.

**Witness shape per address type (BIP322 simple):**

```
P2WPKH:    [<sig>, <pubkey>]                   # 2 items
P2WSH:     [<arg1>, <arg2>, ..., witness_script]   # depends on script
P2TR keypath:  [<schnorr_sig_64_or_65>]            # 1 item
P2TR scriptpath: [<args>, leaf_script, control_block]  # 3+ items
```

**Sighash type per address (recommended):**

| Address | sighash | Sig length |
|---------|---------|------------|
| P2PKH | SIGHASH_ALL (0x01) | 71-72 bytes ECDSA + 1 byte |
| P2WPKH | SIGHASH_ALL | 71-72 + 1 |
| P2WSH | SIGHASH_ALL | per script |
| P2TR (BIP322) | SIGHASH_DEFAULT (0x00) | 64 bytes Schnorr |

## Worked example

**P2PKH (BIP137 canonical):**

```python
def sign_p2pkh(privkey, message):
    msg_bytes = message.encode()
    preamble = compactsize(len("Bitcoin Signed Message:\n"))
    preamble += b"Bitcoin Signed Message:\n"
    preamble += compactsize(len(msg_bytes))
    preamble += msg_bytes
    digest = sha256d(preamble)
    sig, recid = ecdsa_sign_recoverable(privkey, digest)
    header = bytes([27 + recid + (4 if privkey.compressed else 0)])
    return base64.b64encode(header + sig).decode()
```

Verifier:
```python
def verify_p2pkh(addr, message, sig_b64):
    raw = base64.b64decode(sig_b64)
    header, rs = raw[0], raw[1:]
    recid = (header - 27) & 3
    compressed = (header - 27) & 4
    digest = sha256d(preamble(message))
    pubkey = ecdsa_recover(digest, rs, recid)
    derived = p2pkh_address(pubkey, compressed)
    return derived == addr
```

**P2WPKH (BIP322 simple):**

```python
def sign_p2wpkh(privkey, message, addr):
    spk = bech32_decode_to_spk(addr)             # OP_0 PUSH20 <h160>
    msg_hash = tagged("BIP0322-signed-message", message.encode())
    to_spend = build_to_spend(spk, msg_hash)
    to_sign  = build_to_sign(to_spend.txid())

    sighash = bip143_sighash(to_sign, 0, 0, spk, 0x01)
    sig = ecdsa_sign(privkey, sighash) + b"\x01"
    witness = [sig, privkey.pubkey().sec()]

    return base64.b64encode(serialize_witness(witness)).decode()
```

**P2TR keypath (BIP322 simple):**

```python
def sign_p2tr_key(privkey, message, addr):
    spk = bech32m_decode_to_spk(addr)            # OP_1 PUSH32 <Q_x>
    msg_hash = tagged("BIP0322-signed-message", message.encode())
    to_spend = build_to_spend(spk, msg_hash)
    to_sign  = build_to_sign(to_spend.txid())

    # Taproot needs prevout amount + spk for sighash
    prevouts = [(0, spk)]
    digest = bip341_sighash(to_sign, 0, prevouts, SIGHASH_DEFAULT, ext_flag=0)

    # Tweak the privkey for keypath
    p_tweaked = tap_tweak_seckey(privkey, b"")
    sig = schnorr_sign(p_tweaked, digest)        # 64 bytes
    witness = [sig]

    return base64.b64encode(serialize_witness(witness)).decode()
```

**P2WSH multisig (2-of-3) BIP322 simple:**

```python
def sign_p2wsh_multi(privkeys_used, message, addr, witness_script):
    spk = bech32_decode_to_spk(addr)             # OP_0 PUSH32 <sha256(witness_script)>
    msg_hash = tagged("BIP0322-signed-message", message.encode())
    to_spend = build_to_spend(spk, msg_hash)
    to_sign  = build_to_sign(to_spend.txid())

    sigs = []
    for sk in privkeys_used:
        sighash = bip143_sighash(to_sign, 0, 0, witness_script, 0x01)
        sig = ecdsa_sign(sk, sighash) + b"\x01"
        sigs.append(sig)

    # CHECKMULTISIG bug: leading empty push
    witness = [b""] + sigs + [witness_script]
    return base64.b64encode(serialize_witness(witness)).decode()
```

## Common bugs / pitfalls

1. **Mixing BIP137 header with BIP322 verifier.** A BIP137 base64 has
   a 1-byte header followed by 64 bytes of (r||s). A BIP322 simple
   has a varint-prefixed witness stack. Length alone (BIP137 = ~88
   chars, BIP322 minimal P2WPKH = ~140 chars) lets you discriminate.
2. **Dishonest "address-type hint" header in BIP137.** A signer can
   set the hint to claim a SegWit address while actually deriving
   from a P2PKH. Verifier must independently re-derive the address
   and ignore the hint.
3. **Wrong scriptSig encoding for to_sign legacy.** Full-form BIP322
   for P2PKH must put a real scriptSig (sig + pubkey) on the
   to_sign input, not a witness.
4. **Taproot SIGHASH_ALL byte trailing.** Some libraries always
   append 0x01; verifiers expecting 64-byte sig (DEFAULT) reject the
   65-byte form. Spec allows both for ALL semantics; for cleanest
   verifiability use 64-byte DEFAULT.
5. **P2SH-P2WPKH simple form.** The address is P2SH but the actual
   script being satisfied is the inner P2WPKH. Witness has 2 items
   (sig, pubkey). Full form additionally needs scriptSig with
   PUSH(redeemScript). Some libs only emit witness, breaking
   verification.
6. **Multisig signing order.** Sigs in the witness stack must follow
   the pubkey order in the witness_script (or sortedmulti's sorted
   order), not signer order.
7. **OP_RETURN output value.** to_sign.outputs[0].value must be 0;
   non-zero produces a different sighash.

## References

- BIP137: https://github.com/bitcoin/bips/blob/master/bip-0137.mediawiki
- BIP322: https://github.com/bitcoin/bips/blob/master/bip-0322.mediawiki
- BIP143: https://github.com/bitcoin/bips/blob/master/bip-0143.mediawiki
- BIP341: https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki
- bip322-js: https://github.com/ACken2/bip322-js
