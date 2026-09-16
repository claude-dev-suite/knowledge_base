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

All code here targets **bitcoinj 0.17.x** (`org.bitcoinj:bitcoinj-core:0.17.1`,
released 4 May 2026). 0.17 moved `Coin`, `Address` and `ScriptType` into
`org.bitcoinj.base`, superseded `NetworkParameters` with the `Network`
interface / `BitcoinNetwork` enum, and deprecated the static
`Address.fromString()` constructors in favour of `AddressParser`, so the
0.16-era snippets this walkthrough previously used no longer compile.
The breakage is narrower than it looks: `NetworkParameters`,
`MainNetParams.get()` and `Address.fromString(params, str)` all still
resolve in 0.17.1 as deprecated shims -- it is `Script.ScriptType` and
the `org.bitcoinj.core` spellings of `Coin`/`Address` that are gone.
Ship 0.17.1, not 0.16.x -- see the security note under Common pitfalls.

```kotlin
import org.bitcoinj.base.Address
import org.bitcoinj.base.BitcoinNetwork
import org.bitcoinj.base.Coin
import org.bitcoinj.base.ScriptType
import org.bitcoinj.kits.WalletAppKit
import org.bitcoinj.wallet.KeyChainGroupStructure
import java.io.File

class WalletService {
    private val network = BitcoinNetwork.MAINNET
    private lateinit var kit: WalletAppKit

    fun start(dataDir: File) {
        kit = object : WalletAppKit(network, ScriptType.P2WPKH,
                                    KeyChainGroupStructure.BIP32,
                                    dataDir, "wallet") {
            override fun onSetupCompleted() {
                peerGroup().setMaxConnections(6)
            }
        }
        kit.startAsync()
        kit.awaitRunning()
    }

    fun receiveAddress(): Address = kit.wallet().currentReceiveAddress()
    fun balance(): Coin = kit.wallet().balance
}
```

The `KeyChainGroupStructure` argument is no longer nullable: the 0.17
`WalletAppKit(BitcoinNetwork, ...)` constructor runs
`Objects.requireNonNull(structure)`, so the old `null` placeholder throws.
Pass `KeyChainGroupStructure.BIP32` for the historical bitcoinj layout or
`.BIP43` for BIP-44/84 paths. `WalletAppKit.launch(network, dir, prefix)`
is a one-liner alternative when you do not need `onSetupCompleted()`.

`WalletAppKit` writes a `wallet.spvchain` and a `wallet.wallet` file
into `dataDir`. The keychain is BIP32 + BIP39 by default; pass a
`DeterministicSeed` to restore from a mnemonic.

## Worked example: send a payment

```kotlin
import org.bitcoinj.base.Coin
import org.bitcoinj.core.PeerGroup
import org.bitcoinj.wallet.SendRequest
import org.bitcoinj.wallet.Wallet

fun send(wallet: Wallet, peers: PeerGroup,
         destStr: String, amountSat: Long): String {
    val dest = wallet.parseAddress(destStr)   // or AddressParser.getDefault(network)
    val req  = SendRequest.to(dest, Coin.valueOf(amountSat))
    req.feePerKb = Coin.valueOf(2_000)        // 2 sat/vB (2000 sat/vkB)

    val result = wallet.sendCoins(peers, req)
    return result.transaction().txId.toString()
}
```

`Wallet.sendCoins(peers, ...)` returns a `SendResult`. In 0.17 its public
fields `tx`, `broadcast` and `broadcastComplete` are deprecated in favour
of `transaction()`, `getBroadcast()` and `awaitRelayed()`, which returns a
`java.util.concurrent.CompletableFuture<TransactionBroadcast>` completing
when the tx has reached at least one peer -- Guava `ListenableFuture` is
gone from the API (the transitional `ListenableCompletableFuture`
implements both and disappears after 0.17). For higher-level broadcast
confirmation, attach a `TransactionConfidence.Listener`. To spend change
that has not confirmed yet, call `req.allowUnconfirmed()`; there is no
`Wallet.allowSpendingUnconfirmedTransactions()` in 0.16.x or 0.17.x.

