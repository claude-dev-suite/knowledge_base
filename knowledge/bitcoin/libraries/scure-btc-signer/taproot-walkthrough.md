# @scure/btc-signer Taproot - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/scure-btc-signer`.
> Canonical source: https://github.com/paulmillr/scure-btc-signer
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/scure-btc-signer/SKILL.md

## Concept

`@scure/btc-signer` is a modern audited TypeScript Bitcoin library by
Paul Miller's team. It is pure functional (no global mutation), uses
native `BigInt` for amounts (no `Buffer`/`bn.js` mismatch), and depends
only on `@noble/curves`, `@noble/hashes`, and `@scure/base`. The
output is much smaller than `bitcoinjs-lib` for browser bundles, and
the API more closely mirrors the BIP-level concepts.

For Taproot it gives you `p2tr(internalKey, scripts?, network?, ...)`
which returns a payment object with both key-path and script-path
spending info, plus `p2tr_ns()` for n-of-n via `multi_a`. The
`Transaction` class accepts inputs with `tapInternalKey`,
`tapMerkleRoot`, and `tapLeafScript` PSBT fields (BIP371).

## API walkthrough

```ts
import * as btc from "@scure/btc-signer";
import { hex } from "@scure/base";
import { secp256k1 } from "@noble/curves/secp256k1";

const network = btc.NETWORK;             // mainnet
const sk = hex.decode("0101010101010101010101010101010101010101010101010101010101010101");
const pk = secp256k1.getPublicKey(sk, true);     // compressed 33 bytes
const xonly = pk.slice(1);                       // 32 bytes for taproot

// Key-path-only Taproot output
const tr = btc.p2tr(xonly, undefined, network);
console.log("address:", tr.address);             // bc1p...
console.log("scriptPubKey:", hex.encode(tr.script));
```

## Worked example: build, sign, broadcast a key-path Taproot spend

```ts
import * as btc from "@scure/btc-signer";
import { hex } from "@scure/base";
import { secp256k1 } from "@noble/curves/secp256k1";

const network = btc.NETWORK;
const sk = hex.decode("01".repeat(32));
const pk = secp256k1.getPublicKey(sk, true);
const xonly = pk.slice(1);

const tr = btc.p2tr(xonly, undefined, network);

const tx = new btc.Transaction();
tx.addInput({
    txid: "a0b1c2d3e4f5...",                  // 32-byte hex
    index: 0,
    witnessUtxo: { script: tr.script, amount: 100_000n },
    tapInternalKey: xonly,
});
tx.addOutputAddress("bc1q...recipient...", 90_000n, network);

tx.sign(sk);                                  // sigs all key-path inputs
tx.finalize();

const rawHex = hex.encode(tx.extract());
console.log("tx hex:", rawHex);
console.log("txid:", tx.id);
```

For script-path spends, build a leaf list and pass it as the second arg
to `p2tr`. The library tweaks the internal key with the merkle root
and exposes `tapLeafScript` info you must include on the input:

```ts
const leafA = btc.p2tr_pk(xonlyAlice);
const leafB = btc.p2tr_pk(xonlyBob);
const tr = btc.p2tr(xonlyInternal, [leafA, leafB], network);
// ... addInput with tapLeafScript: tr.leaves[0].leafTaprootScript
```

## Common pitfalls

- `@scure/btc-signer` uses `BigInt` for amounts; passing plain `number`
  silently fails type checks at runtime. Always use `100_000n` not
  `100_000`.
- `tapInternalKey` is the **x-only** internal key (32 bytes), not the
  33-byte compressed pubkey. The library will throw on a 33-byte input.
- `btc.NETWORK` is mainnet; testnet is `btc.TEST_NETWORK`. Mismatched
  network silently produces invalid addresses.
- BIP327 MuSig2 isn't built in -- combine with `@noble/curves` MuSig
  primitives if needed.
- Older Safari (pre-2020) lacks `BigInt` -- ship a polyfill if you
  target ancient browsers.

## References

- Repo: https://github.com/paulmillr/scure-btc-signer
- npm: https://www.npmjs.com/package/@scure/btc-signer
- BIP371 (Taproot PSBT fields): https://github.com/bitcoin/bips/blob/master/bip-0371.mediawiki
- Companion: [bitcoinjs-lib/SKILL.md](../bitcoinjs-lib/SKILL.md), [mempool-js/SKILL.md](../mempool-js/SKILL.md)
