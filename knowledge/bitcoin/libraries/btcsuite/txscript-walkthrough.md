# btcsuite txscript Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/btcsuite`.
> Canonical source: https://github.com/btcsuite/btcd/tree/master/txscript
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/btcsuite/SKILL.md

## Concept

`txscript` is the Bitcoin Script engine inside btcd, exposed as a
standalone Go package. It provides:

- `ScriptBuilder` -- fluent assembler for opcodes + push data.
- `Engine` -- executes a scriptSig + scriptPubKey pair against a `MsgTx`,
  honouring segwit / Taproot / sighash variants.
- `SighashCache` (named `TxSigHashes` here) -- precomputes BIP143 /
  BIP341 sighash midstates.
- High-level helpers: `PayToAddrScript`, `SignTxOutput`,
  `WitnessSignature`, `RawTxInWitnessSignature`, `RawTxInTaprootSignature`.

These primitives also underpin LND, Loop, Pool, and any Go wallet that
wants to hand-roll script paths (timelocked refunds, HTLCs, custom
multisig).

## API walkthrough

```go
package main

import (
    "fmt"
    "log"

    "github.com/btcsuite/btcd/btcec/v2"
    "github.com/btcsuite/btcd/btcutil"
    "github.com/btcsuite/btcd/chaincfg"
    "github.com/btcsuite/btcd/chaincfg/chainhash"
    "github.com/btcsuite/btcd/txscript"
    "github.com/btcsuite/btcd/wire"
)

func main() {
    // Address from random key
    priv, _ := btcec.NewPrivateKey()
    pub := priv.PubKey()
    addr, err := btcutil.NewAddressWitnessPubKeyHash(
        btcutil.Hash160(pub.SerializeCompressed()), &chaincfg.MainNetParams)
    if err != nil { log.Fatal(err) }
    fmt.Println("addr:", addr.EncodeAddress())

    // Build scriptPubKey
    spk, err := txscript.PayToAddrScript(addr)
    if err != nil { log.Fatal(err) }
    fmt.Printf("scriptPubKey: %x\n", spk)

    // Hand-rolled multisig redeem script
    pub2, _ := btcec.NewPrivateKey()
    builder := txscript.NewScriptBuilder()
    builder.AddOp(txscript.OP_2)
    builder.AddData(pub.SerializeCompressed())
    builder.AddData(pub2.PubKey().SerializeCompressed())
    builder.AddOp(txscript.OP_2)
    builder.AddOp(txscript.OP_CHECKMULTISIG)
    redeem, _ := builder.Script()
    fmt.Printf("redeem: %x\n", redeem)
}
```

## Worked example: sign a P2WPKH input with WitnessSignature

```go
import (
    "github.com/btcsuite/btcd/txscript"
    "github.com/btcsuite/btcd/wire"
)

func signP2WPKH(tx *wire.MsgTx, idx int, prevAmount int64,
                prevSPK []byte, priv *btcec.PrivateKey) error {
    fetcher := txscript.NewCannedPrevOutputFetcher(prevSPK, prevAmount)
    sigHashes := txscript.NewTxSigHashes(tx, fetcher)

    witness, err := txscript.WitnessSignature(
        tx, sigHashes, idx, prevAmount,
        prevSPK, txscript.SigHashAll, priv, true)
    if err != nil { return err }
    tx.TxIn[idx].Witness = witness
    return nil
}
```

For Taproot key-path:

```go
witness, err := txscript.TaprootWitnessSignature(
    tx, sigHashes, idx, prevAmount, prevSPK,
    txscript.SigHashDefault, priv)
if err != nil { return err }
tx.TxIn[idx].Witness = wire.TxWitness{witness}
```

Both helpers handle BIP143 / BIP341 sighash construction. For
script-path Taproot, use `RawTxInTaprootSignature` and assemble the
control block + leaf script yourself.

## Verifying via the Engine

```go
import "github.com/btcsuite/btcd/txscript"

const flags = txscript.StandardVerifyFlags
fetcher := txscript.NewCannedPrevOutputFetcher(prevSPK, prevAmount)
sigHashes := txscript.NewTxSigHashes(tx, fetcher)

vm, err := txscript.NewEngine(prevSPK, tx, idx, flags,
    nil, sigHashes, prevAmount, fetcher)
if err != nil { return err }
if err := vm.Execute(); err != nil { return err }   // returns nil on success
```

Use this in tests to sanity-check signed inputs before broadcasting.

## Common pitfalls

- API drift between `btcec/v1` and `btcec/v2`. `v2` aligns key types
  with stdlib `crypto/ecdsa` and changes signing helpers. Pin a major.
- `TxSigHashes` must be regenerated if you mutate the tx after
  computing it -- the cache is bound to the tx's witness layout.
- `SigHashAll` is **not** the default for Taproot; use
  `SigHashDefault` (== `SigHashAll` for Taproot semantics) so output
  serialisation matches BIP341.
- `PrevOutputFetcher`: the segwit/Taproot sighash needs **all** prev
  outputs. Use `MultiPrevOutFetcher` when signing multiple inputs.
- Engine flags affect what scripts you accept. `StandardVerifyFlags`
  matches Core mempool policy; `MandatoryVerifyFlags` is consensus-
  only. Pick the right set for your test goal.

## References

- Repo: https://github.com/btcsuite/btcd/tree/master/txscript
- pkg.go.dev: https://pkg.go.dev/github.com/btcsuite/btcd/txscript
- BIP143: https://github.com/bitcoin/bips/blob/master/bip-0143.mediawiki
- BIP341: https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki
- Companion: [btcd/SKILL.md](../btcd/SKILL.md), [lnd-go/SKILL.md](../lnd-go/SKILL.md)
