# Foundation Passport QR-only Flow - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/hardware/passport`.
> Canonical source: https://docs.foundation.xyz/passport/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/hardware/passport/SKILL.md

## Concept

Passport eliminates the USB data attack surface entirely: there is no
USB pin connected to the SE for signing data. The USB-C port is for
charging only. All wallet interaction happens via:

1. The on-device camera reading animated QRs from a coordinator.
2. The on-device color screen displaying animated QRs back.
3. A microSD slot as a fallback for very large PSBTs.

This is stricter than Coldcard (which has USB option) and same level of
isolation as SeedSigner / Krux. Foundation's claim is that the only way
to extract the seed is physical SE attack - the QR transport cannot be
manipulated remotely.

Encoding format is **bcur** (UR-2), shared with Jade, Sparrow, Specter,
SeedSigner, etc.

## Walkthrough

### Initial xpub export

1. Boot Passport, unlock via PIN (12-digit anti-phishing words).
2. Account -> Connect Wallet -> pick coordinator (Sparrow / Specter /
   Nunchuk / Envoy / BlueWallet).
3. Pick account derivation (P2WPKH `m/84'/0'/0'`, etc.).
4. Passport shows animated QR (`crypto-account` UR).
5. Coordinator scans frames; xpub registered.

### Sign PSBT via QR

```
Coordinator side:
  1. Build tx, encode as PSBT.
  2. Show animated QR in coordinator UI.

Passport side:
  3. Menu -> Sign with QR.
  4. Camera scans frames until UR is reassembled.
  5. Screen shows summary: total in/out, fee, change.
  6. For each output: address (full bech32) + amount.
  7. User presses checkmark to approve each.
  8. Passport signs internally (SE600M secure element).
  9. Display animated QR of signed PSBT.

Coordinator side:
  10. Scan signed-PSBT QR.
  11. Finalize + broadcast.
```

### microSD fallback

For very large PSBTs (>20 inputs / multisig with many cosigners), the QR
takes too long. Use SD:

1. Coordinator: export PSBT to SD as `tx.psbt`.
2. Eject SD, insert into Passport.
3. Menu -> Sign with microSD.
4. Confirm outputs, sign.
5. Passport writes `tx-signed.psbt`.
6. Eject, insert into coordinator.

## Worked example

UR-2 frame payload structure:

```
ur:crypto-psbt/<seq#>-<total>/<bcur32-encoded fragment>
```

Example animated QR sequence (5 frames):

```
ur:crypto-psbt/1-5/lpadcsiyaxhdcncfgmaeykadaesphkonjnsg...
ur:crypto-psbt/2-5/lpadcsmiaxhdcncfgmaeykadaegltsisrkrym...
ur:crypto-psbt/3-5/lpadcsktaxhdcncfgmaeykadaehkmwjepfryxn...
ur:crypto-psbt/4-5/lpadcsihaxhdcncfgmaeykadaeryqsdkbtzthm...
ur:crypto-psbt/5-5/lpadcsltaxhdcncfgmaeykadaecmcyhmqzsmfgr...
```

Passport's reassembly is fountain-coded so reading frames out of order,
or even missing some, is OK as long as enough partial degree-fragments
are received.

Verification on screen (per output):

```
> Output 1 of 2
  Address: bc1q9...4z7  (line wraps; full string scrollable)
  Amount:  0.00198000 BTC

> Confirm? Yes/No
```

## Common pitfalls

- **Indoor lighting**: QR scan fails in low light. Move under a lamp or
  hold device closer.
- **Screen reflection**: glare on coordinator monitor blanks frames.
  Tilt the screen.
- **PSBT > ~10 KB**: switch to microSD.
- **Wrong UR type**: coordinator sending `crypto-output` (descriptor)
  when Passport expects `crypto-psbt`. Re-check coordinator export
  selection.
- **Multisig wallet not registered**: Passport, like Ledger, requires
  the multisig descriptor to be imported (via QR or SD as
  `wallet.bsms`) before signing. Without it, multisig PSBTs error.
- **Shutting off mid-scan**: Passport battery is small. Charge before
  long sessions.

## References

- Passport docs: https://docs.foundation.xyz/passport/
- bcur UR-2 spec: https://github.com/BlockchainCommons/Research
- Foundation Envoy companion app: https://foundation.xyz/envoy/
