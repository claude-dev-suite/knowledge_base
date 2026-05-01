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
5. Create an invoice via API:

```bash
TOKEN=$(curl -s -u "admin@example.com:<pw>" \
   https://btcpay.example.com/api/v1/api-keys \
   -d '{"label":"deploy-test","permissions":["btcpay.store.canviewstoresettings"]}' \
   -H 'Content-Type: application/json' | jq -r .apiKey)

curl -s -X POST https://btcpay.example.com/api/v1/stores/<STORE_ID>/invoices \
  -H "Authorization: token $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"amount":"5","currency":"USD","metadata":{"orderId":"TEST-1"}}' | jq
```

Expected: invoice with `checkoutLink`, `paymentMethods.BTC.destination`
(an on-chain address) and `paymentMethods.BTC-LightningNetwork.destination`
(a BOLT11). Pay the LN invoice from a phone, watch the BTCPay UI flip to
"Settled" within seconds.

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

## References

- Docker deployment: https://github.com/btcpayserver/btcpayserver-docker
- Setup docs: https://docs.btcpayserver.org/Deployment/
- Manual deployment: https://docs.btcpayserver.org/Deployment/ManualDeployment/
- Greenfield API: https://docs.btcpayserver.org/API/Greenfield/v1/
