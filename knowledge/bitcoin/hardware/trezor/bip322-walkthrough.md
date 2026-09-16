# Trezor Message Signing vs BIP-322 - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/hardware/trezor`.
> Canonical source: https://github.com/bitcoin/bips/blob/master/bip-0322.mediawiki
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/hardware/trezor/SKILL.md

## Concept

BIP-322 generalizes Bitcoin signed-message proofs to all script types
(P2PKH, P2SH-P2WPKH, P2WPKH, P2TR, multisig). It works by building a
"to_spend" virtual transaction that locks funds to the message, then a
"to_sign" tx that spends it and is signed normally - so any script the
device can sign for ordinary spending also works for proof-of-control.

Trezor firmware does **not** implement BIP-322 message signing, as of
firmware 2.12.4 (19 August 2026) - verified against
`trezor/trezor-firmware` `main`. `SignMessage` / `VerifyMessage` on every
model (Trezor One, Model T, Safe 3, Safe 5, Safe 7) produce and check the
legacy "Bitcoin Signed Message" format (BIP-137 style), optionally in
Electrum-compatible encoding.

What the firmware does reuse from BIP-322 is its `SignatureProof`
container (length-prefixed scriptSig followed by the witness), implemented
in `core/src/apps/bitcoin/scripts.py` as `write_bip322_signature_proof` /
`read_bip322_signature_proof`. That container carries **SLIP-0019 proofs
of ownership**, which are a different mechanism from BIP-322 message
signing - see the walkthrough below.

## Walkthrough

Message signing via trezorctl is legacy-only:

```bash
# Single-sig native segwit (Model T, Safe 3, Safe 5, Safe 7)
trezorctl btc sign-message \
  -c bitcoin \
  -n "m/84h/0h/0h/0/0" \
  -t segwit \
  "Hello, Bitcoin"

# Output is a base64-encoded legacy "Bitcoin Signed Message" signature
```

`-t` / `--script-type` selects the *script* type, not a BIP-322 flavor.
The CLI accepts `address` / `pkh`, `segwit` / `wpkh`, `p2shsegwit` /
`sh-wpkh`, `taproot` / `tr` (`trezorlib/cli/btc.py`, `INPUT_SCRIPTS`), but
`core/src/apps/bitcoin/sign_message.py` rejects `SPENDTAPROOT` with
"Unsupported script type" - message signing covers P2PKH, P2SH-P2WPKH and
P2WPKH only. There is no `bip322-simple` or `bip322-full` option.

The script type is encoded by adding an offset to the recovery byte: 0 for
`SPENDADDRESS`, 4 for `SPENDP2SHWITNESS`, 8 for `SPENDWITNESS`. `-e` /
`--electrum-compat` (`no_script_type`) suppresses that offset, producing a
plain BIP-137 recovery byte.

The BIP-322-adjacent feature Trezor does ship is the SLIP-0019 proof of
ownership (`GetOwnershipProof` / `VerifyOwnershipProof`), exposed through
trezorlib rather than trezorctl:

```python
from trezorlib import btc, messages
from trezorlib.tools import parse_path

proof, signature = btc.get_ownership_proof(
    session,
    "Bitcoin",
    parse_path("m/84h/0h/0h/0/0"),
    script_type=messages.InputScriptType.SPENDWITNESS,
    user_confirmation=True,
    commitment_data=b"coinjoin round id",
)
```

Internally, the firmware (`core/src/apps/bitcoin/ownership.py`):

1. Writes the proof body: 4-byte version magic `SL\x00\x19`, a flags
   byte (bit 0 = user confirmed), then a compact-size count of 32-byte
   ownership identifiers.
2. Hashes `sha256(proof_body)` followed by the length-prefixed
   scriptPubKey and the length-prefixed `commitment_data`.
3. Signs that digest with ECDSA, or BIP-340 for `SPENDTAPROOT`.
4. Appends the BIP-322 `SignatureProof` (scriptSig, witness).

`verify_nonownership` reverses the same construction to prove an input is
*external* - the CoinJoin case SLIP-0019 was written for.

## Worked example

Signing a message with a bc1q address:

```python
from trezorlib import messages
from trezorlib.btc import sign_message
from trezorlib.tools import parse_path

res = sign_message(
    session,
    "Bitcoin",
    parse_path("m/84h/0h/0h/0/0"),
    "Proof of UTXO",
    script_type=messages.InputScriptType.SPENDWITNESS,
)
print(res.address)    # bc1q...
print(res.signature)  # legacy "Bitcoin Signed Message", segwit variant
```

Bitcoin Core cannot verify BIP-322 signatures. Its only message RPCs are
`signmessage` / `verifymessage`, which implement the legacy format for
P2PKH addresses only. Both attempts to add BIP-322 to Core were closed
unmerged - PR #16440 (closed 25 March 2020) and PR #24058 (closed
3 August 2023) - and no BIP-322 code is present in Core as of release
31.1 (8 July 2026):

```bash
# Legacy only, P2PKH addresses:
bitcoin-cli verifymessage <p2pkh_address> <signature_b64> "Proof of UTXO"
```

Verifying a real BIP-322 signature therefore needs a third-party
implementation such as `rust-bitcoin/bip322`, or the signing wallet's own
verifier. Trezor's segwit signatures are not BIP-322 either, so they need
a verifier that understands the recovery-byte offsets above - Core's
`verifymessage` will not accept them.

## Common pitfalls

- **Assuming Trezor emits BIP-322**: it does not, as of firmware 2.12.4
  (August 2026). If a counterparty demands a BIP-322 proof, a Trezor
  cannot produce one; use a wallet that implements BIP-322 itself.
- **Wrong script type**: passing `SPENDP2SHWITNESS` for a native segwit
  address yields a different signature; verifier fails.
- **Address mismatch**: derivation path must match the address you claim
  to control. Many tools default to BIP-44 path for a BIP-84 address.
- **Confusing ownership proofs with message proofs**: a SLIP-0019 proof
  binds a scriptPubKey and commitment data, not an arbitrary message, and
  ordinary message-verification tools cannot read it.
- **Taproot message signing**: unsupported. `-t taproot` is accepted by
  the CLI but the firmware rejects `SPENDTAPROOT` in `SignMessage`.
- **Signature type mismatch on verifier**: the verifier must understand
  the same convention (plain BIP-137 recovery byte vs. the +4 / +8
  script-type offsets).

## References

- BIP-322: https://github.com/bitcoin/bips/blob/master/bip-0322.mediawiki
- SLIP-0019 (proof of ownership): https://github.com/satoshilabs/slips/blob/master/slip-0019.md
- Trezor firmware repo: https://github.com/trezor/trezor-firmware
- Ownership proof implementation: https://github.com/trezor/trezor-firmware/blob/main/core/src/apps/bitcoin/ownership.py
- Bitcoin Core BIP-322 PR (closed unmerged): https://github.com/bitcoin/bitcoin/pull/24058
- Third-party BIP-322 verifier: https://github.com/rust-bitcoin/bip322
- trezorctl docs: https://docs.trezor.io/trezor-suite/packages/connect/methods/signMessage.html
