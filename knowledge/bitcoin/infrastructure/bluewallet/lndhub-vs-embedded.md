# BlueWallet LndHub vs Embedded LND - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/infrastructure/bluewallet`.
> Canonical source: https://bluewallet.io/lndhub/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/infrastructure/bluewallet/SKILL.md

## Concept

BlueWallet offers two Lightning modes that are functionally similar from
the user's perspective but radically different from a custody and trust
standpoint. **LndHub** treats the BlueWallet app as a thin client to a
remote LND node managed by someone else (BlueWallet team or a self-hosted
hub). **Embedded LND** runs an actual LND node on the user's phone, so the
private keys, channels, and channel state never leave the device. This
article compares the two precisely - protocol, UX, custody, recovery, and
when each is appropriate - and shows how to self-host LndHub to recover
some custody benefits without committing to a phone-resident node.

## Walkthrough / mechanics

LndHub model:

- A REST server (`https://lndhub.io/` by default) exposes per-user
  endpoints `/auth`, `/balance`, `/addinvoice`, `/payinvoice`,
  `/getuserinvoices`, `/gettxs`.
- A single shared LND node behind LndHub holds all channels.
- Users authenticate with a `lndhub://login:password@host` URL stored
  locally; this URL **is** the credential to spend the user's sub-account.
- Funds are an IOU from the LndHub operator to the user.

Embedded LND model:

- Phone runs a stripped-down LND build (Neutrino light-client mode by
  default to avoid running a full bitcoind on-device).
- Channel state, peer connections, gossip data all live in the app's
  sandbox.
- Initial sync downloads the chain headers and filters (~1.5 GB) before
  channels can open.
- Backup is a 24-word seed plus channel.backup file; restoration reopens
  channels via SCB (Static Channel Backup).

Network architecture diagrams:

```
LndHub:
  [BlueWallet app] --HTTPS--> [LndHub server] --gRPC--> [shared LND] -> LN

Embedded:
  [BlueWallet app + LND] --P2P--> Lightning peers
                          --Neutrino--> Bitcoin full nodes
```

API spec for LndHub (sample request):

```bash
HUB=https://lndhub.io
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

In BlueWallet, "Add wallet -> Lightning Custodial -> Server URI" -> point
at your LndHub HTTPS endpoint.

## Worked example

Compare a 1000-sat send between the two modes from app launch:

LndHub (typical 2-3s end-to-end):

```
T+0  : User taps "Send 1000 sats"
T+0.1: App requests bearer token via /auth
T+0.3: App POSTs /payinvoice
T+0.4: LndHub backend submits to shared LND
T+1.5: LND finds route and pays
T+1.8: LndHub returns 200, app shows success
```

Embedded LND (5-15s and only if channels are funded + online):

```
T+0  : User taps "Send 1000 sats"
T+0.1: Embedded LND parses BOLT11
T+0.5: Pathfind across local channel graph
T+1.0: HTLC initiated on outbound channel
T+...: Possibly retry on failed routes
T+5.0: Preimage received, success
```

| Aspect | LndHub (default) | LndHub (self-hosted) | Embedded LND |
|--------|------------------|----------------------|--------------|
| Custody | custodial | self (you operate hub) | self |
| Initial sync | none | none (server-side) | 1.5 GB headers |
| Disk usage | minimal | minimal | 2 GB+ |
| Battery impact | minimal | minimal | high |
| Channel ownership | hub's | hub's | user's |
| Inbound liquidity | hub's | hub's | user's burden |
| Recovery | restore login URL | restore hub credential | seed + channel.backup |
| Fees | 0 (hub absorbs) | hub's choice | network fees + on-chain anchors |
| Censorship resistance | low | medium | high |
| UX speed | fastest | fast | slowest |

## Common pitfalls

- Treating BlueWallet's default LN wallet as self-custodial - the hosted
  LndHub is custodial; lost or seized server = lost funds. UI does not
  loudly distinguish.
- Sharing the `lndhub://...` URL - this is the credential. Anyone who
  scans the export QR can spend.
- Embedded LND on iOS background limits - iOS aggressively suspends apps;
  channels can be force-closed by counterparty if the user doesn't open
  the app for weeks. Watchtowers (Eye of Satoshi etc.) help.
- Embedded LND restore: requires SCB + on-chain reserve to claim closed
  channels - if the seed is restored on a fresh device, force-close all
  channels and recover on-chain (you lose any zero-conf or pending HTLCs).
- Self-hosted LndHub TLS - default config uses HTTP; expose only behind
  HTTPS reverse proxy (Caddy + Let's Encrypt is one line).
- Inbound liquidity for embedded mode - new channels are outbound-only;
  to receive, user must request inbound (paid liquidity) or use a
  liquidity service like Lightning Loop / LSP.

## References

- LndHub repo: https://github.com/BlueWallet/LndHub
- LndHub API: https://github.com/BlueWallet/LndHub/blob/master/doc/Send-requirements.md
- BlueWallet docs: https://bluewallet.io/docs
- Static Channel Backup: https://docs.lightning.engineering/lightning-network-tools/lnd/scb
