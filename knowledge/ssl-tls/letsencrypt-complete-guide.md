# Let's Encrypt & Certbot: Complete Production Guide

> Written from the perspective of a senior sysadmin who has managed hundreds of certificate lifecycles across bare-metal, cloud, and containerized environments. This guide is opinionated because ambiguity kills production systems.

---

## Table of Contents

1. [What Let's Encrypt Actually Is](#what-lets-encrypt-actually-is)
2. [ACME Challenge Types](#acme-challenge-types)
3. [Rate Limits — Read This Before Touching Production](#rate-limits)
4. [Staging Environment — Non-Negotiable First Step](#staging-environment)
5. [Installing Certbot](#installing-certbot)
6. [Certbot Modes](#certbot-modes)
7. [DNS Plugins for Wildcard Certificates](#dns-plugins-for-wildcard-certificates)
8. [Multi-Domain SANs](#multi-domain-sans)
9. [Certificate Files Explained](#certificate-files-explained)
10. [Renewal](#renewal)
11. [Pre/Post and Deploy Hooks](#prepost-and-deploy-hooks)
12. [Revocation](#revocation)
13. [Server Migration](#server-migration)
14. [Troubleshooting](#troubleshooting)
15. [Hardening and Security Notes](#hardening-and-security-notes)
16. [Quick Reference Cheatsheet](#quick-reference-cheatsheet)

---

## What Let's Encrypt Actually Is

Let's Encrypt is a free, automated, and open Certificate Authority (CA) run by the Internet Security Research Group (ISRG). It issues Domain Validation (DV) certificates — it does not issue OV or EV certificates. If you need organization or extended validation, you need a paid CA. For 99% of web infrastructure, DV is exactly what you need.

Certificates are valid for **90 days**. This is intentional. Short lifetimes force automation and reduce exposure windows from compromised keys. Do not treat this as a bug. Automate renewal from day one.

The protocol used is **ACME** (Automatic Certificate Management Environment), standardized as RFC 8555. Any ACME-compatible client works with Let's Encrypt — Certbot is the official EFF-maintained client and the one this guide uses, but `acme.sh`, `Caddy` (built-in), `Traefik` (built-in), and others all work.

**Let's Encrypt CA Roots:**
- Production: ISRG Root X1, ISRG Root X2
- Intermediate: R10, R11, E5, E6 (rotated periodically)
- Staging: (STAGING) Pretend Pear X1 — NOT trusted by browsers

---

## ACME Challenge Types

The ACME protocol requires you to prove you control the domain before a certificate is issued. There are three challenge types. Picking the wrong one for your situation will cause you pain.

### HTTP-01 (The Default — Use This Unless You Have a Reason Not To)

**How it works:**

The CA sends a random token. Your ACME client writes a file at:
```
http://<domain>/.well-known/acme-challenge/<token>
```
The CA's validation servers fetch that URL over plain HTTP (port 80) from multiple vantage points globally. If the file is there and the content matches, the challenge passes.

**Requirements:**
- Port 80 must be open and publicly reachable
- The domain must resolve to the server running certbot
- No HTTPS redirect interference during validation (or a proper redirect chain that certbot can follow)
- Works for a single domain or multiple explicit domains — NOT wildcards

**Use HTTP-01 when:**
- You are on a public-facing server with port 80 accessible
- You need simplicity and reliability
- You are managing a handful of explicit FQDNs
- You are using the `webroot`, `standalone`, `nginx`, or `apache` plugins

**Do NOT use HTTP-01 when:**
- You need a wildcard certificate (`*.example.com`) — HTTP-01 cannot issue wildcards, full stop
- Your server has no public IP (internal/air-gapped networks)
- Your firewall blocks port 80 inbound from Let's Encrypt validation servers (which are globally distributed — you cannot whitelist them reliably)
- You are behind a load balancer that cannot route `.well-known/acme-challenge/` traffic predictably

**Multi-vantage-point validation note (as of 2024):** Let's Encrypt validates from multiple geographic perspectives simultaneously. Inconsistent DNS or temporary port 80 blocks that used to pass occasionally will now fail more reliably. This is a good thing — it catches split-horizon and BGP hijacking issues.

---

### DNS-01 (Required for Wildcards, Best for Internal Servers)

**How it works:**

The CA provides a token. Your ACME client creates a DNS TXT record at:
```
_acme-challenge.<domain>  TXT  "<token-value>"
```
The CA queries DNS for that record from multiple resolvers. If the record exists and matches, the challenge passes. The token changes every issuance, so you need DNS API access for automation.

**Requirements:**
- DNS API access (Route 53, Cloudflare, GoDaddy, etc.) OR manual DNS edits
- Patience for DNS propagation (30s–48h depending on TTL and provider)
- No public port requirements — server can be completely firewalled

**Use DNS-01 when:**
- You need `*.example.com` — this is the ONLY way to get wildcard certificates
- Your server has no public IP or port 80 is blocked
- You want to issue certificates for internal/private hostnames that still resolve publicly (intranet servers with public DNS)
- You are issuing certificates from a separate "cert management" server that doesn't host the actual services
- You need to centralize certificate issuance away from individual app servers

**The manual DNS-01 flow (do not use in production):**
```bash
certbot certonly --manual --preferred-challenges dns -d "*.example.com" -d "example.com"
```
This will pause and ask you to add a TXT record. It cannot auto-renew. It is useful for one-time testing only.

**DNS propagation is the main pain point.** Set your DNS TTL to 60 seconds before doing anything with DNS-01 automation. A 3600-second TTL on `_acme-challenge` will cause renewal failures if your DNS API and the CA's resolver see different values.

---

### TLS-ALPN-01 (Niche — Only When Port 443 Is All You Have)

**How it works:**

The CA initiates a TLS connection to port 443 of the domain and negotiates with `acme-tls/1` as the ALPN protocol identifier. The server responds with a self-signed certificate containing a special `acmeValidation-v1` extension with the challenge token. If the CA reads it, the challenge passes.

**Requirements:**
- Port 443 must be publicly reachable
- Your web server or ACME client must handle the `acme-tls/1` ALPN negotiation
- Port 80 is NOT required
- Wildcards are NOT supported

**Use TLS-ALPN-01 when:**
- Port 80 is genuinely unavailable (some hosting providers disable it)
- Port 443 is open and you can configure your server to handle the special ALPN exchange
- You cannot use DNS-01 for some reason

**Why TLS-ALPN-01 is rarely the right answer:**
- Certbot's `standalone` mode supports it via `--preferred-challenges tls-sni` (deprecated in older versions) but actual `tls-alpn-01` support requires specific server configuration
- If port 443 is open, port 80 is almost certainly also controllable — just open it for validation
- Limited plugin ecosystem compared to HTTP-01 and DNS-01
- Most people choose this when they should have chosen DNS-01

**Bottom line:** HTTP-01 for almost everything. DNS-01 for wildcards and private networks. TLS-ALPN-01 only when you genuinely can't use either of the others.

---

## Rate Limits

**Read this section before issuing any production certificate.** Hitting rate limits and being locked out of your domain for a week is an avoidable catastrophe.

### Primary Limits (as of 2025)

| Limit | Value | Notes |
|---|---|---|
| Certificates per Registered Domain per week | 5 | This is the killer one. "Registered domain" means `example.com` regardless of subdomain. |
| Duplicate Certificate limit | 5 per week | Same set of SANs issued again — counts separately |
| Failed Validations | 5 per account per hostname per hour | Too many failed attempts = temporary ban |
| New Orders | 300 per 3 hours per account | Rarely hit unless you're automating badly |
| SANs per Certificate | 100 | Max unique domains on one certificate |
| Accounts per IP | 10 per 3 hours | Relevant for shared IPs / NAT |
| New Registrations per IP | 10 per 3 hours | |
| Pending Authorizations | 300 per account | |

### The Registered Domain Rule

This is what trips everyone up. Let's Encrypt counts against the **registered domain** (the public suffix + one label), not the subdomain. So:

- `example.com` — registered domain: `example.com`
- `www.example.com` — registered domain: `example.com`
- `api.example.com` — registered domain: `example.com`
- `staging.api.example.com` — registered domain: `example.com`

All four count against the same weekly limit of 5 certificates. If you're running CI/CD that issues a new cert per deployment without thinking, you will exhaust this in an afternoon.

### Renewals Are Exempt

Renewing an existing certificate (same set of SANs, same account) does **not** count against the 5 certificates/week limit. Only new issuances count. This is why cert rotation is safe to automate aggressively.

### Staging Has Separate (Higher) Limits

The staging environment has much higher limits (see [Staging Environment](#staging-environment) section). Always test in staging. Always.

### What to Do If You Hit Rate Limits

You wait. There is no bypass, no support ticket that will lift it, no workaround. The limits reset on a rolling weekly window. To see your current usage, use the [crt.sh](https://crt.sh) transparency log viewer — search your domain to see all issued certificates and their timestamps.

```bash
# Check existing certs for your domain via CT logs
curl -s "https://crt.sh/?q=%.example.com&output=json" | jq '.[].not_before' | sort | uniq -c
```

---

## Staging Environment — Non-Negotiable First Step

Let's Encrypt's staging environment uses a fake CA that issues untrusted certificates. The certificates look and behave identically to production certificates but are NOT trusted by browsers or operating systems.

**Staging ACME URL:** `https://acme-staging-v02.api.letsencrypt.org/directory`

**Why staging is mandatory for any new setup:**
1. You will make mistakes the first time. Staging mistakes don't consume production rate limits.
2. DNS misconfigurations, firewall issues, and plugin credential errors all surface in staging identically to production.
3. Staging has far more lenient rate limits — effectively unlimited for testing purposes.

**Always run staging first:**
```bash
certbot certonly --staging \
  --webroot -w /var/www/html \
  -d example.com \
  -d www.example.com \
  --email admin@example.com \
  --agree-tos \
  --no-eff-email
```

**When staging passes**, run the exact same command without `--staging`:
```bash
certbot certonly \
  --webroot -w /var/www/html \
  -d example.com \
  -d www.example.com \
  --email admin@example.com \
  --agree-tos \
  --no-eff-email
```

**Staging certificate CN:** The CN will contain "Fake LE Intermediate X1" or similar. If you ever see this in a production system, someone skipped staging and didn't clean up.

---

## Installing Certbot

### Snap — The Recommended Method (Ubuntu/Debian/Fedora/RHEL)

Snap is the officially recommended installation method as of 2021. The EFF deprecated OS package repositories for Certbot because they lagged behind releases and caused version skew issues.

```bash
# Remove any OS-packaged certbot first
sudo apt remove certbot  # Debian/Ubuntu
sudo dnf remove certbot  # RHEL/Fedora

# Install snapd if not present
sudo apt install snapd   # Debian/Ubuntu
sudo dnf install snapd   # RHEL/Fedora

# Ensure snapd is current
sudo snap install core
sudo snap refresh core

# Install certbot
sudo snap install --classic certbot

# Create the symlink so certbot is in PATH
sudo ln -s /snap/bin/certbot /usr/local/bin/certbot

# Verify
certbot --version
```

### Why Snap Over apt/dnf

- Always gets the latest certbot release (not distro-packaged version lagging 6+ months behind)
- Plugins install via snap as well, keeping them in sync with certbot's API
- Automatic updates (snap handles this)
- Consistent behavior across distros

### Installing DNS Plugins via Snap

```bash
# Cloudflare
sudo snap install certbot-dns-cloudflare

# Route 53
sudo snap install certbot-dns-route53

# Connect the plugin to certbot
sudo snap connect certbot:plugin certbot-dns-cloudflare
sudo snap connect certbot:plugin certbot-dns-route53

# Verify plugins are available
certbot plugins
```

### Alternative: pip (for advanced users only)

If snap is unavailable (some minimal container images, air-gapped environments):

```bash
python3 -m pip install certbot certbot-nginx certbot-dns-cloudflare
```

This is harder to keep updated and is not recommended for servers you care about.

### Alternative: acme.sh (if you refuse to use snap)

`acme.sh` is a pure-shell ACME client that requires no system dependencies and is excellent for environments where snap is unavailable. It is not covered in this guide but is a first-class alternative to Certbot.

---

## Certbot Modes

### Standalone Mode

Certbot spins up its own temporary web server on port 80 (or 443 for TLS-ALPN-01). It handles the ACME challenge entirely itself.

**Use when:**
- No web server is running (initial certificate before nginx/apache starts)
- Simple servers without complex webroot setups
- Port 80 is available and your web server can be stopped briefly

**Critical requirement:** Your web server must be stopped, or port 80 must be free, before running standalone mode.

```bash
# Stop nginx first
sudo systemctl stop nginx

# Issue the certificate
sudo certbot certonly --standalone \
  -d example.com \
  -d www.example.com \
  --email admin@example.com \
  --agree-tos \
  --no-eff-email

# Start nginx again
sudo systemctl start nginx
```

**For renewal with standalone** you must hook nginx stop/start (see [Hooks](#prepost-and-deploy-hooks)). This causes downtime on every renewal — 90 days is frequent enough to matter in production.

**Bottom line:** Standalone is great for initial issuance. For production renewals, switch to webroot or the nginx plugin.

---

### Webroot Mode

Certbot writes the ACME challenge file into a directory that your running web server serves. No downtime, no web server restart.

**Use when:**
- A web server is running and serving files from disk
- You want zero-downtime renewals
- You have control over the nginx/apache config to serve `.well-known/acme-challenge/`

**nginx configuration prerequisite:**
```nginx
server {
    listen 80;
    server_name example.com www.example.com;

    # ACME challenge — must come before any catch-all redirect
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
        default_type "text/plain";
        try_files $uri =404;
    }

    # Everything else: redirect to HTTPS
    location / {
        return 301 https://$host$request_uri;
    }
}
```

```bash
# Create the webroot directory
sudo mkdir -p /var/www/certbot
sudo chown www-data:www-data /var/www/certbot  # or nginx:nginx on RHEL

# Test nginx config
sudo nginx -t && sudo systemctl reload nginx

# Issue certificate
sudo certbot certonly --webroot \
  -w /var/www/certbot \
  -d example.com \
  -d www.example.com \
  --email admin@example.com \
  --agree-tos \
  --no-eff-email
```

**Multiple domains with different webroots:**
```bash
sudo certbot certonly --webroot \
  -w /var/www/example.com -d example.com -d www.example.com \
  -w /var/www/api.example.com -d api.example.com \
  --email admin@example.com \
  --agree-tos
```

**Verify the challenge works before running certbot:**
```bash
# Test that nginx serves the challenge path
echo "test" | sudo tee /var/www/certbot/test.txt
curl -v http://example.com/.well-known/acme-challenge/test.txt
sudo rm /var/www/certbot/test.txt
```

---

### Nginx Plugin Mode

The nginx plugin reads your existing nginx configuration, automatically adds the ACME challenge location, issues the certificate, and optionally configures SSL in your nginx config.

**Use when:**
- You want certbot to manage the nginx SSL configuration for you
- You trust automated config changes (review them — certbot adds `# managed by Certbot` comments)
- You're comfortable with certbot modifying your nginx files

```bash
# Issue and auto-configure nginx
sudo certbot --nginx \
  -d example.com \
  -d www.example.com \
  --email admin@example.com \
  --agree-tos \
  --no-eff-email

# Issue only (don't touch nginx SSL config)
sudo certbot --nginx certonly \
  -d example.com \
  -d www.example.com \
  --email admin@example.com \
  --agree-tos
```

**What the nginx plugin does to your config:**
- Adds `listen 443 ssl;` directives
- Adds `ssl_certificate` and `ssl_certificate_key` pointing to `/etc/letsencrypt/live/`
- Adds a redirect from port 80 to 443
- Does NOT configure `ssl_protocols`, `ssl_ciphers`, `ssl_session_cache`, etc. — you must add hardening yourself

**My opinion:** The nginx plugin is convenient for simple setups but creates tight coupling between certbot and your nginx config. For anything beyond a single-server personal site, I prefer `--webroot` with explicit deploy hooks. The plugin-modified configs have surprised me in upgrades more than once.

---

### Apache Plugin Mode

Identical concept to the nginx plugin, for Apache httpd:

```bash
sudo certbot --apache \
  -d example.com \
  -d www.example.com \
  --email admin@example.com \
  --agree-tos \
  --no-eff-email
```

Same caveats apply. Apache's config is more fragmented than nginx's (`.conf` files, `sites-available/sites-enabled/`, `.htaccess`) and the plugin handles it, but review what it does.

---

## DNS Plugins for Wildcard Certificates

For wildcard certificates (`*.example.com`), DNS-01 is mandatory. You need a DNS plugin that can create and delete TXT records via your DNS provider's API.

### Cloudflare DNS Plugin

**Credentials file format:**
```ini
# /etc/letsencrypt/secrets/cloudflare.ini
# Restrict permissions: chmod 600 /etc/letsencrypt/secrets/cloudflare.ini

# Option 1: API Token (PREFERRED — scoped to Zone:DNS:Edit)
dns_cloudflare_api_token = your-cloudflare-api-token-here

# Option 2: Global API Key (legacy, avoid if possible)
# dns_cloudflare_email = admin@example.com
# dns_cloudflare_api_key = your-global-api-key-here
```

**Creating a scoped Cloudflare API token:**
1. Go to Cloudflare Dashboard → My Profile → API Tokens
2. Create Token → Edit zone DNS template
3. Set Zone Resources to your specific zone (or All zones)
4. Set Permissions: Zone → DNS → Edit
5. Copy the token — you only see it once

**Issuing a wildcard certificate:**
```bash
sudo mkdir -p /etc/letsencrypt/secrets
sudo chmod 700 /etc/letsencrypt/secrets

# Create credentials file
sudo tee /etc/letsencrypt/secrets/cloudflare.ini > /dev/null <<EOF
dns_cloudflare_api_token = your-token-here
EOF
sudo chmod 600 /etc/letsencrypt/secrets/cloudflare.ini

# Staging test first
sudo certbot certonly --staging \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/secrets/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 30 \
  -d "*.example.com" \
  -d "example.com" \
  --email admin@example.com \
  --agree-tos \
  --no-eff-email

# Production (after staging passes)
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/secrets/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 30 \
  -d "*.example.com" \
  -d "example.com" \
  --email admin@example.com \
  --agree-tos \
  --no-eff-email
```

**Note on `--dns-cloudflare-propagation-seconds`:** Default is 10 seconds. Cloudflare's API propagates almost instantly, so 30 seconds is a comfortable buffer. Slower DNS providers may need 60–120 seconds.

**Note on including the apex domain:** `*.example.com` does NOT cover `example.com`. Always add both `-d "*.example.com" -d "example.com"` unless you specifically don't want the apex covered.

---

### Route 53 DNS Plugin

**Authentication methods (in order of preference):**

1. IAM Instance Role (EC2) — no credentials file needed
2. IAM credentials file (`~/.aws/credentials`)
3. Environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`)

**IAM Policy required (minimum permissions):**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:GetChange"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets"
      ],
      "Resource": "arn:aws:route53:::hostedzone/YOUR_ZONE_ID"
    }
  ]
}
```

**Using with an IAM credentials file:**
```ini
# /etc/letsencrypt/secrets/route53.ini
# chmod 600

[default]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

**Certbot command (EC2 with Instance Role — simplest):**
```bash
sudo certbot certonly \
  --dns-route53 \
  -d "*.example.com" \
  -d "example.com" \
  --email admin@example.com \
  --agree-tos \
  --no-eff-email
```

**Certbot command (with credentials file):**
```bash
# dns-route53 reads ~/.aws/credentials automatically, or set env vars:
sudo AWS_PROFILE=certbot certbot certonly \
  --dns-route53 \
  --dns-route53-propagation-seconds 30 \
  -d "*.example.com" \
  -d "example.com" \
  --email admin@example.com \
  --agree-tos \
  --no-eff-email
```

**Route 53 propagation:** Route 53 changes propagate to Route 53's own name servers very quickly (usually under 60 seconds). The `--dns-route53-propagation-seconds` default of 10 is usually fine. If you have secondary DNS or unusual TTL setups, increase to 60.

---

### Other DNS Plugins

Available via snap: `certbot-dns-digitalocean`, `certbot-dns-google`, `certbot-dns-linode`, `certbot-dns-ovh`, `certbot-dns-cloudxns`, `certbot-dns-dnsimple`, `certbot-dns-dnsmadeeasy`, `certbot-dns-gehirn`, `certbot-dns-luadns`, `certbot-dns-nsone`, `certbot-dns-rfc2136`, `certbot-dns-sakuracloud`.

For providers without an official plugin, `certbot-dns-rfc2136` works with BIND and any DNS server supporting RFC 2136 dynamic updates.

---

## Multi-Domain SANs

A single certificate can cover multiple domains (Subject Alternative Names). Up to 100 domains per certificate.

**Basic multi-domain issuance:**
```bash
sudo certbot certonly --webroot \
  -w /var/www/html \
  -d example.com \
  -d www.example.com \
  -d api.example.com \
  -d mail.example.com \
  --email admin@example.com \
  --agree-tos
```

**Mixed webroots (different document roots per domain):**
```bash
sudo certbot certonly --webroot \
  -w /var/www/main -d example.com -d www.example.com \
  -w /var/www/api  -d api.example.com \
  -w /var/www/app  -d app.example.com \
  --email admin@example.com \
  --agree-tos
```

**The certificate is stored under the first domain name:**
```
/etc/letsencrypt/live/example.com/fullchain.pem
/etc/letsencrypt/live/example.com/privkey.pem
```

**When to use a single multi-SAN cert vs. separate certs:**

Use a single cert when:
- All domains are on the same server
- You want simplified renewal management
- All domains have the same ownership/team

Use separate certs when:
- Domains go to different servers (cert distribution becomes complex)
- Different teams own different domains (key separation)
- You are approaching the 100 SAN limit
- You want independent revocation capability

**Adding a domain to an existing certificate** (re-issue with expanded SANs):
```bash
sudo certbot certonly --webroot \
  -w /var/www/html \
  -d example.com \
  -d www.example.com \
  -d api.example.com \
  -d NEW-DOMAIN.example.com \
  --expand \
  --email admin@example.com \
  --agree-tos
```

The `--expand` flag tells certbot to replace the existing cert with a new one containing additional SANs. Without it, certbot will refuse if the domain list differs from the stored config.

---

## Certificate Files Explained

Let's Encrypt issues certificates into `/etc/letsencrypt/live/<domain>/`. There are four files:

```
/etc/letsencrypt/live/example.com/
├── cert.pem        -> ../../archive/example.com/cert1.pem
├── chain.pem       -> ../../archive/example.com/chain1.pem
├── fullchain.pem   -> ../../archive/example.com/fullchain1.pem
└── privkey.pem     -> ../../archive/example.com/privkey1.pem
```

These are **symlinks** to the actual files in `/etc/letsencrypt/archive/example.com/`. The symlinks always point to the most recently issued certificate. When a certificate is renewed, certbot updates the symlinks and increments the archive number (`cert2.pem`, `cert3.pem`, etc.).

### cert.pem — Your Leaf Certificate Only

Contains only the end-entity (leaf) certificate for your domain. It is a PEM-encoded X.509 certificate — one `-----BEGIN CERTIFICATE-----` block.

**When to use:** Almost never directly. Most software needs the full chain. Some software (HAProxy) wants them in specific separate files. Some monitoring scripts that check the domain cert only.

**What it contains:**
```
Subject: CN=example.com
Issuer: CN=R10, O=Let's Encrypt, C=US
SubjectAltName: DNS:example.com, DNS:www.example.com
Validity: Not Before / Not After (90-day window)
```

### chain.pem — Intermediate Certificate(s) Only

Contains the intermediate CA certificate(s) — everything between your leaf cert and Let's Encrypt's root, excluding both endpoints.

**When to use:** Rarely directly. Some software explicitly wants the chain separate from the leaf. OCSP stapling configurations sometimes need it. Sending intermediate certs to clients is required for proper SSL trust chain building.

**What it contains:** Currently Let's Encrypt's intermediate (e.g., R10 or E5) signed by ISRG Root X1.

### fullchain.pem — Leaf + Chain Combined (Use This for Most Things)

`cert.pem` + `chain.pem` concatenated. This is what your web server should present to clients.

**Use this for:**
- nginx `ssl_certificate`
- Apache `SSLCertificateFile`
- HAProxy `crt` (combined with privkey.pem — see below)
- Any application that wants a "certificate file" that includes intermediates

**Why you need the full chain:** Browsers trust root CAs, not Let's Encrypt's intermediates directly (even though ISRG Root X1 is now in most trust stores, best practice is always send the chain). If you only send `cert.pem`, clients that don't have Let's Encrypt's intermediate cached may fail with "untrusted certificate" — especially non-browser clients, embedded systems, and older mobile OSes.

### privkey.pem — Private Key

The private key corresponding to the certificate. PEM-encoded. Never leave this server. Never log it. Never copy it to an untrusted location.

**Permissions:** Certbot sets this `0600` owned by root. Do not change the ownership to your web server user. Instead, run your web server as root for the certificate load phase, or use capabilities (`CAP_NET_BIND_SERVICE`), or copy via deploy hooks to a restricted location.

**Use this for:**
- nginx `ssl_certificate_key`
- Apache `SSLCertificateKeyFile`
- Any application's "private key" field

### nginx Configuration Example

```nginx
server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # Recommended SSL hardening (add this, certbot nginx plugin does NOT)
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;
    ssl_session_timeout 1d;
    ssl_session_cache shared:MozSSL:10m;
    ssl_session_tickets off;

    # HSTS — add only when you are sure your HTTPS works
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;
    resolver 1.1.1.1 8.8.8.8 valid=300s;
    resolver_timeout 5s;
}
```

### HAProxy Configuration Example

HAProxy requires the certificate and key in a single file:

```bash
# Create combined file (run in deploy hook)
cat /etc/letsencrypt/live/example.com/fullchain.pem \
    /etc/letsencrypt/live/example.com/privkey.pem \
    > /etc/haproxy/certs/example.com.pem
chmod 600 /etc/haproxy/certs/example.com.pem
```

```haproxy
frontend https_front
    bind *:443 ssl crt /etc/haproxy/certs/example.com.pem
    default_backend app_servers
```

---

## Renewal

### The 90-Day Reality

Certbot's auto-renewal target is **30 days before expiration** — it will attempt renewal when the certificate has 30 days or fewer remaining. With a 90-day cert, this means renewal happens at roughly the 60-day mark. Certbot checks this on every run and skips if the cert is not due.

### Checking Certificate Status

```bash
# Show all managed certs and their expiry
certbot certificates

# Output:
# Found the following certs:
#   Certificate Name: example.com
#     Serial Number: ...
#     Key Type: RSA
#     Domains: example.com www.example.com
#     Expiry Date: 2025-09-15 12:00:00+00:00 (VALID: 75 days)
#     Certificate Path: /etc/letsencrypt/live/example.com/fullchain.pem
#     Private Key Path: /etc/letsencrypt/live/example.com/privkey.pem
```

### systemd Timer (Modern — Preferred)

On systems with systemd, certbot snap installs a timer automatically. Check it:

```bash
# Check the timer unit
systemctl status snap.certbot.renew.timer

# Check the service
systemctl status snap.certbot.renew.service

# Manually trigger renewal (dry run)
certbot renew --dry-run

# Manually trigger actual renewal
certbot renew
```

The snap timer runs twice daily (at random offsets to distribute load on Let's Encrypt's servers — this is intentional and good).

**If using OS-packaged certbot (not snap):**
```bash
# Timer file: /lib/systemd/system/certbot.timer
systemctl status certbot.timer
systemctl list-timers certbot.timer
```

### /etc/cron.d/certbot (Legacy — Only If No systemd)

OS-packaged certbot on older Debian/Ubuntu systems uses cron:

```
# /etc/cron.d/certbot
0 */12 * * * root test -x /usr/bin/certbot -a \! -d /run/systemd/system && perl -e 'sleep int(rand(43200))' && certbot renew -q
```

Key points:
- Runs twice daily (every 12 hours)
- Only runs if systemd is NOT present (`\! -d /run/systemd/system`)
- Adds a random sleep up to 12 hours to avoid hammering Let's Encrypt
- The `-q` flag suppresses output — errors still go to cron's mail

**Do not have both the systemd timer and cron active simultaneously.** Pick one. On snap installs, the timer is always used.

### Renewal Configuration

Each certificate has a renewal configuration at `/etc/letsencrypt/renewal/<domain>.conf`:

```ini
# /etc/letsencrypt/renewal/example.com.conf
[renewalparams]
authenticator = webroot
webroot_path = /var/www/certbot,
account = abc123...
server = https://acme-v02.api.letsencrypt.org/directory

[[webroot_map]]
example.com = /var/www/certbot
www.example.com = /var/www/certbot
```

This is where certbot stores the parameters to auto-renew. Do not hand-edit this unless you know what you're doing. If you change the issuance method (e.g., from standalone to webroot), update this file or re-issue with the new method.

---

## Pre/Post and Deploy Hooks

Hooks allow you to run commands before validation, after validation, and after successful renewal. This is how you handle service reloads, combined file generation, and notifications.

### Hook Locations

```
/etc/letsencrypt/renewal-hooks/
├── pre/        # Run before renewal attempt (stop services for standalone)
├── post/       # Run after renewal attempt (start services, whether cert renewed or not)
└── deploy/     # Run only when a certificate is successfully renewed
```

All files in these directories are executed if they are executable. Naming convention: `00-`, `10-`, `20-` prefixes control execution order.

### Deploy Hook — nginx Reload (Most Common)

```bash
# /etc/letsencrypt/renewal-hooks/deploy/01-nginx-reload.sh
#!/bin/bash
set -e
nginx -t && systemctl reload nginx
```

```bash
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/01-nginx-reload.sh
```

This is the hook you almost always want. It reloads nginx config only when a cert is actually renewed (not on every certbot run).

### Pre/Post Hooks — Standalone Mode Service Management

```bash
# /etc/letsencrypt/renewal-hooks/pre/01-stop-nginx.sh
#!/bin/bash
systemctl stop nginx

# /etc/letsencrypt/renewal-hooks/post/01-start-nginx.sh
#!/bin/bash
systemctl start nginx
```

```bash
sudo chmod +x /etc/letsencrypt/renewal-hooks/pre/01-stop-nginx.sh
sudo chmod +x /etc/letsencrypt/renewal-hooks/post/01-start-nginx.sh
```

**Warning:** pre and post hooks run for EVERY renewal attempt, even if the cert doesn't actually need renewal (but certbot skips the actual ACME exchange). For standalone mode, this means nginx stops and starts on every certbot timer run. That is up to 4 times per day (certbot checks twice daily, but with systemd timer randomization, depends on config). Consider switching to webroot to avoid this.

### Deploy Hook — HAProxy Combined File

```bash
# /etc/letsencrypt/renewal-hooks/deploy/10-haproxy.sh
#!/bin/bash
set -e

DOMAIN="example.com"
LIVE="/etc/letsencrypt/live/${DOMAIN}"
HAPROXY_CERT="/etc/haproxy/certs/${DOMAIN}.pem"

cat "${LIVE}/fullchain.pem" "${LIVE}/privkey.pem" > "${HAPROXY_CERT}"
chmod 600 "${HAPROXY_CERT}"

systemctl reload haproxy
```

### Deploy Hook — Notification

```bash
# /etc/letsencrypt/renewal-hooks/deploy/99-notify.sh
#!/bin/bash
# Send a Slack notification on successful renewal
DOMAIN="${RENEWED_DOMAINS}"
EXPIRY="${RENEWED_LINEAGE}"

curl -s -X POST https://hooks.slack.com/services/YOUR/WEBHOOK/URL \
  -H 'Content-type: application/json' \
  -d "{\"text\": \"Certificate renewed for: ${DOMAIN}\"}"
```

**Environment variables available in deploy hooks:**

| Variable | Contents |
|---|---|
| `RENEWED_LINEAGE` | Path to the live cert directory (e.g., `/etc/letsencrypt/live/example.com`) |
| `RENEWED_DOMAINS` | Space-separated list of renewed domains |

### Testing Hooks

```bash
# Dry run — runs hooks but doesn't issue real certs
certbot renew --dry-run

# Force renewal (even if cert isn't due) — actually renews
certbot renew --force-renewal

# Test specific certificate
certbot renew --cert-name example.com --dry-run
```

---

## Revocation

Revoke a certificate when:
- The private key was compromised
- The server is decommissioned
- You suspect unauthorized access

**Revoke a certificate you still have:**
```bash
sudo certbot revoke \
  --cert-path /etc/letsencrypt/live/example.com/cert.pem \
  --reason keycompromise
```

**Valid `--reason` values:**
- `unspecified` (default)
- `keycompromise` — use this if the private key was exposed
- `affiliationchanged`
- `superseded`
- `cessationofoperation`

**Revoke and delete:**
```bash
sudo certbot revoke \
  --cert-path /etc/letsencrypt/live/example.com/cert.pem \
  --reason keycompromise \
  --delete-after-revoke
```

**Revoke by certificate name:**
```bash
sudo certbot revoke --cert-name example.com
```

**After revoking, delete the cert files:**
```bash
sudo certbot delete --cert-name example.com
```

**If you no longer have the private key** (server is gone, disk is dead), you can still revoke using your account key. Contact Let's Encrypt support or use a different ACME client that supports account-key-based revocation.

**OCSP and CRL:** Let's Encrypt uses OCSP. CRL is published at `http://c.letsencrypt.org/`. Revocation via OCSP takes effect within minutes. Browser checking is another story — most browsers use OCSP stapling or CRL sets rather than real-time OCSP checks.

---

## Server Migration

Migrating certificates between servers is straightforward. The key insight: certificates and their keys live entirely in `/etc/letsencrypt/`. Copy that directory and you have everything.

### Method 1: rsync Copy (Same-Owner Migration)

```bash
# On the OLD server — create a tarball
sudo tar czf /tmp/letsencrypt-backup.tar.gz \
  --preserve-permissions \
  /etc/letsencrypt/

# Transfer to new server
rsync -avz /tmp/letsencrypt-backup.tar.gz newserver:/tmp/

# On the NEW server — extract
sudo tar xzf /tmp/letsencrypt-backup.tar.gz -C /

# Verify permissions (privkey.pem must be 0600 root:root)
ls -la /etc/letsencrypt/archive/example.com/
ls -la /etc/letsencrypt/live/example.com/
```

**Verify the symlinks are intact:**
```bash
ls -la /etc/letsencrypt/live/example.com/
# Should show symlinks pointing to ../../archive/...
```

**Install certbot on the new server** and run `certbot certificates` to confirm it reads the migrated certs:
```bash
certbot certificates
```

**Update your renewal config if needed** (`/etc/letsencrypt/renewal/example.com.conf`) — if the webroot path or authenticator differs on the new server, update it before the next renewal.

### Method 2: DNS-01 Re-Issuance (Clean Migration)

If you can't copy files (e.g., old server is gone, different OS), re-issue using DNS-01 on the new server. This works without the old server being accessible:

```bash
# On new server — issue fresh certificate via DNS-01
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/secrets/cloudflare.ini \
  -d example.com \
  -d www.example.com \
  --email admin@example.com \
  --agree-tos
```

This does not require the old server's IP to route traffic yet. You can issue the cert before cutting over DNS.

### Migration Checklist

```
[ ] Install certbot on new server
[ ] Copy /etc/letsencrypt/ with preserved permissions (or re-issue)
[ ] Verify symlinks in /etc/letsencrypt/live/
[ ] Update renewal config authenticator/webroot if changed
[ ] Configure nginx/apache on new server with cert paths
[ ] Test: openssl s_client -connect newserver-ip:443 -servername example.com
[ ] Cut over DNS
[ ] Verify: curl -I https://example.com
[ ] Set up renewal hooks on new server
[ ] Test renewal: certbot renew --dry-run
[ ] Decommission old server
[ ] Do NOT revoke the old cert unless key was compromised — it expires in 90 days anyway
```

---

## Troubleshooting

### Renewal Failures — Diagnosing

```bash
# Run renewal in verbose/debug mode
certbot renew --dry-run -v

# Check the certbot log
tail -100 /var/log/letsencrypt/letsencrypt.log

# Force a specific cert to renew (ignore timing)
certbot renew --cert-name example.com --force-renewal --dry-run
```

---

### Port 80 Blocked (HTTP-01 Failures)

**Symptom:**
```
Failed to connect to http://example.com/.well-known/acme-challenge/<token>
Connection refused / Connection timed out
```

**Check:**
```bash
# Verify port 80 is listening locally
ss -tlnp | grep ':80'

# Verify from an external perspective (use a VPS or online tool)
nc -zv example.com 80

# Check firewall rules
sudo iptables -L INPUT -n -v | grep -E '80|http'
sudo ufw status verbose  # if using ufw
sudo firewall-cmd --list-all  # if using firewalld
```

**Common causes:**
- Cloud security groups (AWS, GCP, Azure) blocking port 80 inbound
- `ufw` or `iptables` rules not allowing port 80
- `fail2ban` rate-limiting Let's Encrypt's validation IPs
- Port 80 redirecting to HTTPS before serving `.well-known/acme-challenge/`

**Fix for ufw:**
```bash
sudo ufw allow 80/tcp
```

**Fix for firewalld:**
```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

**Fix for AWS Security Group:** Manually add an inbound rule for port 80 from `0.0.0.0/0`. You cannot whitelist Let's Encrypt's validation IPs — they change and are geographically distributed.

---

### DNS Propagation Issues (DNS-01 Failures)

**Symptom:**
```
DNS problem: NXDOMAIN looking up TXT for _acme-challenge.example.com
Timeout during connect (likely firewall problem)
```

**Debug steps:**
```bash
# Check what the TXT record shows right now
dig TXT _acme-challenge.example.com +short
dig TXT _acme-challenge.example.com @8.8.8.8 +short   # Google's resolver
dig TXT _acme-challenge.example.com @1.1.1.1 +short   # Cloudflare's resolver

# Check current TTL on the record
dig TXT _acme-challenge.example.com +ttlid
```

**Fix:** Increase propagation seconds:
```bash
certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/secrets/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 120 \
  -d "*.example.com" -d "example.com"
```

**Fix:** Lower the TTL on your DNS zone before next renewal attempt:
```bash
# In your DNS provider, set TTL of _acme-challenge to 60 seconds
# Wait for old TTL to expire before triggering renewal
```

**Split-horizon DNS:** If your internal DNS resolves `_acme-challenge.example.com` differently from public DNS, Let's Encrypt will see the public response — which may be missing the record. Ensure your DNS plugin writes to public DNS.

---

### Wrong Webroot / 404 on Challenge File

**Symptom:**
```
404 Not Found for http://example.com/.well-known/acme-challenge/<token>
```

**Debug:**
```bash
# Manually create a test file
echo "hello" | sudo tee /var/www/certbot/hello.txt

# Test from the server itself
curl -v http://localhost/.well-known/acme-challenge/hello.txt

# Test externally
curl -v http://example.com/.well-known/acme-challenge/hello.txt

# Clean up
sudo rm /var/www/certbot/hello.txt
```

**Common causes:**
- Webroot path in certbot doesn't match nginx `root` or `alias` directive
- nginx config for `.well-known/acme-challenge/` is inside the HTTPS server block, not HTTP
- `try_files` or `index` directive consuming the request before the file is served
- nginx catches all requests and proxies to an app, eating the challenge request

**nginx fix — ensure challenge is served before any proxy/redirect:**
```nginx
server {
    listen 80;
    server_name example.com;

    # THIS MUST COME FIRST — before any return 301 or proxy_pass
    location ^~ /.well-known/acme-challenge/ {
        root /var/www/certbot;
        default_type "text/plain";
    }

    location / {
        return 301 https://$host$request_uri;
    }
}
```

The `^~` modifier gives this location block priority over regex locations.

---

### Certificate Already Exists / Certbot Refuses to Re-Issue

**Symptom:**
```
Certificate not yet due for renewal. Use --force-renewal to force issuance.
```

```bash
# Force renewal regardless of expiry
certbot renew --cert-name example.com --force-renewal

# Or for certonly
certbot certonly --force-renewal --webroot -w /var/www/certbot \
  -d example.com -d www.example.com
```

---

### Staging Certificate Stuck in Production Config

**Symptom:** Browser shows "NET::ERR_CERT_AUTHORITY_INVALID" with a cert issued by "Fake LE Intermediate"

**Fix:** Delete the staging cert and re-issue from production:
```bash
# Remove the staging certificate
sudo certbot delete --cert-name example.com

# Re-issue from production
sudo certbot certonly --webroot \
  -w /var/www/certbot \
  -d example.com -d www.example.com \
  --email admin@example.com \
  --agree-tos \
  --no-eff-email
```

---

### Rate Limit Hit

**Symptom:**
```
too many certificates already issued for exact set of domains
Error: urn:ietf:params:acme:error:rateLimited
```

**Action:**
1. Check crt.sh to see how many certs were issued this week:
   ```bash
   curl -s "https://crt.sh/?q=example.com&output=json" | \
     jq -r '.[].not_before' | sort -r | head -20
   ```
2. Wait for the rolling 7-day window to clear
3. In the meantime, use the staging environment to continue testing
4. After rate limit clears, use `--force-renewal` only if needed

---

### Certbot Renew Succeeds But nginx Still Serves Old Cert

**Symptom:** cert renewed successfully but `openssl s_client` shows the old expiry date

**Cause:** nginx reads the certificate once at startup/reload and caches it. Renewal without reloading nginx means the old cert stays in memory.

**Fix:** Add a deploy hook (see [Deploy Hooks](#prepost-and-deploy-hooks)):
```bash
# Immediate fix
sudo systemctl reload nginx

# Permanent fix — deploy hook
echo '#!/bin/bash
nginx -t && systemctl reload nginx' | \
  sudo tee /etc/letsencrypt/renewal-hooks/deploy/01-nginx-reload.sh
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/01-nginx-reload.sh
```

**Verify the hook runs:**
```bash
certbot renew --force-renewal --cert-name example.com
# Should see "Running deploy-hook command: ..."
```

---

### Checking Certificate Details

```bash
# Check the cert directly
openssl x509 -in /etc/letsencrypt/live/example.com/fullchain.pem \
  -text -noout | grep -E 'Subject:|Issuer:|DNS:|Not Before|Not After'

# Check what a remote server is presenting
openssl s_client -connect example.com:443 -servername example.com \
  -showcerts < /dev/null 2>/dev/null | \
  openssl x509 -text -noout | grep -E 'Subject:|Issuer:|DNS:|Not Before|Not After'

# Check expiry in days remaining
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | \
  openssl x509 -noout -dates

# Quick expiry check
certbot certificates --cert-name example.com
```

---

### Failed Validation Lockout (5 Failures Per Hour)

**Symptom:**
```
Error: urn:ietf:params:acme:error:rateLimited
There were too many requests of a given type :: Error creating new order :: too many failed authorizations recently
```

**Cause:** Five or more failed validation attempts in the past hour against the same hostname.

**Action:** Wait one hour. Do not keep retrying — each attempt burns toward the limit while you're already locked. Fix the underlying problem (port 80, DNS, webroot) first, then retry.

---

### Certbot Permission Errors

```bash
# Check who owns /etc/letsencrypt/
ls -la /etc/letsencrypt/

# If you've accidentally changed permissions:
sudo chown -R root:root /etc/letsencrypt/
sudo chmod 755 /etc/letsencrypt/
sudo chmod 755 /etc/letsencrypt/live/
sudo chmod 755 /etc/letsencrypt/archive/
sudo chmod 700 /etc/letsencrypt/archive/example.com/
sudo chmod 600 /etc/letsencrypt/archive/example.com/privkey*.pem
```

---

## Hardening and Security Notes

### Key Type and Size

Certbot defaults to RSA 2048-bit keys. For new deployments, use ECDSA P-256:

```bash
certbot certonly --webroot \
  -w /var/www/certbot \
  -d example.com \
  --key-type ecdsa \
  --elliptic-curve secp256r1 \
  --email admin@example.com \
  --agree-tos
```

ECDSA P-256 is faster, smaller, and equally secure for TLS 1.2+ purposes. The only reason to use RSA is compatibility with very old clients (pre-2011 mobile browsers, embedded systems).

### privkey.pem Access Control

The private key must be readable only by root. If your web server runs as a non-root user (it should), nginx reads the key during the `nginx -t` / reload that runs as root before dropping privileges. Never:
- `chown www-data /etc/letsencrypt/archive/*/privkey*.pem`
- `chmod 644 /etc/letsencrypt/archive/*/privkey*.pem`
- Copy privkey.pem to a world-readable location

If an application (not nginx) needs the key, copy it to a controlled location in a deploy hook and restrict access:

```bash
# In deploy hook
install -m 640 -o root -g myapp \
  /etc/letsencrypt/live/example.com/privkey.pem \
  /etc/myapp/tls/privkey.pem
```

### HSTS and Preloading

Once HTTPS is working reliably, enable HSTS:

```nginx
# Start with a short max-age (5 minutes) to test
add_header Strict-Transport-Security "max-age=300" always;

# Once confirmed working, increase to 1 year with includeSubDomains
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

# Only add preload if you intend to submit to the HSTS preload list
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
```

### OCSP Stapling

Enable OCSP stapling to improve performance (client doesn't need to contact Let's Encrypt's OCSP responder):

```nginx
ssl_stapling on;
ssl_stapling_verify on;
ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;
resolver 1.1.1.1 8.8.8.8 valid=300s;
resolver_timeout 5s;
```

### Certificate Transparency Monitoring

Set up CT log monitoring to detect unauthorized certificate issuance:

```bash
# Use crt.sh RSS or API
curl -s "https://crt.sh/?q=%.example.com&output=json" | \
  jq -r '.[] | "\(.not_before) \(.common_name) \(.name_value)"' | \
  sort -r | head -20
```

Consider services like `certspotter` (open-source) or commercial CT monitoring for production domains.

---

## Quick Reference Cheatsheet

```bash
# ── INSTALLATION ──────────────────────────────────────────────────────────────
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/local/bin/certbot
sudo snap install certbot-dns-cloudflare
sudo snap connect certbot:plugin certbot-dns-cloudflare

# ── STAGING FIRST ─────────────────────────────────────────────────────────────
certbot certonly --staging --webroot -w /var/www/certbot \
  -d example.com -d www.example.com \
  --email admin@example.com --agree-tos --no-eff-email

# ── HTTP-01 (webroot) ─────────────────────────────────────────────────────────
certbot certonly --webroot -w /var/www/certbot \
  -d example.com -d www.example.com \
  --email admin@example.com --agree-tos --no-eff-email

# ── HTTP-01 (standalone) ──────────────────────────────────────────────────────
systemctl stop nginx
certbot certonly --standalone \
  -d example.com -d www.example.com \
  --email admin@example.com --agree-tos --no-eff-email
systemctl start nginx

# ── HTTP-01 (nginx plugin) ────────────────────────────────────────────────────
certbot --nginx -d example.com -d www.example.com \
  --email admin@example.com --agree-tos --no-eff-email

# ── DNS-01 WILDCARD (Cloudflare) ──────────────────────────────────────────────
certbot certonly --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/secrets/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 30 \
  -d "*.example.com" -d "example.com" \
  --email admin@example.com --agree-tos --no-eff-email

# ── DNS-01 WILDCARD (Route 53) ────────────────────────────────────────────────
certbot certonly --dns-route53 \
  -d "*.example.com" -d "example.com" \
  --email admin@example.com --agree-tos --no-eff-email

# ── RENEWAL ───────────────────────────────────────────────────────────────────
certbot renew --dry-run                         # Test renewal
certbot renew                                   # Actual renewal
certbot renew --force-renewal --cert-name example.com  # Force specific cert

# ── CERT INFO ─────────────────────────────────────────────────────────────────
certbot certificates
openssl x509 -in /etc/letsencrypt/live/example.com/fullchain.pem -text -noout

# ── REVOCATION ────────────────────────────────────────────────────────────────
certbot revoke --cert-path /etc/letsencrypt/live/example.com/cert.pem \
  --reason keycompromise --delete-after-revoke

# ── DELETE CERT ───────────────────────────────────────────────────────────────
certbot delete --cert-name example.com

# ── EXPAND CERT (add SANs) ────────────────────────────────────────────────────
certbot certonly --webroot -w /var/www/certbot \
  -d example.com -d www.example.com -d api.example.com \
  --expand --email admin@example.com --agree-tos

# ── RATE LIMIT CHECK ─────────────────────────────────────────────────────────
curl -s "https://crt.sh/?q=example.com&output=json" | \
  jq -r '.[].not_before' | sort -r | head -10

# ── CHECK REMOTE CERT ─────────────────────────────────────────────────────────
openssl s_client -connect example.com:443 -servername example.com \
  < /dev/null 2>/dev/null | openssl x509 -noout -dates

# ── NGINX DEPLOY HOOK ─────────────────────────────────────────────────────────
echo '#!/bin/bash
nginx -t && systemctl reload nginx' \
  | sudo tee /etc/letsencrypt/renewal-hooks/deploy/01-nginx-reload.sh
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/01-nginx-reload.sh
```

---

## Decision Tree

```
Need a Let's Encrypt certificate?
│
├─ Is it a wildcard (*.example.com)?
│   └─ YES → DNS-01 (required). Pick dns-cloudflare or dns-route53 plugin.
│
├─ Is the server publicly reachable on port 80?
│   ├─ YES → Use HTTP-01
│   │   ├─ Web server is running? → webroot mode (no downtime)
│   │   ├─ No web server running? → standalone mode
│   │   └─ Want certbot to manage nginx config? → nginx plugin
│   │
│   └─ NO → Why not?
│       ├─ Internal/private network → DNS-01 (only option)
│       ├─ Firewall blocks port 80 → DNS-01 or open port 80
│       └─ Port 443 only available → TLS-ALPN-01 (last resort)
│
└─ Always test with --staging first. Always.
```

---

*Last updated: April 2026. Let's Encrypt intermediates, rate limits, and certbot plugin names change periodically — verify against https://letsencrypt.org/docs/ and https://certbot.eff.org/ for the latest.*
