# SLIP-132 Extended Key Version Bytes - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/cryptography/bip32`.
> Canonical source: SLIP-0132 (https://github.com/satoshilabs/slips/blob/master/slip-0132.md).
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/cryptography/bip32/SKILL.md

## Concept

The 4-byte version prefix of an extended key is metadata: it tells a
wallet how to display the key (the human prefix `xpub`/`ypub`/`zpub`),
and it hints at the intended script type for derived addresses. SLIP-132
catalogues the prefixes Trezor and the broader ecosystem adopted before
descriptor-based wallets became standard. They remain a constant source
of integration bugs because they are policy hints encoded as bytes -
applications routinely treat them as authoritative when they should not.

## Walkthrough / mechanics

### Wire layout reminder

A serialized BIP32 extended key is exactly 78 bytes pre-Base58Check:

```
offset  size  field
0       4     version
4       1     depth
5       4     parent fingerprint
9       4     child number
13      32    chain code
45      33    key data  (0x00 || privkey  for xprv;  compressed pubkey for xpub)
```

Base58Check adds a 4-byte checksum and produces the 111-character text.

The version bytes determine the human prefix per the Base58 alphabet:

```
0x0488B21E (xpub mainnet, unbounded script type or P2PKH per BIP44)
0x0488ADE4 (xprv mainnet)
0x049D7CB2 (ypub mainnet, BIP49 P2SH-wrapped P2WPKH)
0x049D7878 (yprv mainnet)
0x04B24746 (zpub mainnet, BIP84 P2WPKH native segwit)
0x04B2430C (zprv mainnet)
0x0295B43F (Ypub mainnet multisig P2SH-wrapped P2WSH)
0x02AA7ED3 (Zpub mainnet multisig P2WSH)

0x043587CF (tpub testnet)
0x04358394 (tprv testnet)
0x044A5262 (upub testnet, BIP49)
0x045F1CF6 (vpub testnet, BIP84)
```

The Base58 alphabet maps the leading byte to the first character; that
mapping is why `0x04 0x88` produces `x`, `0x04 0x9D` produces `y`, etc.

### What SLIP-132 actually says

SLIP-132 explicitly documents these as a **convention**: the BIP32 standard
itself only specifies `xpub`/`xprv` and their testnet variants. The
y/z/Y/Z prefixes were added by Trezor and adopted by Electrum, Sparrow,
and others. Modern descriptor wallets (Bitcoin Core 0.21+, BDK) bypass
this layer entirely - they accept `xpub` and read the script type from
the descriptor wrapper (`wpkh(xpub.../84'/0'/0'/0/*)`).

### Why the prefix is metadata, not authority

Two `xpub` strings with different prefixes can encode the **same** keys
algorithmically:

```
KEY_BYTES = [version (4)] [depth] [fp] [child_num] [c] [pubkey]
                ^^^^^^^^^
                only this changes between xpub and zpub
```

The chain code, public key, depth, etc. are identical. A wallet that
re-serializes an xpub with the zpub version prefix produces a different
Base58 string but the same underlying key. This means:

- A consumer that "trusts" the prefix to choose the script type can be
  mismatched with the original wallet's intent.
- Two wallets sharing an xpub via different prefixes will compute the
  same addresses if they agree on the script type out-of-band, but
  different addresses if they read the prefix.

The *correct* posture: treat version bytes as a **display hint only**.
Always carry script type in a descriptor or a separate field.

## Worked example

Take a Trezor BIP84 export:

```
zpub6r1FCfvKE...(111 chars)
```

Base58Check decode:

```
version  = 0x04B24746          (zpub mainnet)
depth    = 0x03                (m/84'/0'/0' is depth 3)
fp       = AB CD EF 12          (parent fingerprint of m/84'/0')
child#   = 0x80000000           (account 0', high bit set = hardened)
c        = 32 bytes
pubkey   = 33 bytes (02 || x)
```

Now re-serialize with the xpub version:

```
version  = 0x0488B21E
depth    = 0x03                 (unchanged)
fp       = AB CD EF 12          (unchanged)
child#   = 0x80000000           (unchanged)
c        = same 32 bytes
pubkey   = same 33 bytes

Result: a valid xpub6... string, semantically identical for derivation.
```

A Bitcoin Core descriptor that takes this xpub and wraps it as
`wpkh([abcdef12/84'/0'/0']xpub6...)` produces the *same* P2WPKH addresses
as the original zpub. Conversely, wrapping it as `pkh(...)` produces
P2PKH addresses - which is wrong for funds the user actually held in
P2WPKH. The version byte was right; the descriptor was wrong; both must
agree.

### Conversion table for tooling

Common one-line conversions:

```
xpub <-> ypub <-> zpub:    edit only the first 4 bytes after Base58Check
                            decode, recompute checksum, re-encode

testnet xpub (tpub) <-> mainnet xpub:   change version 0x043587CF <-> 0x0488B21E
                                         (caution: testnet keys derived from
                                          a different seed are still on testnet
                                          regardless of prefix - keys are not
                                          coin-aware)
```

Tools: `slip132` Python lib, `descriptors.dev`, `https://jlopp.github.io/xpub-converter/`.

## Common pitfalls

- **Treating zpub as "BIP84-only"**: a zpub is just a key + chain code
  with a metadata hint. Funds at non-BIP84 paths are still mathematically
  derivable, the wallet just chooses not to show them.
- **Round-tripping prefixes**: importing zpub into a wallet that only
  accepts xpub, then re-exporting as xpub, then importing into a wallet
  that derives BIP44 addresses by default. Funds appear "lost" - actually
  at different addresses. Always carry derivation path with the key.
- **Confusing fingerprint and version**: fingerprint is 4 bytes of
  HASH160(parent pubkey), version is fixed-per-prefix. Both are 4 bytes
  and adjacent in the serialized form - copy-paste bugs are common.
- **Mainnet/testnet leakage**: hardcoding `0x0488B21E` in a wallet that
  also handles testnet causes silent wrong-network behavior. Use a
  network parameter object.
- **PSBT and version bytes**: BIP174 PSBTs carry global xpubs in field
  `PSBT_GLOBAL_XPUB`. Some signers reject if the version is not exactly
  `0x0488B21E` (BIP32 default). Use canonical `xpub` for PSBT, no
  exceptions.
- **Multisig and capital-letter prefixes**: `Ypub`/`Zpub` (capital Y/Z)
  signal multisig variants. Confusing them with `ypub`/`zpub` in single-
  sig wallets produces address mismatches across cosigners.

## References

- SLIP-0132: https://github.com/satoshilabs/slips/blob/master/slip-0132.md
- BIP32: https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki
- BIP49 / BIP84 / BIP86: derivation paths and intended script types.
- BIP380 (Output Descriptors): the modern alternative to prefix-based
  script-type signaling.
- xpub-converter source: https://github.com/jlopp/xpub-converter
