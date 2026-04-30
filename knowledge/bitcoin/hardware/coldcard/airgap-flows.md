# Coldcard Airgap Flows (microSD / NFC / QR) - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/hardware/coldcard`.
> Canonical source: https://coldcard.com/docs/microsd/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/hardware/coldcard/SKILL.md

## Concept

Coldcard offers four PSBT transports that progressively reduce the
attack surface vs a hot signing host:

1. **USB** - direct, fastest, but PC malware can MITM input data.
2. **microSD** - airgapped via FAT32 card; classic Coldcard flow.
3. **NFC** - tap-to-transfer (Mk4 + Q); 4 kB chunks.
4. **QR** - animated bcur QR codes (Q only, with its color screen).

All four share the same PSBT semantics; only the wire layer differs.
Coldcard always shows full output details on its own screen for user
verification, regardless of transport.

## Walkthrough

### microSD flow (canonical Coldcard airgap)

```
Coordinator (Sparrow / Specter)        Coldcard
1. Build PSBT, save to SD as
   yourwallet.psbt (or unsigned.psbt)
2. Eject SD -> insert into Coldcard
                                       3. Menu: "Ready To Sign"
                                       4. Pick file from SD
                                       5. Confirm outputs on screen
                                       6. Signs, writes
                                          *-signed.psbt next to original
                                       7. Eject SD
8. Insert into coordinator,
   load -finalized.psbt, broadcast
```

File naming: Coldcard appends `-signed`, `-final`, or `-part` per
operation. Multisig signing rounds produce `-part.psbt`; the last signer
sees `-final.psbt`.

### NFC flow (Mk4 / Q)

```
Phone wallet (Nunchuk / Sparrow Android)        Coldcard
1. Build PSBT in app                            2. Menu: "NFC Tools" -> "Sign PSBT"
3. Hold phone over Coldcard                     4. Receives via NFC (NDEF tag)
                                                5. Confirm + sign on screen
                                                6. Tap phone again to read back
7. Phone gets signed PSBT
```

NFC is limited to ~4-8 kB single tap; large multisig PSBTs may need
several taps. NFC works best on Android (full read/write); iOS reads
only.

### QR flow (Q model only)

```
Coordinator                                     Coldcard Q
1. Display animated bcur QR of PSBT             2. Menu: "Scan QR"
                                                3. Camera reads frames
                                                4. Confirm + sign
                                                5. Display animated QR back
6. Scan with coordinator (e.g. Sparrow)
```

The Q uses **bcur** (Blockchain Commons UR Type Registry) with
`crypto-psbt` as the type. Frame rate ~3-5 fps; large PSBTs (>50 inputs)
take 30+ seconds.

## Worked example

PSBT round-trip via SD on Linux:

```bash
# Coordinator side: build with Sparrow CLI (or any wallet)
sparrow-cli psbt-build --recipient bc1q... --amount 0.001 \
    --wallet myhotwallet -o /media/sd/unsigned.psbt

# Eject SD, insert into Coldcard
# On Coldcard menu: "Ready To Sign" -> select unsigned.psbt
# Coldcard writes /media/sd/unsigned-signed.psbt

# Back to coordinator
bitcoin-cli decodepsbt $(base64 < /media/sd/unsigned-signed.psbt)
bitcoin-cli finalizepsbt $(base64 < /media/sd/unsigned-signed.psbt)
bitcoin-cli sendrawtransaction <hex>
```

## Common pitfalls

- **SD format**: must be **FAT32**, max 32 GB. exFAT or ext4 fails to
  mount. New SDXC cards default to exFAT - reformat.
- **Multiple PSBT files on SD**: Coldcard lists alphabetically. Name
  carefully to avoid signing wrong one.
- **NFC tag collisions**: tapping while another NFC-active app is in
  foreground (Apple Pay, payment apps) causes lost reads.
- **QR refresh rate**: too fast loses frames on the receiver; too slow
  bores the user. Default Sparrow speed is good.
- **Wallet policy not registered** for multisig flows: even airgapped,
  device must "know" the multisig descriptor. Use `Settings -> Multisig
  Wallets -> Import from File` first (also via SD).
- **NFC + iOS write**: iOS does not support writing PSBTs back to NFC;
  use Android or fall back to QR/SD.

## References

- Coldcard microSD docs: https://coldcard.com/docs/microsd/
- bcur format spec: https://github.com/BlockchainCommons/Research
- Sparrow docs on Coldcard: https://www.sparrowwallet.com/docs/quick-start.html
