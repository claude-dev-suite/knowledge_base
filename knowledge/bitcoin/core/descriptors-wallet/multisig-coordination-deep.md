# Multisig Coordination with Bitcoin Core - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/core/descriptors-wallet`.
> Canonical source: https://github.com/bitcoin/bitcoin/blob/master/doc/psbt.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/core/descriptors-wallet/SKILL.md

## Concept

Coordinating a k-of-n multisig with Bitcoin Core means three wallets in three different processes contributing to one transaction without ever sharing keys. The mechanism is PSBT (BIP 174 / 370). One node holds the watch-only multisig descriptor and constructs the transaction; each signer holds its own private-key descriptor and adds its signature; the constructor combines partials and finalizes. The descriptor must be byte-identical (including key origin paths and `sortedmulti` versus `multi`) on every machine, otherwise the addresses diverge silently and funds land somewhere nobody can spend.

## Walkthrough / mechanics

The canonical 2-of-3 BIP 48 setup uses native segwit (`wsh(sortedmulti(...))`). Each signer derives a separate hardened path (`m/48h/0h/0h/2h`) and exports an xpub plus its origin fingerprint+path (`[fp/48h/0h/0h/2h]xpub...`). The coordinator concatenates the three origin-tagged xpubs inside `sortedmulti`. `sortedmulti` lexicographically sorts the public keys at each derivation index, which makes the address invariant under signer ordering. Plain `multi` does not sort, so `multi(2,A,B,C)` and `multi(2,B,A,C)` produce different addresses; this is a frequent source of "funds sent to wrong address" incidents.

PSBT roles per BIP 174:

- **Creator**: build empty PSBT with inputs + outputs.
- **Updater**: add UTXOs, redeem scripts, BIP 32 derivations.
- **Signer**: produce a partial signature for inputs the wallet knows.
- **Combiner**: merge multiple PSBTs into one.
- **Finalizer**: convert partial sigs into final scriptSig/witness.
- **Extractor**: pull final transaction hex from a finalized PSBT.

Bitcoin Core implements all roles. `walletcreatefundedpsbt` is creator+updater. `walletprocesspsbt` is signer+optionally finalizer. `combinepsbt`, `finalizepsbt`, `decodepsbt`, `analyzepsbt` cover the rest.

## Worked example

Three signers Alice, Bob, Carol. Each has a hardware wallet exporting an xpub with origin `[fp/48h/0h/0h/2h]`.

Create the watch-only coordinator wallet on the always-online node:

```bash
$ bitcoin-cli -named createwallet \
    wallet_name="ms23-watch" \
    disable_private_keys=true blank=true descriptors=true

$ DESC='wsh(sortedmulti(2,[a1b2c3d4/48h/0h/0h/2h]xpubA.../<0;1>/*,[e5f6a7b8/48h/0h/0h/2h]xpubB.../<0;1>/*,[c9d0e1f2/48h/0h/0h/2h]xpubC.../<0;1>/*))'

# Get checksum
$ bitcoin-cli getdescriptorinfo "$DESC"
{ "checksum": "abc12def", ... }

$ bitcoin-cli -rpcwallet=ms23-watch importdescriptors "[{
  \"desc\": \"${DESC}#abc12def\",
  \"active\": true, \"range\": [0,999], \"timestamp\": \"now\"
}]"
```

Each signer creates a private-key wallet with only their own xprv:

```bash
# On Alice's machine
$ bitcoin-cli -named createwallet wallet_name="alice-sign" blank=true descriptors=true
$ bitcoin-cli -rpcwallet=alice-sign importdescriptors '[{
  "desc": "wsh(sortedmulti(2,[a1b2c3d4/48h/0h/0h/2h]xprvA.../<0;1>/*,[e5f6a7b8/48h/0h/0h/2h]xpubB.../<0;1>/*,[c9d0e1f2/48h/0h/0h/2h]xpubC.../<0;1>/*))#abc12def",
  "active": true, "range": [0,999], "timestamp": "now"
}]'
```

Alice's wallet imports the same descriptor but with her xprv replacing her xpub. Bob and Carol mirror this. The descriptor checksum is identical because the *resolved* keys are identical.

Build, fund, and broadcast:

```bash
# Coordinator: create funded PSBT
$ PSBT=$(bitcoin-cli -rpcwallet=ms23-watch walletcreatefundedpsbt \
    '[]' '[{"bc1qrecipient...": 0.5}]' 0 \
    '{"feeRate": 0.00002, "change_type": "bech32"}' | jq -r .psbt)

# Coordinator analyses
$ bitcoin-cli analyzepsbt "$PSBT" | jq '.next, .inputs[].missing'
"signer"
{"signatures":["a1b2c3d4","e5f6a7b8","c9d0e1f2"]}

# Hand $PSBT to Alice -> she signs
$ PSBT_A=$(bitcoin-cli -rpcwallet=alice-sign walletprocesspsbt "$PSBT" | jq -r .psbt)

# Hand $PSBT_A to Bob -> he signs
$ PSBT_AB=$(bitcoin-cli -rpcwallet=bob-sign walletprocesspsbt "$PSBT_A" | jq -r .psbt)

# Coordinator combines (in this case trivial since each signer chained), finalizes, broadcasts
$ FINAL=$(bitcoin-cli combinepsbt "[\"$PSBT_AB\"]")
$ HEX=$(bitcoin-cli finalizepsbt "$FINAL" | jq -r .hex)
$ bitcoin-cli sendrawtransaction "$HEX"
```

