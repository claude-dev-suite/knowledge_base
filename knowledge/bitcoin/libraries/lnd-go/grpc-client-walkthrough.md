# LND gRPC Client (Go) Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/lnd-go`.
> Canonical source: https://github.com/lightningnetwork/lnd
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/lnd-go/SKILL.md

## Concept

LND ships a gRPC API split into the core `lnrpc` service plus
optional sub-servers (`walletrpc`, `routerrpc`, `chainrpc`, `signrpc`,
`invoicesrpc`, `wtclientrpc`, ...). Authentication uses TLS plus a
macaroon (HMAC-bearer token). The Go client packages live alongside
LND itself, which is why integrating from Go is the most ergonomic
path -- types match LND exactly, no codegen mismatch, and the
`rpcclient`-shaped helpers compose cleanly with btcsuite.

You typically build one `*grpc.ClientConn` and reuse it across many
typed clients (`lnrpc.NewLightningClient`, `routerrpc.NewRouterClient`,
etc.) so a single TCP/TLS session multiplexes all calls.

## API walkthrough

```go
package main

import (
    "context"
    "encoding/hex"
    "io/ioutil"
    "log"

    "github.com/lightningnetwork/lnd/lnrpc"
    "github.com/lightningnetwork/lnd/macaroons"
    macaroon "gopkg.in/macaroon.v2"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials"
)

func dial(host, tlsPath, macPath string) (*grpc.ClientConn, error) {
    creds, err := credentials.NewClientTLSFromFile(tlsPath, "")
    if err != nil { return nil, err }

    macBytes, err := ioutil.ReadFile(macPath)
    if err != nil { return nil, err }
    mac := &macaroon.Macaroon{}
    if err := mac.UnmarshalBinary(macBytes); err != nil { return nil, err }
    macCred, err := macaroons.NewMacaroonCredential(mac)
    if err != nil { return nil, err }

    return grpc.Dial(host,
        grpc.WithTransportCredentials(creds),
        grpc.WithPerRPCCredentials(macCred),
    )
}

func main() {
    conn, err := dial("localhost:10009",
        "/home/lnd/.lnd/tls.cert",
        "/home/lnd/.lnd/data/chain/bitcoin/mainnet/admin.macaroon")
    if err != nil { log.Fatal(err) }
    defer conn.Close()

    client := lnrpc.NewLightningClient(conn)
    info, err := client.GetInfo(context.Background(), &lnrpc.GetInfoRequest{})
    if err != nil { log.Fatal(err) }
    log.Printf("alias=%s pubkey=%s height=%d",
        info.Alias, info.IdentityPubkey, info.BlockHeight)
}
```

## Worked example: stream invoices and pay one

Server-streaming RPC: subscribe to invoice events, then send a payment
when one arrives.

```go
import (
    "context"
    "io"

    "github.com/lightningnetwork/lnd/lnrpc"
    "github.com/lightningnetwork/lnd/lnrpc/routerrpc"
)

func watchInvoices(ctx context.Context, ln lnrpc.LightningClient) error {
    sub, err := ln.SubscribeInvoices(ctx, &lnrpc.InvoiceSubscription{})
    if err != nil { return err }
    for {
        inv, err := sub.Recv()
        if err == io.EOF { return nil }
        if err != nil { return err }
        if inv.State == lnrpc.Invoice_SETTLED {
            log.Printf("settled: %s amt_msat=%d", inv.RHash, inv.AmtPaidMsat)
        }
    }
}

func payInvoice(ctx context.Context, conn *grpc.ClientConn,
                bolt11 string) error {
    router := routerrpc.NewRouterClient(conn)
    stream, err := router.SendPaymentV2(ctx, &routerrpc.SendPaymentRequest{
        PaymentRequest: bolt11,
        TimeoutSeconds: 60,
        FeeLimitSat:    20,
    })
    if err != nil { return err }
    for {
        upd, err := stream.Recv()
        if err == io.EOF { return nil }
        if err != nil { return err }
        log.Printf("payment status: %s", upd.Status)
        if upd.Status == lnrpc.Payment_SUCCEEDED ||
            upd.Status == lnrpc.Payment_FAILED {
            return nil
        }
    }
}
```

`SendPaymentV2` is preferred over the older `SendPaymentSync` -- it
streams payment progress and exposes proper failure reasons.

## Common pitfalls

- Macaroon scope: `admin.macaroon` is full-access. For services that
  only pay or only receive, bake a custom macaroon with `lncli bakemacaroon`
  and ship that instead.
- TLS hostname mismatch: when LND regenerates `tls.cert` you must
  redistribute it; cached certs in clients will fail validation.
- Streaming RPCs and goroutines: each `SubscribeInvoices` call holds a
  long-lived stream. Use `context.WithCancel` on shutdown to release
  it cleanly.
- Connection multiplexing: don't open a fresh `grpc.Dial` per request;
  reuse the connection across all sub-server clients.
- gRPC max message size defaults to 4 MiB; channel/route lookups can
  exceed it. Add `grpc.WithDefaultCallOptions(grpc.MaxCallRecvMsgSize(50<<20))`
  if calls fail with `received message larger than max`.

## References

- Repo: https://github.com/lightningnetwork/lnd
- API docs: https://lightning.engineering/api-docs
- routerrpc: https://lightning.engineering/api-docs/api/lnd/router/send-payment-v2/
- Companion: [../../lightning/lnd/SKILL.md](../../lightning/lnd/SKILL.md), [btcsuite/SKILL.md](../btcsuite/SKILL.md)
