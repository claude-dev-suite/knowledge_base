# tapd gRPC Flow - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/tapd-go`.
> Canonical source: https://github.com/lightninglabs/taproot-assets
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/tapd-go/SKILL.md

## Concept

`tapd` is the Lightning Labs daemon implementing the Taproot Assets
Protocol (TAP, formerly "Taro"). It runs alongside LND, sharing
LND's wallet for on-chain BTC and using LND's signer for asset
spends. It exposes its own gRPC API (`taprpc` and friends) for asset
issuance, transfer, balance queries, and -- combined with LND -- 
asset-aware Lightning channels.

For a Go client, the integration pattern is the same as LND: TLS +
macaroon to authenticate, one gRPC connection multiplexing many
typed clients (`taprpc.TaprootAssetsClient`, `mintrpc.MintClient`,
`assetwalletrpc.AssetWalletClient`, ...).

## API walkthrough

```go
package main

import (
    "context"
    "io/ioutil"
    "log"

    "github.com/lightninglabs/taproot-assets/taprpc"
    "github.com/lightninglabs/taproot-assets/taprpc/mintrpc"
    "github.com/lightningnetwork/lnd/macaroons"
    macaroon "gopkg.in/macaroon.v2"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials"
)

func dialTapd(host, tlsPath, macPath string) (*grpc.ClientConn, error) {
    creds, err := credentials.NewClientTLSFromFile(tlsPath, "")
    if err != nil { return nil, err }
    raw, err := ioutil.ReadFile(macPath)
    if err != nil { return nil, err }
    mac := &macaroon.Macaroon{}
    if err := mac.UnmarshalBinary(raw); err != nil { return nil, err }
    macCred, err := macaroons.NewMacaroonCredential(mac)
    if err != nil { return nil, err }

    return grpc.Dial(host,
        grpc.WithTransportCredentials(creds),
        grpc.WithPerRPCCredentials(macCred))
}

func main() {
    conn, err := dialTapd("localhost:10029",
        "/home/tapd/.tapd/tls.cert",
        "/home/tapd/.tapd/data/mainnet/admin.macaroon")
    if err != nil { log.Fatal(err) }
    defer conn.Close()

    tap := taprpc.NewTaprootAssetsClient(conn)
    info, err := tap.GetInfo(context.Background(), &taprpc.GetInfoRequest{})
    if err != nil { log.Fatal(err) }
    log.Printf("network=%s lnd_version=%s", info.Network, info.LndVersion)
}
```

## Worked example: mint, address, send

Minting goes through a batch (`MintAsset` queues, `FinalizeBatch`
mines the genesis tx into a Taproot output):

```go
import (
    "context"

    "github.com/lightninglabs/taproot-assets/taprpc"
    "github.com/lightninglabs/taproot-assets/taprpc/mintrpc"
)

func mintAndSend(ctx context.Context, conn *grpc.ClientConn,
                 destAddr string) error {
    tap   := taprpc.NewTaprootAssetsClient(conn)
    minter := mintrpc.NewMintClient(conn)

    // 1. Queue a mint
    _, err := minter.MintAsset(ctx, &mintrpc.MintAssetRequest{
        Asset: &mintrpc.MintAsset{
            AssetType: taprpc.AssetType_NORMAL,
            Name:      "USD-Test",
            AssetMeta: &taprpc.AssetMeta{
                Data: []byte("issuer=demo,supply=1000000"),
            },
            Amount: 1_000_000,
        },
    })
    if err != nil { return err }

    // 2. Finalize the batch (broadcasts the genesis tx)
    fin, err := minter.FinalizeBatch(ctx, &mintrpc.FinalizeBatchRequest{})
    if err != nil { return err }
    log.Printf("genesis batch txid: %s", fin.Batch.BatchTxid)

    // 3. Send to a recipient address (out-of-band string)
    res, err := tap.SendAsset(ctx, &taprpc.SendAssetRequest{
        TapAddrs: []string{destAddr},
    })
    if err != nil { return err }
    log.Printf("transfer txid: %s", res.Transfer.AnchorTxHash)
    return nil
}
```

To **receive** assets, the recipient generates a `taprpc.Addr` via
`NewAddr(...)` and shares the bech32m string with the sender.

## Common pitfalls

- `tapd` macaroons are **separate** from LND's. The default
  `~/.tapd/data/.../admin.macaroon` is required for tapd RPCs even when
  the underlying LND is using its own.
- Universe servers (the discovery/proof layer) need to be reachable
  for senders to validate proofs. Run a local universe or rely on
  `universe.lightning.finance`.
- Asset proofs grow with transfer history; for high-frequency assets
  consider proof courier services (BIP-style overlay) rather than
  inline gossip.
- Asset-aware Lightning channels need both peers running `tapd` plus
  a compatible LND; mismatched versions silently fall back to BTC
  routing.
- API churn: `taprpc` is still pre-1.0; pin Go module versions
  exactly and read release notes between bumps.

## References

- Repo: https://github.com/lightninglabs/taproot-assets
- Spec (TAP BIPs): https://github.com/Roasbeef/bips/blob/bip-tap/bip-taro.mediawiki
- API reference: https://lightning.engineering/api-docs/api/taproot-assets
- Companion: [../../l2/taproot-assets/SKILL.md](../../l2/taproot-assets/SKILL.md), [lnd-go/SKILL.md](../lnd-go/SKILL.md)
