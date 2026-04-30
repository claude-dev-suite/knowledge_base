# Trezor BIP-322 Message Signing - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/hardware/trezor`.
> Canonical source: https://github.com/bitcoin/bips/blob/master/bip-0322.mediawiki
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/hardware/trezor/SKILL.md

## Concept

BIP-322 generalizes Bitcoin signed-message proofs to all script types
(P2PKH, P2SH-P2WPKH, P2WPKH, P2TR, multisig). It works by building a
"to_spend" virtual transaction that locks funds to the message, then a
"to_sign" tx that spends it and is signed normally - so any script the
device can sign for ordinary spending also works for proof-of-control.

Trezor implements BIP-322 in firmware 2.6.0+ (Model T) and Safe 3 / 5.
Trezor One supports only legacy "Bitcoin Signed Message" (BIP-137 style)
because its UI cannot fit BIP-322 confirmation screens for all input
types.

## Walkthrough

Sign via trezorctl:

```bash
# Single-sig native segwit (Safe 3, Model T, Safe 5)
trezorctl btc sign-message \
  -c bitcoin \
  -n "m/84h/0h/0h/0/0" \
  -t bip322-simple \
  "Hello, Bitcoin"

# Output is a base64-encoded BIP-322 simple signature
```

Three signature flavors the firmware accepts:

- `legacy` - BIP-137, P2PKH only.
- `bip322-simple` - 1-input/1-output to_sign, witness-only encoding.
- `bip322-full` - full PSBT/witness, used for multisig and Tapscript.

Trezor Suite exposes BIP-322 under "Sign / Verify message" with a
dropdown for signature type. The device screen shows:

1. The full message (scrollable).
2. The address being proved (full bech32 string).
3. Confirm / Cancel buttons.

Internally, the firmware:

1. Constructs `to_spend` deterministically (BIP-322 spec: nSequence 0,
   nLockTime 0, OP_RETURN message hash output value 0).
2. Constructs `to_sign` spending `to_spend:0` to OP_RETURN.
3. Hashes per BIP-143 (segwit) or BIP-341 (Taproot) sighash rules.
4. Signs with key at the requested derivation path.
5. Serializes the witness (simple form) or full PSBT.

## Worked example

Verifying a BIP-322 signature on bc1q address:

```python
from trezorlib.btc import sign_message
from trezorlib.tools import parse_path

client = get_default_client()
res = sign_message(
    client,
    "Bitcoin",
    parse_path("m/84h/0h/0h/0/0"),
    "Proof of UTXO",
    script_type=proto.InputScriptType.SPENDWITNESS,
)
print(res.address)    # bc1q...
print(res.signature)  # base64 BIP-322-simple
```

Verify with bitcoin-cli `verifymessage` (legacy) or `verifymessagewithsig`
(BIP-322; available in Bitcoin Core 28+):

```bash
bitcoin-cli verifymessagewithsig <address> <signature_b64> "Proof of UTXO"
```

## Common pitfalls

- **Trezor One** only supports legacy; UI rejects BIP-322 requests.
- **Wrong script type**: passing `SPENDP2SHWITNESS` for a native segwit
  address yields a different signature; verifier fails.
- **Address mismatch**: derivation path must match the address you claim
  to control. Many tools default to BIP-44 path for a BIP-84 address.
- **Tapscript multisig BIP-322** still experimental; firmware support
  varies. Test on testnet first.
- **Signature type mismatch on verifier**: verifier must understand the
  same flavor (legacy / simple / full).

## References

- BIP-322: https://github.com/bitcoin/bips/blob/master/bip-0322.mediawiki
- Trezor firmware repo: https://github.com/trezor/trezor-firmware
- trezorctl docs: https://docs.trezor.io/trezor-suite/packages/connect/methods/signMessage.html
