# NBitcoin PSBT Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/nbitcoin`.
> Canonical source: https://github.com/MetacoSA/NBitcoin
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/nbitcoin/SKILL.md

## Concept

`NBitcoin` is the comprehensive .NET / C# Bitcoin library by Nicolas
Dorier. It powers BTCPay Server, NBXplorer, and most enterprise .NET
Bitcoin integrations. The `PSBT` class implements BIP174 with a fluent
API: build a transaction, wrap it in a PSBT, attach prevout coins +
HD derivations, sign in any order across cooperating signers,
finalize, and extract.

Compared to bitcoinjs-lib, NBitcoin's PSBT integrates with the rest of
the library (`Coin` abstraction, `TransactionBuilder`, `BitcoinExtKey`)
so end-to-end wallet flows feel native rather than bolted on.

## API walkthrough

```csharp
using NBitcoin;

var network = Network.Main;

// Random key + address
var key  = new Key();
var addr = key.PubKey.GetAddress(ScriptPubKeyType.Segwit, network);
Console.WriteLine(addr);                  // bc1q...

// Manual PSBT
var tx = network.CreateTransaction();
tx.Inputs.Add(new TxIn(new OutPoint(uint256.Parse("ab12..."), 0)));
tx.Outputs.Add(new TxOut(Money.Coins(0.001m), addr));

var psbt = PSBT.FromTransaction(tx, network);
psbt.AddCoins(new Coin(
    new OutPoint(uint256.Parse("ab12..."), 0),
    new TxOut(Money.Coins(0.002m), addr)));
psbt.SignAll(ScriptPubKeyType.Segwit, key);
var final = psbt.Finalize().ExtractTransaction();
Console.WriteLine(final.ToHex());
```

`SignAll(ScriptPubKeyType.Segwit, key)` walks every input the key can
sign; for finer control use `psbt.Inputs[i].Sign(key)`.

## Worked example: builder-driven PSBT with HD signing

```csharp
using NBitcoin;

var network = Network.TestNet;

// Restore HD key from mnemonic
var mnemonic = new Mnemonic("abandon abandon abandon abandon abandon abandon " +
                            "abandon abandon abandon abandon abandon about");
var seed     = mnemonic.DeriveSeed();
var master   = ExtKey.CreateFromSeed(seed);
var account  = master.Derive(new KeyPath("m/84'/1'/0'"));
var receive  = account.Derive(new KeyPath("0/0"));
var change   = account.Derive(new KeyPath("1/0"));

var receiveAddr = receive.Neuter().PubKey
    .GetAddress(ScriptPubKeyType.Segwit, network);
var changeAddr  = change.Neuter().PubKey
    .GetAddress(ScriptPubKeyType.Segwit, network);

// Pretend UTXO
var prevOut  = new OutPoint(uint256.Parse("aa".PadLeft(64, '0')), 0);
var prevTxOut = new TxOut(Money.Coins(0.01m), receiveAddr);
var coin     = new Coin(prevOut, prevTxOut);

// TransactionBuilder integrates with PSBT
var dest = new BitcoinAddress(...);
var builder = network.CreateTransactionBuilder()
    .AddCoins(coin)
    .Send(dest, Money.Coins(0.005m))
    .SetChange(changeAddr)
    .SendEstimatedFees(new FeeRate(2.0m));   // sat/vB

var psbt = builder.BuildPSBT(false);          // unsigned

// Attach BIP32 derivation hints so HW signers can identify keys
psbt.AddKeyPath(receive, new KeyPath("0/0"));

// Sign with the leaf key
psbt.SignAll(ScriptPubKeyType.Segwit, receive);
var tx = psbt.Finalize().ExtractTransaction();
```

The `AddKeyPath` calls populate `bip32_derivation` PSBT fields used by
hardware wallets / coordinators to know which xpub originated each
input.

## HW wallet integration via HWI

```csharp
using NBitcoin.HWWallets;     // experimental helper

var hwi = new HwiClient(NetworkType.Mainnet);
var devices = await hwi.EnumerateAsync(CancellationToken.None);
var device  = devices.First();
var signed  = await hwi.SignTxAsync(device.DeviceSelector, psbt,
                                    CancellationToken.None);
```

NBitcoin shells out to the `hwi` Python tool by default; the call
returns the PSBT with partial signatures filled in.

## Common pitfalls

- `Money` vs `long`: `Money.Coins(0.001m)` is BTC; `Money.Satoshis(100_000)`
  is sats. Mixing throws when units mismatch.
- Network mismatch: passing a testnet `Coin` to a mainnet
  `TransactionBuilder` produces silent invalid signatures.
- `PSBT.AddCoins` is idempotent; the witness vs non-witness UTXO is
  inferred from the coin's scriptPubKey type. For legacy P2SH inputs
  attach the redeem script via `psbt.Inputs[i].RedeemScript`.
- BTCPay coexistence: if you embed NBitcoin in a process that also
  hosts BTCPay, pin the same NBitcoin major to avoid two copies of
  `Money` / `Network` in the AppDomain.
- `SetChange` on a builder fed manual outputs may add a second change
  output unexpectedly; prefer `SendAll`/`Send` + explicit change.

## References

- Repo: https://github.com/MetacoSA/NBitcoin
- Docs: https://github.com/MetacoSA/NBitcoin/blob/master/docs/index.md
- BIP174: https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki
- Companion: [../../infrastructure/btcpay/SKILL.md](../../infrastructure/btcpay/SKILL.md), [../../protocol/psbt/SKILL.md](../../protocol/psbt/SKILL.md)
