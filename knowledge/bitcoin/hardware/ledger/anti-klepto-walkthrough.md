# Ledger Anti-Klepto Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/hardware/ledger`.
> Canonical source: https://eprint.iacr.org/2018/1086 (Bitstream-style nonce sidechannel)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/hardware/ledger/SKILL.md

## Concept

A malicious or buggy hardware wallet firmware can leak the seed
**through the signatures it produces** by choosing the ECDSA / Schnorr
nonce `k` non-uniformly (e.g., `k = HMAC(seed, counter)`). To an
external observer, signatures look random; to the attacker who knows
the embedded scheme, two signatures are enough to extract the secret
key. This class of attack is sometimes called DUSEC or kleptography.

Anti-klepto (a.k.a. **anti-exfil** in BitBox docs, **commit-sign** in
some specs) defeats this by making the **host** contribute randomness
that the device must commit to **before** signing. The device cannot
later choose a malicious nonce because the nonce derivation is bound
to the host commitment, and the host can verify after-the-fact.

## Walkthrough

Three-message protocol per signature:

```
Host                                  Device
----                                  ------
1. h_aux = random 32 bytes
                  ----h_aux---->
2.                                    R = k*G   (k internal random)
                  <----R-----
3. host commits and reveals h_aux'
                  ----h_aux'--->      (must equal h_aux)
4.                                    k' = k + H(h_aux' || R)  (Schnorr)
                                      sig = sign with nonce k'
                  <----sig-----
5. Host verifies: signature is valid AND
   R_recovered_from_sig == R + H(h_aux || R) * G
```

Because step 2 (`R`) is committed before the device sees any host
randomness it cannot adapt, AND step 4 forces the final nonce to mix
in the host's contribution, the device cannot embed a covert channel
in the resulting `s` value.

Ledger implements this for Schnorr (BIP-340) signing in the Bitcoin app
v2.1+. ECDSA support depends on app version; the Bitcoin app
augments PSBT input data with the host commitment.

## Worked example

Using `ledger-bitcoin` Python with anti-klepto enabled (default in
recent versions):

```python
from ledger_bitcoin import createClient, Chain
from ledger_bitcoin.client_legacy import LedgerClient

client = createClient(transport, chain=Chain.MAIN)
result = client.sign_psbt(psbt, wallet_policy, hmac)

# Behind the scenes for Taproot inputs:
#  host: aux_rand = os.urandom(32)
#  send AntiKlepto_HostCommit APDU (0xE3) with H(aux_rand)
#  device responds with R
#  host sends AntiKlepto_HostReveal APDU (0xE4) with aux_rand
#  device returns Schnorr signature using mixed nonce
#  host verifies R was honored (via partial signature equation)
```

To verify the device honored the protocol, a host can replay:

```python
# Reconstruct expected R-tweak
expected_R = R_device + tagged_hash("BIP0340/aux", aux_rand) * G
recovered_R_from_sig = compute_R(sig, msg, pubkey)
assert recovered_R_from_sig == expected_R
```

## Common pitfalls

- **Host omits aux_rand**: device may fall back to deterministic Schnorr
  (RFC-6979 / BIP-340 default), which still binds nonce to seed but
  loses anti-exfil property. Always supply random aux_rand from a real
  CSPRNG.
- **Stale aux_rand reuse**: don't cache; per-signature freshness is
  required.
- **Partial signature aggregation** in MuSig2 / FROST flows changes
  what "leak channel" means - anti-klepto must be designed per scheme;
  Ledger's MuSig2 support is still beta.
- **Unverified results**: many host wallets accept the signature without
  checking the anti-klepto invariant, defeating the purpose. Sparrow
  and BitBox host code do verify; ensure your integration does too.
- **Battery-backed Bluetooth Nano X**: same protocol, but watch for
  dropped APDUs that look like protocol failures; retry rather than
  assume malice.

## References

- BIP-340 anti-klepto rationale section
- BitBox02 anti-exfil docs: https://shiftcrypto.support/help/en/4-bitbox02/64-what-is-anti-klepto
- Ledger app-bitcoin-new sign_psbt.c (search "ANTI_KLEPTO")
- Specter Desktop verification code
