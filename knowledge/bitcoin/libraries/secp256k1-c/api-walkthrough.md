# libsecp256k1 C API Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/secp256k1-c`.
> Canonical source: https://github.com/bitcoin-core/secp256k1
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/secp256k1-c/SKILL.md

## Concept

`libsecp256k1` is the canonical C library for the secp256k1 curve.
It's the engine inside Bitcoin Core's signing path and the foundation
of every language binding (`rust-secp256k1`, `nbitcoin`'s native code,
`tiny-secp256k1` via WASM, etc.). It's constant-time everywhere,
endomorphism-optimised, and structured so unused modules can be
compiled out for embedded targets.

Modules are opt-in at build time:

- default: ECDSA sign / verify, pubkey derivation.
- `extrakeys` + `schnorrsig`: BIP340 (x-only pubkeys + Schnorr).
- `recovery`: ECDSA with public-key recovery.
- `ecdh`: Diffie-Hellman.
- `musig`: BIP327 MuSig2.
- `frost`: experimental FROST.

## Build

```bash
git clone https://github.com/bitcoin-core/secp256k1
cd secp256k1
./autogen.sh
./configure \
    --enable-module-recovery \
    --enable-module-schnorrsig \
    --enable-module-extrakeys \
    --enable-module-musig \
    --enable-experimental
make -j
sudo make install                     # libsecp256k1.so + headers
```

## API walkthrough

A Schnorr (BIP340) sign + verify in pure C:

```c
#include <stdio.h>
#include <string.h>
#include <secp256k1.h>
#include <secp256k1_extrakeys.h>
#include <secp256k1_schnorrsig.h>

static int fill_random(unsigned char *buf, size_t len) {
    /* Use your platform's CSPRNG. For the example assume buf is
       already populated. */
    return 1;
}

int main(void) {
    secp256k1_context *ctx = secp256k1_context_create(SECP256K1_CONTEXT_NONE);

    unsigned char seckey[32], randomize[32];
    fill_random(seckey, 32);
    fill_random(randomize, 32);
    secp256k1_context_randomize(ctx, randomize);   // side-channel hardening

    if (!secp256k1_ec_seckey_verify(ctx, seckey)) {
        fprintf(stderr, "invalid secret key\n");
        return 1;
    }

    secp256k1_keypair keypair;
    secp256k1_keypair_create(ctx, &keypair, seckey);

    secp256k1_xonly_pubkey xonly;
    secp256k1_keypair_xonly_pub(ctx, &xonly, NULL, &keypair);

    unsigned char msg32[32] = {0x42};            /* 32-byte message */
    unsigned char sig64[64];
    secp256k1_schnorrsig_sign32(ctx, sig64, msg32, &keypair, NULL);

    int valid = secp256k1_schnorrsig_verify(ctx, sig64, msg32, 32, &xonly);
    printf("valid: %d\n", valid);

    secp256k1_context_destroy(ctx);
    return 0;
}
```

Compile with `cc demo.c -o demo -lsecp256k1`.

## Worked example: ECDSA with public-key recovery

Recoverable ECDSA is what Bitcoin's `signmessage` / Ethereum-style
flows depend on -- the verifier reconstructs the pubkey from the
signature instead of taking it as input.

```c
#include <secp256k1.h>
#include <secp256k1_recovery.h>

void recover_demo(secp256k1_context *ctx, const unsigned char seckey[32],
                  const unsigned char msg32[32]) {
    secp256k1_ecdsa_recoverable_signature rsig;
    secp256k1_ecdsa_sign_recoverable(ctx, &rsig, msg32, seckey, NULL, NULL);

    unsigned char compact[64];
    int recid;
    secp256k1_ecdsa_recoverable_signature_serialize_compact(
        ctx, compact, &recid, &rsig);

    /* Verifier side: reconstruct (compact, recid) -> rsig -> pubkey */
    secp256k1_ecdsa_recoverable_signature parsed;
    secp256k1_ecdsa_recoverable_signature_parse_compact(
        ctx, &parsed, compact, recid);

    secp256k1_pubkey pubkey;
    secp256k1_ecdsa_recover(ctx, &pubkey, &parsed, msg32);

    unsigned char ser[33];
    size_t outlen = sizeof(ser);
    secp256k1_ec_pubkey_serialize(ctx, ser, &outlen, &pubkey,
                                  SECP256K1_EC_COMPRESSED);
}
```

## Common pitfalls

- **Module flags at build time**: calling `secp256k1_schnorrsig_sign32`
  in code linked against a build without `--enable-module-schnorrsig`
  produces an undefined-symbol link error -- always feature-gate at
  configure time.
- **Context reuse**: contexts are expensive to create. Create one
  global context at process start (`SECP256K1_CONTEXT_NONE` for
  modern builds; older builds split sign / verify) and pass references.
- **`secp256k1_context_randomize`**: call once after creation with 32
  random bytes for side-channel hardening. Skipping it doesn't break
  correctness but weakens the timing model.
- **Cleanup**: pair every `secp256k1_context_create` with a
  `secp256k1_context_destroy` to free heap. Long-running daemons
  should keep one global context.
- **Length validation**: `secp256k1_ec_pubkey_parse` accepts both
  compressed (33) and uncompressed (65) lengths -- pass the actual
  byte length, not a fixed value.

## References

- Repo: https://github.com/bitcoin-core/secp256k1
- BIP340 (Schnorr): https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki
- BIP327 (MuSig2): https://github.com/bitcoin/bips/blob/master/bip-0327.mediawiki
- Examples directory: https://github.com/bitcoin-core/secp256k1/tree/master/examples
- Companion: [secp256k1-rs/SKILL.md](../secp256k1-rs/SKILL.md), [../../cryptography/secp256k1/SKILL.md](../../cryptography/secp256k1/SKILL.md)
