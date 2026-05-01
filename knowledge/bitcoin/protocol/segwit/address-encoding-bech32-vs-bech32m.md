# Bech32 vs Bech32m Address Encoding - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/segwit`.
> Canonical source: BIP173 (Bech32), BIP350 (Bech32m), BIP141 (witness program)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/segwit/SKILL.md

## Concept

Bech32 (BIP173) and Bech32m (BIP350) share the same alphabet, HRP rules,
and 6-character checksum format, but differ in **one constant** xored
into the polymod final step. The split exists because BIP173 has a
length-extension flaw: appending a `q` (zero) character to a valid
Bech32 string produces another valid Bech32 string when the original
was at the boundary of a 1-byte data field. SegWit v0 happens to be
unaffected (only 20- or 32-byte programs), but generic v1+ programs
of arbitrary length are not safe under BIP173. BIP350 fixes this.

## Walkthrough / mechanics

**Shared alphabet (32 chars):** `qpzry9x8gf2tvdw0s3jn54khce6mua7l`.
Excludes `1`, `b`, `i`, `o` to reduce visual ambiguity.

**Format:** `<HRP> 1 <data> <6-char checksum>`. HRP is `bc` (mainnet),
`tb` (testnet), `bcrt` (regtest), `tbs` (signet on some forks). The
separator `1` is ASCII 0x31; everything after the **last** `1` is the
data + checksum portion.

**Checksum constants:**
```
Bech32  (BIP173): polymod(values) == 1
Bech32m (BIP350): polymod(values) == 0x2bc830a3
```

The polymod itself is a BCH code over GF(32) with generator polynomial
`{0x3b6a57b2, 0x26508e6d, 0x1ea119fa, 0x3d4233dd, 0x2a1462b3}`.

**Selection rule for SegWit (BIP350 section "Addresses"):**
- Witness version `0` -> Bech32 constant.
- Witness version `1..16` -> Bech32m constant.

So `bc1q...` (v0, P2WPKH/P2WSH) uses Bech32, while `bc1p...` (v1,
P2TR) uses Bech32m. The first character after the `1` (`q` = 0,
`p` = 1) leaks the witness version directly.

## Worked example

Encode a P2WPKH for hash160 `751e76e8199196d454941c45d1b3a323f1433bd6`:

```
hrp     = "bc"
witver  = 0
witprog = 751e76e8199196d454941c45d1b3a323f1433bd6   # 20 bytes

# 1) convert 8-bit groups to 5-bit groups
data5 = convertbits(witprog, 8, 5, pad=True)
      = [0e 0e 0f 12 17 13 18 13 06 16 11 03 14 12 03 03 0a 04 11 17 06 09 03 17 18 09 0a 14 1e 16 06 0b]

# 2) prepend witver
values = [0] + data5

# 3) compute checksum over hrp_expand(hrp) + values + [0]*6 with const=1
checksum = bech32_create_checksum("bc", values, const=1)
# 4) join
addr = "bc1" + base32_encode(values + checksum)
     = "bc1qw508d6qejxtdg4y5r3zarvary0c5xw7kv8f3t4"
```

For Taproot, given x-only output key `Q_x = a3c1c...` (32 bytes):

```
hrp     = "bc"
witver  = 1
witprog = Q_x                                   # 32 bytes
addr    = "bc1p" + base32_encode([1] + convertbits(Q_x, 8, 5, True) + checksum_BECH32M)
```

## Common bugs / pitfalls

1. **Using Bech32 constant for Taproot** -> address validates locally
   but every other wallet rejects it (length-extension hardening).
2. **Mixed case:** spec mandates the full address be lower OR upper,
   never mixed. `Bc1q...` is invalid. QR codes use upper case for
   density; wallets must lowercase before parsing.
3. **HRP confusion:** `tb1...` is testnet AND signet by default. Some
   wallets show signet addresses as `tbs1...` but that is a custom
   HRP, not standardized.
4. **convertbits padding:** when going 8->5, padding adds zero bits;
   when going 5->8 to decode, padding bits MUST be zero or the address
   is invalid. Many libraries silently accept non-zero padding.
5. **Length validation:** v0 must be 20 or 32 bytes; lengths in
   `[2, 40]` are otherwise allowed by BIP141 but only v0 is
   consensus-defined. Wallets should reject unknown-length v0.
6. **First-byte semantics:** `bc1q...` is v0, `bc1p...` is v1, but
   `bc1z...` would be v2 (anyone-can-spend until soft fork) - never
   send funds to such addresses.

## References

- BIP173: https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki
- BIP350: https://github.com/bitcoin/bips/blob/master/bip-0350.mediawiki
- BIP141: https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki
- Reference implementation: https://github.com/sipa/bech32
- rust-bitcoin `bech32` crate: https://docs.rs/bech32