`SendRequest.feePerKb` defaults to `Transaction.DEFAULT_TX_FEE`, which
0.17.1 lowered to 10000 sat/vkB.

## Restoring from mnemonic

```kotlin
import org.bitcoinj.wallet.DeterministicSeed
import java.time.Duration
import java.time.Instant

val mnemonic = "abandon abandon abandon ... about"
val creationTime = Instant.now().minus(Duration.ofDays(30))
val seed = DeterministicSeed.ofMnemonic(mnemonic, "", creationTime)
kit.restoreWalletFromSeed(seed)
kit.startAsync()
kit.awaitRunning()
```

0.17 replaced the `DeterministicSeed(...)` constructors with static
factories (`ofMnemonic`, `ofEntropy`, `ofRandom`) and swapped every
integer timestamp in the API for `java.time.Instant` (and intervals for
`java.time.Duration`). The old constructors survive as deprecated stubs
in 0.17.x. `DeterministicSeed` itself stays in `org.bitcoinj.wallet`; it
is the crypto API around it that moved to `org.bitcoinj.crypto`, where
`AesKey` replaces the BouncyCastle `KeyParameter`.

The `creationTime` lets bitcoinj skip downloading filters before the
seed existed -- crucial on Android where SPV sync from genesis is
infeasible.

## Common pitfalls

- **Run 0.17.1 or newer**: 0.15.x, 0.16.x and 0.17 are all affected by
  GHSA-hfcf-v2f8-x9pc / CVE-2026-44714 (CVSS 7.5, disclosed 4 May 2026),
  two fast-path bugs in `ScriptExecution.correctlySpends()` where the
  P2PKH and native-P2WPKH branches verify an attacker-supplied signature
  against an attacker-supplied pubkey without binding that pubkey to the
  hash in the `scriptPubKey`. Only 0.17.1 is patched -- there is no 0.16.x
  backport as of September 2026. A pure BIP37 SPV wallet never verifies
  input signatures and so is not affected by this, but any code path that
  calls `correctlySpends()` is.
- **BIP37 privacy**: the default SPV mode publishes a bloom filter to
  every peer that effectively deanonymises you. For a real wallet,
  use a trusted Electrum / Esplora server or migrate to bdk-android +
  BIP157/158.
- **Segwit / Taproot**: bitcoinj added P2WPKH late, and Taproot is
  send-only as of 0.17.1 (September 2026) -- `ScriptType.P2TR` and
  `SegwitAddress` let you pay a `bc1p...` address, but
  `DeterministicKeyChain` still accepts only P2PKH or P2WPKH as the
  wallet's output script type and there is no Schnorr signing, so you
  cannot receive to or spend from P2TR. Don't ship a wallet that
  promises Taproot to users on bitcoinj alone.
- **Memory pressure on Android**: `WalletAppKit` keeps the chain
  store and recent blocks in memory; aggressive GC kills can corrupt
  the spvchain. Run as a foreground `Service` with `START_STICKY`.
- **Thread safety**: `Wallet` is internally synchronised, but
  long-running listeners run on the bitcoinj thread; marshal back to
  the main thread for UI updates.
- **Soft fork lag**: bitcoinj sometimes ships consensus rule support
  months after Core. Verify any rule you depend on against
  `BitcoinNetwork.TESTNET` before relying on it under
  `BitcoinNetwork.MAINNET`.

## References

- Repo: https://github.com/bitcoinj/bitcoinj
- API docs: https://bitcoinj.org/javadoc/0.17.1/
- Release notes: https://bitcoinj.org/release-notes
- Advisory: https://github.com/bitcoinj/bitcoinj/security/advisories/GHSA-hfcf-v2f8-x9pc
- Examples: https://github.com/bitcoinj/bitcoinj/tree/master/examples
- Companion: [bdk-jvm/SKILL.md](../bdk-jvm/SKILL.md), [../../wallets/hd/SKILL.md](../../wallets/hd/SKILL.md)
