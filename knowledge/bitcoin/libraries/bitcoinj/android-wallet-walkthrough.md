# bitcoinj Android Wallet Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/bitcoinj`.
> Canonical source: https://github.com/bitcoinj/bitcoinj
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/bitcoinj/SKILL.md

## Concept

`bitcoinj` is the long-running pure-Java Bitcoin library. It includes
an HD wallet, BIP37 SPV client (deprecated for privacy reasons), tx
construction + signing, and network discovery. For new projects on
Android, `bdk-android` is generally a better choice (descriptor-first,
modern segwit/Taproot support, Rust-backed performance), but bitcoinj
remains the foundation of older wallets and is still used where pure-
Java/Kotlin is a hard requirement.

For Android specifically, the typical pattern is a foreground/background
`Service` that owns the `WalletAppKit` (which bundles the Wallet,
BlockChain, PeerGroup, and a SPV block store). The UI binds to the
service and reads `Wallet` events on the main thread via a handler.

## API walkthrough

```kotlin
import org.bitcoinj.core.*
import org.bitcoinj.kits.WalletAppKit
import org.bitcoinj.params.MainNetParams
import org.bitcoinj.script.Script
import java.io.File

class WalletService {
    private val params: NetworkParameters = MainNetParams.get()
    private lateinit var kit: WalletAppKit

    fun start(dataDir: File) {
        kit = object : WalletAppKit(params, Script.ScriptType.P2WPKH,
                                    null, dataDir, "wallet") {
            override fun onSetupCompleted() {
                wallet().allowSpendingUnconfirmedTransactions()
            }
        }
        kit.startAsync()
        kit.awaitRunning()
    }

    fun receiveAddress(): Address = kit.wallet().currentReceiveAddress()
    fun balance(): Coin = kit.wallet().balance
}
```

`WalletAppKit` writes a `wallet.spvchain` and a `wallet.wallet` file
into `dataDir`. The keychain is BIP32 + BIP39 by default; pass a
`DeterministicSeed` to restore from a mnemonic.

## Worked example: send a payment

```kotlin
import org.bitcoinj.core.*
import org.bitcoinj.wallet.SendRequest
import org.bitcoinj.wallet.Wallet

fun send(wallet: Wallet, peers: PeerGroup,
         destStr: String, amountSat: Long): String {
    val params = wallet.params
    val dest   = Address.fromString(params, destStr)
    val req    = SendRequest.to(dest, Coin.valueOf(amountSat))
    req.feePerKb = Coin.valueOf(2_000)        // 2 sat/byte (~2000 sat/kvB)

    val result = wallet.sendCoins(peers, req)
    return result.tx.txId.toString()
}
```

`Wallet.sendCoins(peers, ...)` returns a `SendResult` with a future
that completes when the tx hits at least one peer. For higher-level
broadcast confirmation, attach a `TransactionConfidence.Listener`.

## Restoring from mnemonic

```kotlin
import org.bitcoinj.crypto.MnemonicCode
import org.bitcoinj.wallet.DeterministicSeed
import java.time.Instant

val mnemonic = "abandon abandon abandon ... about"
val creationTime = Instant.now().epochSecond - 60L * 60L * 24L * 30L
val seed = DeterministicSeed(mnemonic, null, "", creationTime)
kit.restoreWalletFromSeed(seed)
kit.startAsync()
kit.awaitRunning()
```

The `creationTime` lets bitcoinj skip downloading filters before the
seed existed -- crucial on Android where SPV sync from genesis is
infeasible.

## Common pitfalls

- **BIP37 privacy**: the default SPV mode publishes a bloom filter to
  every peer that effectively deanonymises you. For a real wallet,
  use a trusted Electrum / Esplora server or migrate to bdk-android +
  BIP157/158.
- **Segwit / Taproot**: bitcoinj added P2WPKH late and has limited
  Taproot support. Don't ship a wallet that promises Taproot to users
  on bitcoinj alone.
- **Memory pressure on Android**: `WalletAppKit` keeps the chain
  store and recent blocks in memory; aggressive GC kills can corrupt
  the spvchain. Run as a foreground `Service` with `START_STICKY`.
- **Thread safety**: `Wallet` is internally synchronised, but
  long-running listeners run on the bitcoinj thread; marshal back to
  the main thread for UI updates.
- **Soft fork lag**: bitcoinj sometimes ships consensus rule support
  months after Core. Verify `TestNet3Params` versus `MainNetParams`
  for activation height of any feature you depend on.

## References

- Repo: https://github.com/bitcoinj/bitcoinj
- API docs: https://bitcoinj.org/javadoc/0.16.4/
- Examples: https://github.com/bitcoinj/bitcoinj/tree/master/examples
- Companion: [bdk-jvm/SKILL.md](../bdk-jvm/SKILL.md), [../../wallets/hd/SKILL.md](../../wallets/hd/SKILL.md)
