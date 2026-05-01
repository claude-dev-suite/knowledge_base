# SeedQR Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/wallets/backup`.
> Canonical source: https://github.com/SeedSigner/seedsigner/blob/dev/docs/seed_qr/README.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/wallets/backup/SKILL.md

## Concept

SeedQR encodes a BIP39 mnemonic as a QR code so an air-gapped signer (SeedSigner, Krux, Coldcard Q) can ingest the seed by camera in a fraction of a second instead of typing 12 or 24 words. The "Standard SeedQR" variant maps each word to its 4-digit decimal index in the BIP39 English wordlist; the result is a fixed-length numeric string that QR encoders fit into the dense Numeric Mode (3.33 bits per digit). The "CompactSeedQR" variant skips the wordlist round-trip and encodes the raw entropy bytes directly via QR Byte Mode, producing an even smaller code at the cost of slightly worse cross-wallet support. Both formats are plain QR (Version 1 to 5) with no proprietary metadata; any QR scanner reads them.

## Walkthrough / mechanics

Standard SeedQR encoding:

```
for word in mnemonic.split():
    idx = BIP39_ENGLISH.index(word)         # 0..2047
    s   = f"{idx:04d}"                      # always 4 chars
    out.append(s)
return "".join(out)
```

Length:

| words | output digits | QR mode    | typical version |
|------:|--------------:|------------|----------------:|
|    12 |            48 | Numeric    | V1 (21 x 21)    |
|    24 |            96 | Numeric    | V3 (29 x 29)    |

Decoding is the trivial inverse. The decoder MUST validate the BIP39 checksum; a mistyped or misread digit produces an in-range index but breaks the SHA256 checksum and the wallet rejects the seed.

CompactSeedQR encoding:

```
entropy = bip39.decode(mnemonic).entropy_bytes   # 16 or 32 bytes
qr.encode_byte_mode(entropy)                     # QR Byte Mode
```

Length: 16 or 32 raw bytes -> Version 1 (16-byte) or Version 2 (32-byte) QR. The decoder feeds the raw bytes back through `bip39.encode_from_entropy(bytes)` to regenerate the mnemonic and verify checksum.

QR error correction levels: SeedSigner defaults to "L" (~7% recovery) for the densest code; bumping to "Q" (~25%) adds resilience to scratches at the cost of a larger version. Privacy implication: a SeedQR is a plaintext seed in optical form. Photographing it through a phone camera at 1m is a known exfiltration vector; cover or laminate after creation.

## Worked example

Mnemonic (12 words, BIP39 English, well-known test vector):

```
abandon abandon abandon abandon abandon abandon abandon abandon abandon
abandon abandon about
```

Word indices:

| word     | idx  | 4-digit |
|----------|-----:|---------|
| abandon  |    0 | 0000    |
| abandon  |    0 | 0000    |
| abandon  |    0 | 0000    |
| abandon  |    0 | 0000    |
| abandon  |    0 | 0000    |
| abandon  |    0 | 0000    |
| abandon  |    0 | 0000    |
| abandon  |    0 | 0000    |
| abandon  |    0 | 0000    |
| abandon  |    0 | 0000    |
| abandon  |    0 | 0000    |
| about    |    3 | 0003    |

Standard SeedQR digit string:

```
000000000000000000000000000000000000000000000003
```

48 digits, fits a Version 1 Numeric QR (max 41 numeric chars at level Q, 41-50 at L).

A more realistic 12-word seed:

```
"praise you muffin lion enable neck grocery crash acute change host vault"
indices  -> [1369, 2056, 1170, 1041, 583, 1182, 824, 397, 24, 308, 887, 1942]
4-digits -> "1369 2056 1170 1041 0583 1182 0824 0397 0024 0308 0887 1942"
collapse -> "136920561170104105831182082403970024030808871942"
```

CompactSeedQR for the same seed (16 bytes of entropy):

```
entropy = bip39.decode("praise you muffin ... vault").entropy
        = 0xa9d0afa17c8842b76e02fb96a90b9f31
qr.encode_byte_mode(0xa9d0afa17c8842b76e02fb96a90b9f31)
```

That's 16 raw bytes -> ~ 128 bits -> Version 1 QR Byte Mode. The QR is visually tiny but readable by a 2 MP camera at 30 cm.

Reverse path on a SeedSigner:

1. Camera frames the QR.
2. Library decodes the QR payload (numeric or byte).
3. If 48/96 digits of decimals: standard SeedQR, split into 4-digit groups, look up words.
4. If 16/32 raw bytes: CompactSeedQR, run BIP39 entropy-to-mnemonic.
5. Verify BIP39 checksum.
6. Optionally prompt for passphrase.
7. Derive xpub for verification (e.g., `m/84'/0'/0'`) and display fingerprint to compare against the user's records.

## Common pitfalls

- Encoding word indices as 0-padded but variable length ("0", "00", "000", "0001"). The 4-digit fixed pad is mandatory; variable-length means the decoder cannot split unambiguously.
- Including a space or comma between groups. Standard SeedQR is a continuous digit string; spaces push the encoder out of Numeric Mode (3.33 bits/digit) into Alphanumeric Mode (5.5 bits/char) and the QR no longer fits Version 1.
- Generating SeedQR on a phone or laptop (camera-equipped device). The seed is now in plaintext on a connected device; assume the seed is compromised. SeedQR generation belongs on the same air-gapped signer that will read it.
- Photographing the SeedQR for backup. Phones store photos in cloud-synced libraries by default; this is the modern equivalent of mailing your seed to a stranger.
- Confusing CompactSeedQR with Standard. A 16-byte CompactSeedQR cannot be decoded by a wallet that only knows Standard SeedQR. Document which variant you produced.
- Skipping the BIP39 checksum check. Cameras misread digits; a missing checksum means the wallet silently accepts an invalid seed and derives a different valid-looking wallet at a different fingerprint, leading to "where did my coins go" reports.
- Using high error-correction level "H" with a long 24-word seed. The QR may bloat past Version 5, beyond what small camera frames reliably capture.

## References

- SeedSigner SeedQR doc: https://github.com/SeedSigner/seedsigner/blob/dev/docs/seed_qr/README.md
- Krux SeedQR support: https://selfcustody.github.io/krux/getting-started/usage/seed-qr-codes/
- Coldcard Q QR import notes: https://coldcard.com/docs/q
- BIP39 wordlist: https://github.com/bitcoin/bips/blob/master/bip-0039/english.txt
- ISO/IEC 18004 (QR spec): Numeric vs Byte mode encoding densities
