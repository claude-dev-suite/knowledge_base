# bdk-jvm / bdk-android Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/bdk-jvm`.
> Canonical source: https://github.com/bitcoindevkit/bdk-ffi (jvm + android bindings)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/bdk-jvm/SKILL.md

## Concept

`bdk-jvm` and `bdk-android` are UniFFI-generated Kotlin/Java bindings
on top of Rust BDK. They expose the same descriptor-first wallet API
as Rust BDK and bdkpython, ship as Maven artifacts under
`org.bitcoindevkit:bdk-jvm` (desktop / server JVM) and
`org.bitcoindevkit:bdk-android` (Android, with per-ABI native `.so`
files), and integrate cleanly with Kotlin coroutines.

Compared to `bitcoinj`, the value is descriptor-first design (BIP380),
modern segwit + Taproot support, libsecp256k1-backed performance, and
shared API surface across Rust / Python / Swift / Kotlin -- so a
multi-platform wallet has only one core implementation.

## API walkthrough

```kotlin
import org.bitcoindevkit.*

val NET = Network.TESTNET

val descriptor      = Descriptor(
    "wpkh([83b3e5a5/84h/1h/0h]tpub.../0/*)", NET)
val changeDescriptor = Descriptor(
    "wpkh([83b3e5a5/84h/1h/0h]tpub.../1/*)", NET)

val dbPath = "${context.filesDir}/wallet.sqlite"
val wallet = Wallet(descriptor, changeDescriptor, NET, dbPath)

val esplora = EsploraClient(
    url = "https://mempool.space/testnet/api",
    proxy = null,
    concurrency = 4u,
    stopGap = 20u,
    timeoutSeconds = null,
)

val request = wallet.startFullScan().build()
val update  = esplora.fullScan(request, stopGap = 5u, parallelRequests = 1u)
wallet.applyUpdate(update)
println("balance: ${wallet.balance().total}")
```

## Worked example: send a tx from Android (Kotlin coroutines)

Wrap the blocking BDK calls with `withContext(Dispatchers.IO)` so
the main thread stays responsive.

```kotlin
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import org.bitcoindevkit.*

class BdkRepository(private val wallet: Wallet,
                    private val esplora: EsploraClient) {

    suspend fun syncWallet() = withContext(Dispatchers.IO) {
        val req = wallet.startSync().build()
        val upd = esplora.sync(req, parallelRequests = 1u)
        wallet.applyUpdate(upd)
    }

    suspend fun send(toAddress: String, amountSat: ULong,
                     feeRateSatVb: Float): String =
        withContext(Dispatchers.IO) {
            val recipient = Address(toAddress, Network.TESTNET)

            val psbt = TxBuilder()
                .addRecipient(recipient.scriptPubkey(), amountSat)
                .feeRate(FeeRate.fromSatPerVb(feeRateSatVb))
                .finish(wallet)

            require(wallet.sign(psbt, SignOptions())) { "signing failed" }
            val tx = psbt.extractTx()
            esplora.broadcast(tx)
            tx.computeTxid()
        }
}
```

## Persistence and restart

```kotlin
// On a later launch, the same descriptor + db path reopens state
val wallet = Wallet(descriptor, changeDescriptor, NET, dbPath)
runBlocking { repo.syncWallet() }
```

`SqliteStore` (`org.bitcoindevkit.SqliteStore`) is the canonical
persistence layer. For tests, use the in-memory store via
`Wallet(descriptor, changeDescriptor, NET, null)` (older API) or the
explicit memory variant (1.x).

## Gradle setup

```kotlin
// app/build.gradle.kts
dependencies {
    implementation("org.bitcoindevkit:bdk-android:1.0.0")
}

android {
    packaging {
        jniLibs.useLegacyPackaging = false
    }
}
```

The artifact bundles `arm64-v8a`, `armeabi-v7a`, `x86_64`, and `x86`
shared libs; APK size grows by ~3-5 MB per ABI. Ship app bundles
(.aab) so Google Play strips ABIs the device doesn't need.

## Common pitfalls

- APK size: split per ABI or ship as `.aab` to avoid bundling all four
  shared libs into every install.
- ProGuard / R8: keep UniFFI-generated classes:
  `-keep class org.bitcoindevkit.** { *; }`.
- Foreground service: long-running sync should run in a foreground
  service or `WorkManager` job, not the UI thread.
- Version skew: a `bdk-android` major must match `bdk-jvm` if you
  share descriptors across desktop + mobile installs of the same
  wallet -- DB migrations are not always cross-major compatible.
- Esplora rate limiting: a fresh full scan can hit public mempool.space
  hard. Use `concurrency = 4u` and a custom Esplora endpoint for
  retail apps.

## References

- Repo: https://github.com/bitcoindevkit/bdk-ffi
- Maven: https://central.sonatype.com/artifact/org.bitcoindevkit/bdk-android
- BDK book: https://bitcoindevkit.org/docs
- Companion: [bdk/SKILL.md](../bdk/SKILL.md), [bitcoinj/SKILL.md](../bitcoinj/SKILL.md)
