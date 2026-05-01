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

## Common pitfalls

- `multi` versus `sortedmulti`: every signer must use the same form. `sortedmulti` is the modern default and ordering-independent.
- Different origin fingerprints across signers (`[a1b2c3d4/...]` versus `[A1B2C3D4/...]`). The fingerprint is hex; case matters in some tooling but not in Core's parser. Mismatches in the path itself (`/48h/0h/0h/2h` versus `/48'/0'/0'/2'`) parse to the same thing in Core but other wallets may differ.
- Forgetting to import the multisig descriptor on the signers and instead importing only their personal xprv as a single-sig descriptor. Signers will not recognize multisig inputs and produce nothing.
- Using `walletprocesspsbt` with `sign=false` by accident; the call returns the same PSBT unchanged.
- Treating `combinepsbt` as a signer. It only merges signatures; it cannot create them.
- Race during fee bump: `psbtbumpfee` rebuilds the PSBT but each signer must re-sign because the input set or fee changed.
- Letting `range` drift: the coordinator's `range` is `[0,999]` and the signer's is `[0,99]`. The 100th change address Alice's wallet sees is not derived; she signs nothing.

## References

- BIP 174 (PSBT), BIP 370 (PSBT v2).
- BIP 48 (multisig derivation paths).
- `doc/psbt.md` in bitcoin/bitcoin.
- `walletcreatefundedpsbt`, `walletprocesspsbt`, `analyzepsbt`, `combinepsbt`, `finalizepsbt` RPCs.
