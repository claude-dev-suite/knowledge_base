# Blockstream Jade BCUR Airgap Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/hardware/jade`.
> Canonical source: https://github.com/Blockstream/Jade
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/hardware/jade/SKILL.md

## Concept

Jade has both online (Pinserver-mediated USB) and offline modes. The
offline mode uses **bcur** (Blockchain Commons UR), a multi-frame QR
encoding that supports forward error correction via fountain codes.
This lets a coordinator wallet (Sparrow, Specter, Nunchuk, Green) and
Jade exchange xpubs and PSBTs without any USB or network connection.

Jade has both a screen (for output) and a camera (for input), which
makes it a true peer in QR exchanges - it can also act as the displayer
for the partial signature it produces.

The relevant UR types:

- `crypto-account` - bundle of xpubs + key origin info.
- `crypto-psbt` - a PSBT.
- `crypto-output` - a single descriptor output.

## Walkthrough

### Export xpub via QR (initial wallet setup)

1. Boot Jade, unlock via PIN.
2. Settings -> Wallet -> Export Master Pubkey.
3. Pick script type (P2WPKH for native segwit, P2WSH for multisig).
4. Pick derivation (e.g. `m/84'/0'/0'`).
5. Jade displays animated QR, type `crypto-account`.
6. Coordinator (Sparrow): "Add wallet" -> "From Hardware" -> "Jade" ->
   "Scan QR".
7. Coordinator camera reads frames until UR is reassembled.
8. Coordinator validates checksum, imports xpub + fingerprint + path.

### Sign PSBT via QR

1. Coordinator: build tx, generate PSBT.
2. Coordinator: "Send to Jade" -> displays animated `crypto-psbt` QR.
3. Jade: Menu -> Scan QR.
4. Jade reads frames, displays:
   - Total inputs / outputs.
   - Each output destination + amount (verified on-screen).
   - Fee estimate.
5. User confirms via Jade buttons.
6. Jade signs internally, displays signed PSBT as `crypto-psbt` QR.
7. Coordinator: "Receive from Jade" -> scans frames.
8. Coordinator finalizes + broadcasts.

## Worked example

Encoding a PSBT for Jade:

```
psbt_bytes = base64.decode("cHNidP8BAH...")
ur_object = UR("crypto-psbt", psbt_bytes)

encoder = UREncoder(ur_object,
                    max_fragment_len=200,   # bytes per QR
                    first_seq_num=0)
while True:
    part = encoder.next_part()
    display_qr(part)            # ~3-5 fps
    sleep(0.2)
    if encoder.is_complete() and encoder.seq_num >= 2*total_parts:
        break    # send 2x for FEC
```

Decoding on Jade side (firmware C code conceptually):

```c
ur_decoder_t dec = ur_decoder_init();
while (!ur_decoder_is_complete(&dec)) {
    char* qr = camera_capture_qr();
    ur_decoder_receive_part(&dec, qr);
}
const uint8_t* psbt = ur_decoder_result(&dec, &len);
parse_psbt(psbt, len);
```

The fountain coding means receiver doesn't need to read frames in order
or read every frame; it just needs enough degree-1+2+3 fragments to
solve the system. Lost frames are tolerated.

## Common pitfalls

- **UR version mismatch**: bcur has UR-1 (older) and UR-2 (current)
  with different message structures. Coordinator and device must agree
  - Sparrow/Jade default to UR-2.
- **Frame rate too high**: at 10+ fps, Jade's camera misses frames; FEC
  recovers but slower than 3-5 fps direct. Start slow.
- **Lighting**: glare on Jade's screen or coordinator's monitor causes
  scan failures. Adjust angle.
- **Large PSBTs (>5 KB)**: 50+ frames at 3 fps = 15+ seconds; users
  often quit early. Use SD card transport instead.
- **Mixing crypto-psbt and crypto-output**: if Jade expects a PSBT but
  coordinator sends an output descriptor, Jade displays "wrong UR
  type". Re-check coordinator export selection.
- **Liquid-vs-Bitcoin confusion**: Jade also supports Liquid CT PSBT
  (`crypto-elements-psbt`); make sure the coordinator emits the right
  type for the network.

## References

- Jade firmware: https://github.com/Blockstream/Jade
- bcur spec: https://github.com/BlockchainCommons/Research/tree/master/papers
- Sparrow Jade integration docs: https://www.sparrowwallet.com/docs/quick-start.html
