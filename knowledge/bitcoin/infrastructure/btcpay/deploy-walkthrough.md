# BTCPay Server Deploy Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/infrastructure/btcpay`.
> Canonical source: https://docs.btcpayserver.org/Deployment/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/infrastructure/btcpay/SKILL.md

## Concept

BTCPay Server is a multi-component .NET application stack: BTCPay itself,
NBXplorer (the wallet/utxo indexer), Bitcoin Core, an optional Lightning
node (LND/CLN/Eclair), and Postgres. The official "Docker Deployment"
script wires all of these via docker-compose with sensible defaults,
including Let's Encrypt TLS, Tor hidden service, and automated upgrades.
This article walks the canonical VPS deployment - DNS, sizing, the
`btcpay-setup.sh` flags that matter, and post-install verification.

## Walkthrough / mechanics

Server prerequisites:

- Linux x86_64 VPS with 8 GB RAM, 1 TB SSD, root SSH.
- A DNS A record pointing your domain to the VPS (`btcpay.example.com`).
- Open ports: 80, 443 inbound; 8333 (Bitcoin P2P) outbound; 9735 (LN)
  inbound for routing.
- Hostname set: `hostnamectl set-hostname btcpay.example.com`.

Install the deployment script:

```bash
sudo apt update && sudo apt install -y git curl
sudo mkdir -p /var/lib/btcpayserver
sudo chown -R $USER:$USER /var/lib/btcpayserver
cd /var/lib/btcpayserver
git clone https://github.com/btcpayserver/btcpayserver-docker
cd btcpayserver-docker
```

Configure environment:

```bash
export BTCPAY_HOST=btcpay.example.com
export NBITCOIN_NETWORK=mainnet
export BTCPAYGEN_CRYPTO1=btc
export BTCPAYGEN_LIGHTNING=lnd          # or "clightning"
export BTCPAYGEN_REVERSEPROXY=nginx
export BTCPAYGEN_ADDITIONAL_FRAGMENTS="opt-save-storage-s;opt-add-tor;opt-add-thunderhub"
export BTCPAY_ENABLE_SSH=true
export LETSENCRYPT_EMAIL=admin@example.com
```

Version floor (as of September 2026): deploy BTCPay Server >= 2.4.2 with
NBXplorer >= 2.6.10; 2.4.4 (7 September 2026) is current. Every release
before 2.4.2 (7 August 2026), release candidates included, let an
unauthenticated remote attacker read LND `.macaroon` files and take over
the node - BTCPay confirmed the flaw was exploited in the wild and funds
were stolen. BTCPay's own on-chain wallets (hot wallets included) and
CLN/Eclair backends were not exposed - but funds held in LND's internal
on-chain wallet are part of the affected node and may still be at risk -
and the `BTCPAYGEN_LIGHTNING=lnd` above puts this deployment squarely in
scope. Advisory:
https://blog.btcpayserver.org/security-advisory-btcpay-server-2-4-2/

Run the installer (as root for systemd integration):

```bash
sudo -E ./btcpay-setup.sh -i
```

What happens:

1. Generates `docker-compose-generator/docker-compose.generated.yml` from
   fragments.
2. Pulls images: bitcoin, nbxplorer, lnd, btcpayserver, postgres, nginx,
   tor, certbot, thunderhub.
3. Provisions Let's Encrypt cert for `btcpay.example.com`.
4. Creates a systemd service `btcpayserver.service` that controls the stack.
5. Starts everything; bitcoind begins IBD (~600 GB, 1-3 days).

Open `https://btcpay.example.com`, register the first user (becomes admin),
create a Store, attach an on-chain wallet (xpub or generate), connect the
internal LND node, set Lightning Address.

Useful CLI helpers under `btcpayserver-docker/`:

```bash
sudo bitcoin-cli.sh getblockchaininfo            # Bitcoin Core
sudo bitcoin-lncli.sh getinfo                    # LND
sudo btcpay-update.sh                            # update stack
sudo btcpay-down.sh                              # stop containers
sudo btcpay-up.sh                                # start containers
sudo btcpay-reverse-ssh.sh                       # access remotely
```

## Worked example

Verify a fresh deployment end-to-end. From the BTCPay UI:

1. Server Settings -> Maintenance -> "Restart" round-trips containers.
2. Create Store "Coffee Shop" -> Wallet -> Setup -> Use a public xpub
   from Sparrow:
   ```
   wpkh([7c0ce1f9/84'/0'/0']xpub6CUGRUonZSQ4...)
   ```
3. Save -> address verification page shows BIP21 URI for first receive
   address; cross-check in Sparrow.
