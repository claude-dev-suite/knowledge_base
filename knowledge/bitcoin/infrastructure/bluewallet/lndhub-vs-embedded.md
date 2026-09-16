# BlueWallet Lightning: Self-Hosted LndHub vs Ark/Arkade - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/infrastructure/bluewallet`.
> Canonical source: https://bluewallet.io/lightning/
> Status as of September 2026: BlueWallet's hosted LndHub node
> (`lndhub.io`) was announced for sunset on 23 February 2023 and shut
> down on 30 April 2023. There is no "LndHub (default)" mode any more,
> and BlueWallet has no on-device LND. This article previously compared
> those two modes; it now compares self-hosted LndHub against the
> Ark/Arkade wallet that shipped in v7.2.2 (November 2025).
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/infrastructure/bluewallet/SKILL.md

## Concept

BlueWallet offers two Lightning paths, radically different from a custody
and trust standpoint. **LndHub** treats the BlueWallet app as a thin
client to a remote LND node managed by someone - since 30 April 2023 that
someone is never BlueWallet, because the hosted node is gone and the app
ships no default hub. **Ark/Arkade** is a self-custodial single-key Ark
wallet added in v7.2.2 (24 November 2025) that reaches Lightning through
Boltz submarine and reverse swaps; its keys and VTXOs stay on the device.
This article compares the two precisely - protocol, UX, custody, recovery,
and when each is appropriate - and shows how to self-host LndHub.

One correction worth stating plainly, because it circulates widely:
BlueWallet has **no** embedded / on-device LND mode. The v8.0.1 (July
2026) source tree contains no Neutrino module, no lnd-mobile module and
no embedded-node wallet type; `class/wallets/` holds
`lightning-custodian-wallet.ts` (LndHub) and `lightning-ark-wallet.ts`
and nothing else off-chain. Descriptions of a phone-resident LND with a
~1.5 GB header sync are not describing BlueWallet.

## Walkthrough / mechanics

LndHub model:

- A REST server - yours; the app has shipped no default hub URI since
  the hosted node closed - exposes per-user endpoints `/auth`,
  `/balance`, `/addinvoice`, `/payinvoice`, `/getuserinvoices`,
  `/gettxs`.
- A single shared LND node behind LndHub holds all channels.
- Users authenticate with a `lndhub://login:password@host` URL stored
  locally; this URL **is** the credential to spend the user's sub-account.
- Funds are an IOU from the LndHub operator to the user. If you run the
  hub, that operator is you.
- In `screen/wallets/Add.tsx` the Lightning (LNDhub) wallet button is
  rendered only when a hub URI is already stored (`hasStoredLndHub`) or
  the type is already selected.

Ark/Arkade model (v7.2.2, November 2025 onward):

- `class/wallets/lightning-ark-wallet.ts` builds a `SingleKey` Ark
  wallet on `@arkade-os/sdk`, derived from a BIP39 seed held on the
  device - self-custodial, no operator IOU.
- Value lives in VTXOs under an Ark server rather than in LN channels,
  so there is no channel backup, no inbound-liquidity purchase and no
  force-close path; the exit path is unilateral redemption on-chain.
- Lightning interop is swap-based: `@arkade-os/boltz-swap` performs
  submarine swaps to pay a BOLT11 invoice and reverse swaps to receive.
- A per-network delegate service is configured (`delegate.arkade.money`
  on mainnet); networks without a delegator skip `delegatorProvider`
  entirely.
- Gated in v8.0.1 (July 2026): the "LightningArk" button in the
  Add-wallet screen renders only after `backdoorPressed >= 20`.

Network architecture diagrams:

```
LndHub (self-hosted):
  [BlueWallet app] --HTTPS--> [your LndHub] --gRPC--> [your LND] -> LN

Ark/Arkade:
  [BlueWallet app + Ark SingleKey] --REST--> [Ark server / delegator]
                                   --Boltz swap--> Lightning
```

API spec for LndHub (sample request):

```bash
HUB=https://lndhub.example.com   # your own hub; there is no default
# 1. Get tokens from refresh credential
TOKENS=$(curl -s -X POST $HUB/auth?type=auth \
  -H 'Content-Type: application/json' \
  -d '{"login":"abc","password":"def"}')
ACCESS=$(echo "$TOKENS" | jq -r .access_token)

# 2. Add invoice
curl -s -X POST $HUB/addinvoice \
  -H "Authorization: Bearer $ACCESS" \
  -H "Content-Type: application/json" \
  -d '{"amt":1000,"memo":"coffee"}'
# {"r_hash":{"data":[..]},"payment_request":"lnbc10u1p...","add_index":42}

# 3. Pay invoice
curl -s -X POST $HUB/payinvoice \
  -H "Authorization: Bearer $ACCESS" \
  -d '{"invoice":"lnbc..."}'
```

