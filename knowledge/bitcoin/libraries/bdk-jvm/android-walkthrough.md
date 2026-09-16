# bdk-jvm / bdk-android Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/bdk-jvm`.
> Canonical source: https://github.com/bitcoindevkit/bdk-ffi (android bindings,
> `bdk-android/`) and https://github.com/bitcoindevkit/bdk-jvm (jvm bindings,
> split out of bdk-ffi in July 2025)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/bdk-jvm/SKILL.md

## Concept

`bdk-jvm` and `bdk-android` are UniFFI-generated Kotlin/Java bindings
on top of Rust BDK. They expose the same descriptor-first wallet API
as Rust BDK and bdkpython, ship as Maven artifacts under
`org.bitcoindevkit:bdk-jvm` (desktop / server JVM) and
`org.bitcoindevkit:bdk-android` (Android, with per-ABI native `.so`
files), and integrate cleanly with Kotlin coroutines. Both artifacts are at
3.1.0 (September 2026), tracking bdk-ffi 3.1.0 / bdk_wallet 3.1.0.

Compared to `bitcoinj`, the value is descriptor-first design (BIP380),
modern segwit + Taproot support, libsecp256k1-backed performance, and
shared API surface across Rust / Python / Swift / Kotlin -- so a
multi-platform wallet has only one core implementation.

## API walkthrough

```kotlin
import org.bitcoindevkit.*

val NET = Network.TESTNET

// Since 3.0.0 (June 2026) the Descriptor constructor takes a NetworkKind
// (MAIN / TEST), not a Network.
val descriptor      = Descriptor(
    "wpkh([83b3e5a5/84h/1h/0h]tpub.../0/*)", NetworkKind.TEST)
val changeDescriptor = Descriptor(
    "wpkh([83b3e5a5/84h/1h/0h]tpub.../1/*)", NetworkKind.TEST)

val dbPath = "${context.filesDir}/wallet.sqlite"
val persister: Persister = Persister.newSqlite(dbPath)
val wallet = Wallet(descriptor, changeDescriptor, NET, persister)

// As of 3.1.0 the constructor takes only url + optional proxy; stop gap and
// parallelism are arguments to the scan call itself.
val esplora = EsploraClient("https://mempool.space/testnet/api")

val request = wallet.startFullScan().build()
val update  = esplora.fullScan(request, 20uL, 4uL)
wallet.applyUpdate(update)
wallet.persist(persister)
println("balance: ${wallet.balance().total.toSat()}")
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
        val req = wallet.startSyncWithRevealedSpks().build()
        val upd = esplora.sync(req, 1uL)
        wallet.applyUpdate(upd)
    }

    suspend fun send(toAddress: String, amountSat: ULong,
                     feeRateSatVb: ULong): String =
        withContext(Dispatchers.IO) {
            val recipient = Address(toAddress, Network.TESTNET)

            // addRecipient takes an Amount object and fromSatPerVb a ULong.
            val psbt = TxBuilder()
                .addRecipient(recipient.scriptPubkey(), Amount.fromSat(amountSat))
                .feeRate(FeeRate.fromSatPerVb(feeRateSatVb))
                .finish(wallet)

            require(wallet.sign(psbt, null)) { "signing failed" }
            val tx = psbt.extractTx()
            esplora.broadcast(tx)
            tx.computeTxid()
        }
}
```

## Persistence and restart

```kotlin
// On a later launch, load rather than create: same descriptors + same db
val persister = Persister.newSqlite(dbPath)
val wallet = Wallet.load(descriptor, changeDescriptor, persister)
runBlocking { repo.syncWallet() }
```

`Persister` (`org.bitcoindevkit.Persister`) is the canonical persistence
handle as of 3.1.0: `Persister.newSqlite(path)` for on-disk SQLite and
`Persister.newInMemory()` for tests. `Wallet(...)` creates a fresh wallet
and `Wallet.load(...)` reopens an existing one; changes are only durable
after `wallet.persist(persister)`.

## Gradle setup

```kotlin
// app/build.gradle.kts
dependencies {
    // 3.1.0 is current as of September 2026
    implementation("org.bitcoindevkit:bdk-android:3.1.0")
}

android {
    packaging {
        jniLibs.useLegacyPackaging = false
    }
}
```

The 3.1.0 artifact bundles `arm64-v8a`, `armeabi-v7a` and `x86_64`
shared libs (32-bit `x86` is not built); APK size grows per ABI. Ship app
bundles (.aab) so Google Play strips ABIs the device doesn't need.
`minSdk` is 24 and the library compiles against JVM 17.

## Common pitfalls

- APK size: split per ABI or ship as `.aab` to avoid bundling all three
  shared libs into every install.
- ProGuard / R8: keep UniFFI-generated classes:
  `-keep class org.bitcoindevkit.** { *; }`.
- Foreground service: long-running sync should run in a foreground
  service or `WorkManager` job, not the UI thread.
- Version skew: keep `bdk-android` and `bdk-jvm` on the same version if
  you share a database across desktop + mobile installs of the same
  wallet. The 3.x wallet does open v1 and v2 SQLite databases and
  migrates them in place on the next `persist`: a v1 database gains both
  the `bdk_descriptor_derived_spks` and `bdk_wallet_locked_outpoints`
  tables, while a v2 database already has the former and gains only
  `bdk_wallet_locked_outpoints`.
- Pre-1.0 databases (0.32 and earlier) are *not* auto-migrated. Read the
  old keychains with `persister.getPreV1WalletKeychains()` -- exposed in
  3.0.0 (June 2026) -- and replay the derivation indices into a fresh
  wallet with `wallet.revealAddressesTo(keychain, lastDerivationIndex)`.
- Esplora rate limiting: a fresh full scan can hit public mempool.space
  hard. Keep the `parallelRequests` argument to `fullScan` / `sync` low
  and point a custom Esplora endpoint at retail apps.

## References

- Repos: https://github.com/bitcoindevkit/bdk-ffi (android),
  https://github.com/bitcoindevkit/bdk-jvm (jvm)
- Changelog: https://github.com/bitcoindevkit/bdk-ffi/blob/master/CHANGELOG.md
- Maven: https://central.sonatype.com/artifact/org.bitcoindevkit/bdk-android
- bdk-jvm API docs: https://bitcoindevkit.github.io/bdk-jvm/
- Book of BDK: https://bookofbdk.com -- the docs home as of September 2026;
  the old `bitcoindevkit.org/docs` path now 404s and bitcoindevkit.org is
  the project / foundation site. Kotlin cookbook examples pin 3.0.0.
- Companion: [bdk/SKILL.md](../bdk/SKILL.md), [bitcoinj/SKILL.md](../bitcoinj/SKILL.md)