4. Set Lightning -> Internal LND -> Save.
5. Create an invoice via API. `POST .../invoices` needs
   `btcpay.store.cancreateinvoice`, and the payment-methods read below
   needs `btcpay.store.canviewinvoices`. Since 2.4.2 (August 2026)
   Greenfield Basic authentication is disabled by default five minutes
   after account creation, so unless the admin account was created in the
   last five minutes the `curl -u` below is rejected - mint the key in
   the UI (Account -> API Keys) or opt Basic auth back in under account
   settings:

```bash
TOKEN=$(curl -s -u "admin@example.com:<pw>" \
   https://btcpay.example.com/api/v1/api-keys \
   -d '{"label":"deploy-test","permissions":["btcpay.store.cancreateinvoice","btcpay.store.canviewinvoices"]}' \
   -H 'Content-Type: application/json' | jq -r .apiKey)

curl -s -X POST https://btcpay.example.com/api/v1/stores/<STORE_ID>/invoices \
  -H "Authorization: token $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"amount":"5","currency":"USD","metadata":{"orderId":"TEST-1"}}' | jq
```

Expected: an invoice object with `checkoutLink`. The create response
carries no payment-method data at all, so fetch the destinations with a
second call:

```bash
curl -s https://btcpay.example.com/api/v1/invoices/<INVOICE_ID>/payment-methods \
  -H "Authorization: token $TOKEN" \
  | jq '.[] | {paymentMethodId, destination, paymentMethodFee}'
```

That returns a JSON *array*, one entry per enabled payment method: an
entry with `paymentMethodId: "BTC-CHAIN"` whose `destination` is an
on-chain address, and one with `paymentMethodId: "BTC-LN"` whose
`destination` is a BOLT11. Pay the LN invoice from a phone, watch the
BTCPay UI flip to "Settled" within seconds.

Greenfield 2.0 (30 October 2024) did the renaming here: `paymentMethod` ->
`paymentMethodId`, `cryptoCode` -> `currency`, `networkFee` ->
`paymentMethodFee`, and the ids `BTC-OnChain` -> `BTC-CHAIN`,
`BTC-LightningNetwork` -> `BTC-LN`, `BTC-LNURLPAY` -> `BTC-LNURL`. Old ids
are still accepted as input but never returned. The parallel
`paymentMethod` -> `payoutMethodId` rename hit the refund, payout and
payout-processor endpoints.

| Component | Container name | Port (host) |
|-----------|----------------|-------------|
| btcpayserver | btcpayserver_btcpayserver_1 | 49392 (internal) |
| nbxplorer | btcpayserver_nbxplorer_1 | 32838 (internal) |
| bitcoind | btcpayserver_bitcoind_1 | 8332 / 8333 |
| lnd | btcpayserver_lnd_bitcoin_1 | 8080 / 9735 |
| postgres | btcpayserver_postgres_1 | 5432 (internal) |
| nginx | letsencrypt-nginx-proxy-companion | 80 / 443 |
| tor | btcpayserver_tor_1 | 9050 / 9051 |

## Common pitfalls

- Insufficient disk - bitcoind alone is 700 GB+; allocate 1 TB minimum.
- DNS not propagated when running `btcpay-setup.sh` - certbot fails;
  re-run after DNS resolves.
- Running on a VPS with shared CPU - IBD can take a week; prefer dedicated
  cores for the first sync, scale down later.
- Forgetting Tor outbound - some hosts block outbound 9001/9050; if you
  enabled `opt-add-tor` ensure outbound is open or remove the fragment.
- Plugin compat lag - after a major BTCPay version bump, third-party
  plugins (Vault, Boltz, etc.) may temporarily break; consult the plugin
  page before upgrading.
- LND channel backups - back up `channel.backup` regularly; on-host it
  lives at `/var/lib/docker/volumes/.../lnd/data/chain/bitcoin/mainnet/`.
- Inheriting a pre-2.4.2 stack - update first, then rotate LND macaroons
  and audit the node for payments you did not make, unexpected channel
  closures and unfamiliar peers; reconcile on-chain and channel balances
  against your own records. If you cannot update immediately, take the
  server offline.
- External LN wallet access - 2.4.2 temporarily removed public LND API
  exposure on Docker deployments, so Zeus and similar wallets can no
  longer connect through the BTCPay domain or onion address; BTCPay plans
  to restore the option when it is safe to do so, and it is still removed
  as of September 2026.

## References

- Docker deployment: https://github.com/btcpayserver/btcpayserver-docker
- Setup docs: https://docs.btcpayserver.org/Deployment/
- Manual deployment: https://docs.btcpayserver.org/Deployment/ManualDeployment/
- Greenfield API: https://docs.btcpayserver.org/API/Greenfield/v1/
- Greenfield 2.0 breaking changes: https://github.com/btcpayserver/btcpayserver/issues/5964
- 2.4.2 security advisory: https://blog.btcpayserver.org/security-advisory-btcpay-server-2-4-2/
