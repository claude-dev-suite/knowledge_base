# bitcoinjs-lib PSBT Flow - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/bitcoinjs-lib`.
> Canonical source: https://github.com/bitcoinjs/bitcoinjs-lib
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/bitcoinjs-lib/SKILL.md

## Concept

PSBT (BIP174) is the Bitcoin standard for passing partially-signed
transactions between wallets, hardware signers, and coordinators.
`bitcoinjs-lib` exposes it through the `Psbt` class: you build inputs
with prevout context (witness UTXO or non-witness UTXO), add outputs,
sign one input at a time with an `ECPair`-shaped signer, finalize, then
extract the broadcastable transaction.

The library is OO with stateful builders. Modern code uses `Psbt`
exclusively and treats the older `TransactionBuilder` (deprecated) as
gone. Browser usage typically pairs `bitcoinjs-lib` with `tiny-secp256k1`
(WASM) plus `bip32`/`bip39` from the same author.

## API walkthrough

```ts
import * as bitcoin from "bitcoinjs-lib";
import { ECPairFactory, ECPairInterface } from "ecpair";
import * as ecc from "tiny-secp256k1";

bitcoin.initEccLib(ecc);
const ECPair = ECPairFactory(ecc);

const network = bitcoin.networks.testnet;
const signer: ECPairInterface = ECPair.fromWIF("cVt4o7BG...", network);

const { address } = bitcoin.payments.p2wpkh({
    pubkey: signer.publicKey, network,
});

const psbt = new bitcoin.Psbt({ network });
psbt.addInput({
    hash: "a0b1c2d3...",         // prev txid (display order)
    index: 0,
    witnessUtxo: {
        script: bitcoin.address.toOutputScript(address!, network),
        value: 100_000,
    },
});
psbt.addOutput({ address: "tb1q...recipient...", value: 90_000 });

psbt.signInput(0, signer);
psbt.finalizeAllInputs();
const tx = psbt.extractTransaction();
console.log(tx.toHex());
console.log("txid:", tx.getId());
console.log("vsize:", tx.virtualSize());
```

## Worked example: cooperate with a remote signer via base64

This is the most common cross-wallet pattern: builder serialises the
PSBT, signer adds partial signatures, builder finalises.

```ts
// Builder
const psbt = new bitcoin.Psbt({ network });
psbt.addInput({
    hash: "a0b1c2d3...",
    index: 0,
    witnessUtxo: { script: spk, value: 100_000 },
    bip32Derivation: [{
        masterFingerprint: Buffer.from("d34db33f", "hex"),
        path: "m/84'/1'/0'/0/0",
        pubkey: signer.publicKey,
    }],
});
psbt.addOutput({ address: dest, value: 90_000 });
const unsignedB64 = psbt.toBase64();

// Signer (could be a HW wallet, another process, or a server)
const incoming = bitcoin.Psbt.fromBase64(unsignedB64, { network });
incoming.signInput(0, signer);
const signedB64 = incoming.toBase64();

// Builder finalises
const finalPsbt = bitcoin.Psbt.fromBase64(signedB64, { network });
finalPsbt.finalizeAllInputs();
const rawHex = finalPsbt.extractTransaction().toHex();
```

`witnessUtxo` is required for segwit inputs; for legacy inputs supply
`nonWitnessUtxo` (the full prev tx hex) instead. Mixing the two on a
witness input wastes signing material; on a legacy input only
`nonWitnessUtxo` is accepted.

## Common pitfalls

- `ECPair` was removed from the main package in v6+. Install `ecpair`
  separately and pass `tiny-secp256k1` to its factory.
- Forgetting `bitcoin.initEccLib(ecc)` before Taproot ops -> runtime
  error: `No ecc library provided`.
- Passing prev txid in wrong byte order. `addInput.hash` accepts the
  display-order string (same as block explorers); internally it
  reverses to wire order.
- `finalizeAllInputs` requires every input to have a complete signature
  set; partial signing leaves it un-finalisable until other signers add
  theirs.
- Browser bundlers (Webpack 5+, Vite) need `Buffer` and `crypto` shims.
  Use `vite-plugin-node-polyfills` or the `buffer` package.

## References

- Repo: https://github.com/bitcoinjs/bitcoinjs-lib
- BIP174 (PSBT): https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki
- ecpair: https://github.com/bitcoinjs/ecpair
- Companion: [scure-btc-signer/SKILL.md](../scure-btc-signer/SKILL.md), [bolt11-decoder/SKILL.md](../bolt11-decoder/SKILL.md)