Self-host LndHub (Node.js):

```bash
git clone https://github.com/BlueWallet/LndHub
cd LndHub
docker compose up -d   # runs LndHub + Redis; assumes LND elsewhere
```

`config.js`:

```js
module.exports = {
  bitcoind: { rpc: "http://127.0.0.1:8332", user: "x", password: "y" },
  lnd: { url: "127.0.0.1:10009", macaroonHex: "0201036c6e64...",
         certHex: "2d2d2d2d2d424547494e..." },
  redis: { host: "127.0.0.1", port: 6379 }
};
```

In BlueWallet, "Add wallet -> Lightning" -> enter your LndHub HTTPS
endpoint in the LNDHub field. As of v8.0.1 (July 2026) that button is
hidden until a hub URI has been stored, so set the hub in Settings
first if the Lightning type does not appear.

## Worked example

Compare a 1000-sat send between the two modes from app launch:

Self-hosted LndHub (typical 2-3s end-to-end):

```
T+0  : User taps "Send 1000 sats"
T+0.1: App requests bearer token via /auth
T+0.3: App POSTs /payinvoice
T+0.4: LndHub backend submits to shared LND
T+1.5: LND finds route and pays
T+1.8: LndHub returns 200, app shows success
```

Ark/Arkade (swap-mediated; no on-device routing):

```
T+0  : User taps "Send 1000 sats"
T+.. : boltz-swap decodes BOLT11, opens a submarine swap
T+.. : VTXO is spent to the swap script under the Ark server
T+.. : Boltz pays the invoice on Lightning, reveals the preimage
T+.. : Preimage claims the swap output; failure path is a refund
```

(No timings given for the Ark path - none are published and the flow is
dominated by Boltz and Ark round timing.)

| Aspect | LndHub (self-hosted) | Ark/Arkade (v7.2.2+) |
|--------|----------------------|----------------------|
| Custody | self (you operate hub) | self (key on device) |
| Initial sync | none (server-side) | none |
| Disk usage | minimal | minimal (Realm store) |
| Battery impact | minimal | low |
| Channel ownership | hub's | none - VTXOs, not channels |
| Inbound liquidity | hub's | swap capacity, not channel inbound |
| Recovery | restore hub credential | BIP39 seed |
| Fees | hub's choice | Boltz swap fee + Ark server fees |
| Censorship resistance | medium | medium (Ark server + Boltz) |
| Availability in app | any version | tap-gated as of v8.0.1 (July 2026) |

## Common pitfalls

- Assuming a hosted fallback still exists - it does not. Since 30 April
  2023 a BlueWallet Lightning wallet points at a hub someone runs; if
  that is not you, it is custodial for you, and a lost or seized server
  means lost funds. The UI does not loudly distinguish.
- Sharing the `lndhub://...` URL - this is the credential. Anyone who
  scans the export QR can spend.
- Following guides that describe an "Embedded LND" mode, SCB restore or
  on-device Neutrino sync in BlueWallet - no such mode exists (checked
  against the v8.0.1 tree, September 2026). Those instructions belong to
  other wallets (Zeus, Breez) and will not map onto BlueWallet.
- Expecting watchtower settings - `watchtower` appears nowhere in the
  v8.0.1 tree (checked September 2026).
- Self-hosted LndHub TLS - default config uses HTTP; expose only behind
  HTTPS reverse proxy (Caddy + Let's Encrypt is one line).
- Treating the Ark wallet as a supported feature - as of v8.0.1 (July
  2026) it is reachable only through a 20-tap gate in the Add-wallet
  screen, so treat it as experimental and do not route user funds to it
  on the strength of the release notes alone.

## References

- LndHub repo: https://github.com/BlueWallet/LndHub
- LndHub API: https://github.com/BlueWallet/LndHub/blob/master/doc/Send-requirements.md
- BlueWallet docs: https://bluewallet.io/docs
- BlueWallet Lightning page (self-hosted hub only): https://bluewallet.io/lightning/
- Hosted LndHub sunset (announced 23 Feb 2023, off 30 Apr 2023):
  https://bitcoinmagazine.com/business/bluewallet-to-sunset-custodial-lightning-wallet-service
- Ark wallet added, v7.2.2: https://github.com/BlueWallet/BlueWallet/pull/8142
- BlueWallet 8.0.1 release notes (21 July 2026): https://github.com/BlueWallet/BlueWallet/releases/tag/8.0.1
- Static Channel Backup - an LND feature, listed only for context on
  what the mistaken "embedded LND in BlueWallet" guides describe;
  BlueWallet itself has no SCB path:
  https://docs.lightning.engineering/lightning-network-tools/lnd/scb