If signers worked in parallel rather than chained, each returns its own partial PSBT; combine all of them in one shot:

```bash
$ FINAL=$(bitcoin-cli combinepsbt "[\"$PSBT_A\",\"$PSBT_B\"]")
$ bitcoin-cli finalizepsbt "$FINAL"
```

## MuSig2 as an alternative to `sortedmulti`

`wsh(sortedmulti(...))` is no longer the only way to run a shared-custody wallet on Core. Since Bitcoin Core 30.0 (October 2025) the descriptor engine parses the BIP 390 `musig(KEY,KEY,...)` key expression — permitted only inside `tr()` or `rawtr()`, never nested inside another `musig()` — and Core sorts the participant keys after derivation and before aggregation, so the order they are written in does not change the address. Since Bitcoin Core 31.0 (April 2026) the wallet can also *receive and spend* those outputs (PR #29675, merged 2025-10-14); 30.0 could only parse and derive them. Neither the 30.0 nor the 31.0 release notes mention MuSig2, so version-check against `doc/descriptors.md` and `test/functional/wallet_musig.py` at the tag you actually run rather than against the notes.

The tradeoff: MuSig2 aggregation is n-of-n, and its spend is an ordinary Taproot key-path spend — one x-only key and one Schnorr signature, on-chain indistinguishable from single-sig — where `wsh(sortedmulti(2,...))` publishes the policy and every participant key at spend time. A k-of-n policy needs one `musig()` leaf per allowed subset in the taproot tree (`tr(H,{pk(musig(A,B,C)/<0;1>/*),pk(musig(B,C)/0/*)})`, a shape Core's own `wallet_musig.py` exercises), which grows fast. The coordination cost is one extra round: each participant calls `walletprocesspsbt` twice — first over the unsigned PSBT to contribute a public nonce (`PSBT_IN_MUSIG2_PUB_NONCE`), then over the combined nonce PSBT to contribute a partial signature (`PSBT_IN_MUSIG2_PARTIAL_SIG`), both BIP 373 fields. `combinepsbt` merges each round exactly as it does partial ECDSA sigs, `decodepsbt` exposes `musig2_participant_pubkeys`, `musig2_pubnonces` and `musig2_partial_sigs`, and `finalizepsbt` refuses while either round is short. Secnonces are held in memory only and never serialized, so a `bitcoind` restart between the two rounds voids the session and everyone must re-nonce.

## Common pitfalls

- `multi` versus `sortedmulti`: every signer must use the same form. `sortedmulti` is the modern default and ordering-independent.
- Different origin fingerprints across signers (`[a1b2c3d4/...]` versus `[A1B2C3D4/...]`). The fingerprint is hex; case matters in some tooling but not in Core's parser. Mismatches in the path itself (`/48h/0h/0h/2h` versus `/48'/0'/0'/2'`) parse to the same thing in Core but other wallets may differ.
- Forgetting to import the multisig descriptor on the signers and instead importing only their personal xprv as a single-sig descriptor. Signers will not recognize multisig inputs and produce nothing.
- Using `walletprocesspsbt` with `sign=false` by accident; the call returns the same PSBT unchanged.
- Treating `combinepsbt` as a signer. It only merges signatures; it cannot create them.
- Race during fee bump: `psbtbumpfee` rebuilds the PSBT but each signer must re-sign because the input set or fee changed.
- Letting `range` drift: the coordinator's `range` is `[0,999]` and the signer's is `[0,99]`. The 100th change address Alice's wallet sees is not derived; she signs nothing.
- Treating a `musig()` descriptor like `sortedmulti`: one `walletprocesspsbt` pass per participant only contributes a nonce, so `finalizepsbt` fails on a PSBT that looks signed. Run the second pass over the combined PSBT.

## References

- BIP 174 (PSBT), BIP 370 (PSBT v2).
- BIP 48 (multisig derivation paths).
- `doc/psbt.md` in bitcoin/bitcoin.
- `walletcreatefundedpsbt`, `walletprocesspsbt`, `analyzepsbt`, `combinepsbt`, `finalizepsbt` RPCs.
- BIP 327 (MuSig2), BIP 328 (aggregate-key derivation), BIP 373 (MuSig2 PSBT fields), BIP 390 (`musig()` descriptor key expression).
- `test/functional/wallet_musig.py` in bitcoin/bitcoin (present at v31.1, absent at v30.0).
