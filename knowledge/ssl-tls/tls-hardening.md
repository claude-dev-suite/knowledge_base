# TLS Hardening: Production-Grade SSL/TLS Configuration

**Audience:** Senior sysadmins and security engineers deploying nginx/Apache in production.  
**Stance:** Opinionated. TLS 1.2 + 1.3 minimum (TLS 1.3 preferred where client base allows). Mozilla Intermediate profile is the recommended default for most production deployments. Modern profile if your client base is current. Old profile is a legacy crutch — do not use it for new deployments.

---

## Table of Contents

1. [Why Old Protocol Versions Are Dead](#1-why-old-protocol-versions-are-dead)
2. [Protocol Selection: TLS 1.3 Only vs TLS 1.2+1.3](#2-protocol-selection-tls-13-only-vs-tls-121-3)
3. [Cipher Suite Choices](#3-cipher-suite-choices)
4. [Mozilla SSL Config Profiles](#4-mozilla-ssl-config-profiles)
5. [DHParam: When You Need It and When You Don't](#5-dhparam-when-you-need-it-and-when-you-dont)
6. [OCSP Stapling](#6-ocsp-stapling)
7. [HSTS Rollout Strategy](#7-hsts-rollout-strategy)
8. [HSTS Preload](#8-hsts-preload)
9. [Session Tickets vs Session Cache](#9-session-tickets-vs-session-cache)
10. [TLS 1.3 Early Data (0-RTT) Risks](#10-tls-13-early-data-0-rtt-risks)
11. [Testing Your Configuration](#11-testing-your-configuration)
12. [A+ SSL Labs Checklist](#12-a-ssl-labs-checklist)
13. [Complete Reference Configs](#13-complete-reference-configs)

---

## 1. Why Old Protocol Versions Are Dead

### SSLv2 — Broken Since 1995

SSLv2 has no business existing anywhere near production. DROWN (CVE-2016-0800) demonstrated that a server supporting SSLv2 — even if the client never negotiates it — can be used to decrypt TLS 1.x sessions via a cross-protocol attack. If your OpenSSL was built with SSLv2 support, disable it at compile time or via configuration. Modern OpenSSL versions don't ship it at all.

### SSLv3 — POODLE Killed It (2014)

POODLE (Padding Oracle On Downgraded Legacy Encryption, CVE-2014-3566) exploited SSLv3's CBC padding, allowing an active MITM to recover plaintext from SSLv3 sessions. Even with TLS_FALLBACK_SCSV, you cannot safely support SSLv3. It's been browser-dropped for over a decade. Zero legitimate modern clients require it. Disable unconditionally.

### TLS 1.0 — BEAST, POODLE-TLS, and PCI-DSS Non-Compliance

TLS 1.0 has been deprecated by PCI-DSS since June 2018 (required for all PCI-scoped systems). BEAST (CVE-2011-3389) exploited predictable CBC IVs in TLS 1.0. While mitigations exist client-side, the attack surface is real. POODLE-TLS (2014) showed many servers were vulnerable to padding oracle attacks on TLS 1.0 as well. Chrome and Firefox both dropped TLS 1.0 in 2020. RFC 8996 (March 2021) formally deprecated TLS 1.0 and TLS 1.1.

**Supporting TLS 1.0 in 2026 will fail PCI-DSS assessments, HIPAA technical safeguard reviews, and any competent security audit.**

### TLS 1.1 — Deprecated Alongside TLS 1.0

TLS 1.1 addressed BEAST by adding explicit IVs but retained MD5/SHA-1 in the PRF and continued using CBC. No forward secrecy. Deprecated by RFC 8996 alongside TLS 1.0. Drop it.

### Who Still Uses TLS 1.0/1.1?

The honest answer: ancient Java 6/7 clients (EOL), IE 8–10 on Windows XP (EOL), very old Android (< 4.4, effectively extinct), and some embedded/IoT devices with unmaintained firmware. If you genuinely need to support these, you have a legacy exception problem to solve at the network or application level, not by weakening your main TLS configuration. Use a separate endpoint or proxy layer.

---

## 2. Protocol Selection: TLS 1.3 Only vs TLS 1.2+1.3

### TLS 1.3 — What Changed

TLS 1.3 (RFC 8446, August 2018) is not an incremental improvement — it's a redesign. Key changes:

- **Handshake reduced from 2 RTTs to 1 RTT** (plus optional 0-RTT resumption).
- **Forward secrecy is mandatory.** No static RSA key exchange. No DSS. All key exchanges use ephemeral Diffie-Hellman.
- **Cipher suite simplification.** Only five cipher suites are defined, all AEAD. The cipher and hash are decoupled from the key exchange and authentication. You cannot configure "bad" TLS 1.3 ciphers because there aren't any.
- **Removed legacy junk:** RC4, 3DES, export ciphers, DH < 2048, anonymous cipher suites, CBC with MAC-then-Encrypt, RSA key transport, compression — all gone from the spec.
- **Encrypted handshake.** Certificate, CertificateVerify, and Finished are encrypted. An observer sees the SNI and ClientHello but nothing else about the certificate or negotiation.

### TLS 1.3 Only: When to Do It

Use TLS 1.3 only if:

- Your client base is fully modern: Chrome 70+, Firefox 63+, Safari 12.1+, Edge 79+, OpenSSL 1.1.1+ clients, curl 7.61.0+ (built with OpenSSL 1.1.1+).
- You are running an internal API, microservice mesh, or B2B endpoint where you control both sides.
- Your load balancer or reverse proxy handles TLS termination and the backend communication is service-to-service.

In nginx: `ssl_protocols TLSv1.3;`

In Apache: `SSLProtocol -all +TLSv1.3`

### TLS 1.2 + 1.3: The Recommended Default

For public-facing production services, run both TLS 1.2 and TLS 1.3. TLS 1.3 will be negotiated by all modern clients (browsers, mobile apps, modern HTTP clients). TLS 1.2 catches the tail of legitimate modern-ish clients that haven't yet gained TLS 1.3 support: some older Android WebViews, older Java 11 builds, legacy curl, some enterprise proxies, older Golang HTTP clients.

**Recommendation: TLS 1.2 + TLS 1.3. Do not add TLS 1.1 or lower under any circumstances.**

In nginx: `ssl_protocols TLSv1.2 TLSv1.3;`

In Apache: `SSLProtocol -all +TLSv1.2 +TLSv1.3`

---

## 3. Cipher Suite Choices

### The Principles

**Forward Secrecy (PFS):** If an attacker records your encrypted traffic today and later obtains your private key, they should not be able to decrypt that traffic. Forward secrecy ensures that the session key material is ephemeral — discarded after the session ends — so a compromised long-term key cannot unlock past sessions. This requires ECDHE or DHE key exchange. Static RSA key exchange (removed in TLS 1.3, still present in TLS 1.2) provides zero forward secrecy.

**AEAD (Authenticated Encryption with Associated Data):** AEAD modes (GCM, CCM, ChaCha20-Poly1305) provide both confidentiality and integrity in a single primitive, eliminating the MAC-then-Encrypt vulnerability that plagued CBC modes (BEAST, Lucky13, POODLE). Prefer AEAD always.

**Avoid:**

| Algorithm | Why |
|-----------|-----|
| RC4 | Statistically biased, multiple practical attacks (Bar-Mitzvah, NOMORE). Prohibited by RFC 7465. |
| 3DES (Triple-DES) | 64-bit block size makes it vulnerable to SWEET32 (birthday attacks on long sessions). ~112-bit effective security. RFC 7525 recommends against. |
| CBC ciphers (in TLS 1.2) | MAC-then-Encrypt model is fragile. BEAST, Lucky13, POODLE variants. Avoid unless you have no choice for legacy clients. |
| RSA key exchange | No forward secrecy. If private key is compromised, all recorded sessions are decryptable. |
| Export ciphers | Deliberately weakened 40/56-bit keys. FREAK attack. Prohibited. |
| Anonymous DH/ECDH | No authentication. Trivially MITM-able. |
| MD5 MACs | Collision-prone. |
| SHA-1 MACs | Weakened; avoid where possible (SHA-256+ preferred). |
| DH < 2048 bits | Logjam attack. NSA-suspected precomputation targets. |
| ECDH with weak curves | Avoid secp192r1, secp224r1. Use P-256 (secp256r1), P-384, P-521, X25519. |

### TLS 1.3 Cipher Suites

TLS 1.3 only supports five cipher suites, all of which are AEAD and use HKDF for key derivation. You don't configure these in the same way — the server and client negotiate from these automatically:

```
TLS_AES_128_GCM_SHA256         (most common, strong)
TLS_AES_256_GCM_SHA384         (stronger, slightly slower)
TLS_CHACHA20_POLY1305_SHA256   (preferred on devices without AES hardware acceleration)
TLS_AES_128_CCM_SHA256         (rare, niche use)
TLS_AES_128_CCM_8_SHA256       (truncated MAC, avoid)
```

In practice, modern clients will negotiate `TLS_AES_128_GCM_SHA256` or `TLS_CHACHA20_POLY1305_SHA256`. All of these provide forward secrecy (key exchange is handled separately via ephemeral ECDH with X25519 or P-256). You cannot make TLS 1.3 use insecure ciphers.

To restrict TLS 1.3 ciphers in nginx (generally not necessary, but possible):
```nginx
ssl_conf_command Ciphersuites TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256;
```

### TLS 1.2 Cipher Suites

This is where the configuration matters. The Mozilla Intermediate profile cipher string for TLS 1.2 (as of 2024-2025) is:

```
ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305
```

Breaking this down:
- **ECDHE-ECDSA-*** : Elliptic Curve Diffie-Hellman Ephemeral with ECDSA certificate. Fastest, smallest handshake. Use if you have an ECDSA (EC) certificate.
- **ECDHE-RSA-*** : ECDHE key exchange with RSA certificate. Standard for most deployments.
- **DHE-RSA-*** : Finite-field DHE with RSA certificate. Slower than ECDHE but provides forward secrecy for clients that don't support ECDHE. Requires a dhparam file.
- **-AES128-GCM-SHA256 / -AES256-GCM-SHA384** : AES-GCM is AEAD. Hardware-accelerated on virtually all modern CPUs via AES-NI.
- **-CHACHA20-POLY1305** : AEAD. Faster than AES-GCM on hardware without AES-NI (mobile, embedded). OpenSSL selects this automatically via BoringSSL's cipher preference ordering or explicit configuration.

**What is NOT in this string and why:**
- No `RSA` key exchange (no forward secrecy)
- No `3DES` (SWEET32)
- No `RC4` (multiple attacks)
- No `CBC` ciphers (BEAST/Lucky13/POODLE exposure)
- No `DES`, `NULL`, `EXPORT`, `aNULL`, `eNULL`
- No `MD5` MACs
- No `SHA` (SHA-1) only MACs where avoidable

---

## 4. Mozilla SSL Config Profiles

The Mozilla SSL Configuration Generator (https://ssl-config.mozilla.org) is the authoritative reference. Three profiles:

### Modern Profile

**Minimum client support:** Firefox 63, Chrome 70, Safari 12.1, Edge 75, Android 10, Java 11, OpenSSL 1.1.1.

**Protocols:** TLS 1.3 only.

**Cipher suites:** TLS 1.3 handles this automatically — no TLS 1.2 cipher string needed.

**DHParam:** Not needed (TLS 1.3 uses ephemeral ECDH).

**Use when:** Internal services, APIs where you control the client, high-security environments, fintech, anything where you can afford to drop older clients entirely.

#### nginx — Modern Profile

```nginx
# Mozilla Modern Profile — TLS 1.3 only
# Minimum: nginx 1.17.7, OpenSSL 1.1.1

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;

    ssl_certificate     /etc/ssl/certs/example.com.crt;
    ssl_certificate_key /etc/ssl/private/example.com.key;

    # Modern: TLS 1.3 only
    ssl_protocols TLSv1.3;

    # TLS 1.3 has no configurable cipher string in the traditional sense
    # Optionally restrict to strongest TLS 1.3 suites:
    # ssl_conf_command Ciphersuites TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256;

    ssl_prefer_server_ciphers off;

    # HSTS — 2 years, include subdomains, preload (see HSTS section before enabling)
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

    # OCSP stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/ssl/certs/ca-chain.crt;
    resolver 1.1.1.1 8.8.8.8 valid=300s;
    resolver_timeout 5s;

    # Session cache — TLS 1.3 uses its own ticket mechanism, but keep for compat
    ssl_session_timeout 1d;
    ssl_session_cache shared:MozSSL:10m;  # ~40,000 sessions
    ssl_session_tickets off;  # Disable for PFS

    ssl_buffer_size 4k;  # Reduce to minimize first-flight latency
}
```

#### Apache — Modern Profile

```apache
# Mozilla Modern Profile — TLS 1.3 only
# Minimum: Apache 2.4.36, OpenSSL 1.1.1

<VirtualHost *:443>
    ServerName example.com

    SSLEngine on
    SSLCertificateFile      /etc/ssl/certs/example.com.crt
    SSLCertificateKeyFile   /etc/ssl/private/example.com.key
    SSLCertificateChainFile /etc/ssl/certs/ca-chain.crt

    # Modern: TLS 1.3 only
    SSLProtocol             -all +TLSv1.3

    # TLS 1.3 cipher suites are auto-negotiated; no SSLCipherSuite needed for 1.3 only
    # To explicitly set TLS 1.3 ciphers:
    # SSLOpenSSLConfCmd Ciphersuites "TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256"

    SSLHonorCipherOrder     off
    SSLCompression          off
    SSLSessionTickets       off

    # OCSP stapling
    SSLUseStapling          On
    SSLStaplingCache        shmcb:/var/run/ocsp(128000)
    SSLStaplingResponderTimeout 5
    SSLStaplingReturnResponderErrors off

    # HSTS
    Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
</VirtualHost>

# OCSP stapling cache (global context, outside VirtualHost)
SSLUseStapling On
SSLStaplingCache shmcb:/var/run/ocsp(128000)
```

---

### Intermediate Profile (Recommended Default)

**Minimum client support:** Firefox 27, Chrome 30, IE 11, Edge, Opera 17, Safari 9, Android 5.0, Java 8u31.

**Protocols:** TLS 1.2 + TLS 1.3.

**This is the right choice for most public-facing production services.** It balances strong security with broad compatibility, covering virtually every client in active use.

#### nginx — Intermediate Profile

```nginx
# Mozilla Intermediate Profile
# Minimum: nginx 1.17.7, OpenSSL 1.1.1

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;

    ssl_certificate     /etc/ssl/certs/example.com.crt;
    ssl_certificate_key /etc/ssl/private/example.com.key;

    # Intermediate: TLS 1.2 + TLS 1.3
    ssl_protocols TLSv1.2 TLSv1.3;

    # Intermediate cipher suite — ECDHE+AEAD preferred, DHE fallback for PFS
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305;

    # Let the client pick between AES-GCM and ChaCha20 based on its capabilities
    ssl_prefer_server_ciphers off;

    # DHParam for DHE ciphers (generate: openssl dhparam -out /etc/ssl/dhparam.pem 2048)
    ssl_dhparam /etc/ssl/dhparam.pem;

    # ECDH curve preference
    ssl_ecdh_curve X25519:prime256v1:secp384r1;

    # Session cache shared across workers
    ssl_session_timeout 1d;
    ssl_session_cache shared:MozSSL:10m;  # ~40,000 sessions
    ssl_session_tickets off;

    # OCSP stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/ssl/certs/ca-chain.crt;
    resolver 1.1.1.1 8.8.8.8 valid=300s;
    resolver_timeout 5s;

    # HSTS — start conservatively, ramp up (see HSTS section)
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

    ssl_buffer_size 4k;
}
```

#### Apache — Intermediate Profile

```apache
# Mozilla Intermediate Profile
# Minimum: Apache 2.4.36, OpenSSL 1.1.1

<VirtualHost *:443>
    ServerName example.com

    SSLEngine on
    SSLCertificateFile      /etc/ssl/certs/example.com.crt
    SSLCertificateKeyFile   /etc/ssl/private/example.com.key
    SSLCertificateChainFile /etc/ssl/certs/ca-chain.crt

    # Intermediate: TLS 1.2 + TLS 1.3
    SSLProtocol             -all +TLSv1.2 +TLSv1.3

    # Intermediate cipher suite
    SSLCipherSuite          ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305

    # TLS 1.3 ciphers are negotiated separately by OpenSSL
    SSLOpenSSLConfCmd       Ciphersuites "TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256"

    SSLHonorCipherOrder     off
    SSLCompression          off
    SSLSessionTickets       off

    # DHParam
    SSLOpenSSLConfCmd       DHParameters "/etc/ssl/dhparam.pem"

    # OCSP stapling
    SSLUseStapling          On
    SSLStaplingCache        shmcb:/var/run/ocsp(128000)
    SSLStaplingResponderTimeout 5
    SSLStaplingReturnResponderErrors off

    # HSTS
    Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
</VirtualHost>

# Global context
SSLUseStapling On
SSLStaplingCache shmcb:/var/run/ocsp(128000)
```

---

### Old Profile (Legacy — Do Not Use for New Deployments)

**Minimum client support:** IE 8 on Windows XP (this should tell you everything).

**Protocols:** TLS 1.0, TLS 1.1, TLS 1.2, TLS 1.3.

**Why it exists:** Organizations that cannot immediately drop TLS 1.0/1.1 support — usually due to embedded systems, ancient enterprise software, or contractual SLA requirements with clients running legacy stacks. This is a transitional profile, not a target state.

**Includes:** 3DES (for IE 8/XP — the only reason it exists), CBC ciphers, SHA-1 MACs.

**Security tradeoffs:** Exposure to POODLE variants, BEAST (mitigated client-side), Lucky13, SWEET32 on long sessions. SWEET32 can be partially mitigated by setting a 64MB renegotiation limit (`SSLRenegBufferSize` in Apache, HTTP connection limits in nginx).

**If you must use Old profile:** Set a renegotiation limit, monitor for SWEET32, plan a migration timeline, and isolate legacy endpoints from your main TLS-terminated services.

The Old profile cipher string (nginx):
```nginx
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-SHA256:DHE-RSA-AES256-SHA256:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256;
```

Note: The Old profile explicitly excludes 3DES since OpenSSL 3.x and the Mozilla generator no longer recommends it even for legacy support. Accept the small compatibility loss — 3DES is not worth the SWEET32 exposure.

---

## 5. DHParam: When You Need It and When You Don't

### The Short Answer

| Scenario | DHParam needed? |
|----------|----------------|
| TLS 1.3 only (Modern profile) | No |
| TLS 1.2 + 1.3 with ECDHE-only ciphers | No |
| TLS 1.2 + 1.3 with DHE ciphers in cipher string | Yes |
| Mozilla Intermediate profile | Yes (for DHE fallback) |

### Why TLS 1.3 Doesn't Need DHParam

TLS 1.3 handles its own key exchange. The spec mandates ephemeral ECDH (X25519 or P-256 by default) for the key exchange. There is no DHE in TLS 1.3's key exchange options (RFC 8446 removed it). Your `ssl_dhparam` directive has no effect on TLS 1.3 sessions.

### Why ECDHE Doesn't Need DHParam

`ssl_dhparam` configures parameters for DHE (finite-field Diffie-Hellman). ECDHE uses elliptic curve parameters which are standardized and compiled into OpenSSL — no external parameter file needed. The curve is selected via `ssl_ecdh_curve` (nginx) or implicitly via the cipher string.

### When DHParam IS Needed

If your cipher string contains any `DHE-RSA-*` or `DHE-DSS-*` ciphers (finite-field DH), nginx/Apache will use a built-in 1024-bit DH group if no `ssl_dhparam` is provided. **1024-bit DH is the Logjam vulnerability.** This is why Logjam (2015) was so widespread — many servers used the default OpenSSL 1024-bit DH parameters or (worse) the shared 512-bit export DH group.

**Always generate your own 2048-bit DH parameters when using DHE ciphers:**

```bash
# Generate 2048-bit DH parameters (takes ~30 seconds, do it once)
openssl dhparam -out /etc/ssl/dhparam.pem 2048

# If you want 4096-bit (slower handshake, marginal security improvement)
openssl dhparam -out /etc/ssl/dhparam.pem 4096

# Check what you generated
openssl dhparam -in /etc/ssl/dhparam.pem -text -noout | head -5
```

**Why 2048 and not 4096?** 2048-bit DH is computationally equivalent to ~112-bit symmetric security — adequate against any known attack. 4096-bit DH doubles the RSA public key size but doesn't meaningfully improve security against realistic threats; it does add CPU overhead to every DHE handshake. Use 2048. If you're on a site that needs post-quantum resistance, DHE isn't the right tool anyway — look at post-quantum KEMs.

**Secure DH prime alternatives:** Instead of generating your own primes, you can use RFC 7919 named groups (ffdhe2048, ffdhe3072, ffdhe4096). These are audited, widely reviewed primes. OpenSSL 1.1.1+ supports them:

```bash
# Use the RFC 7919 ffdhe2048 group (pre-audited, strongly recommended)
openssl genpkey -genparam -algorithm DH -pkeyopt dh_paramgen_prime_len:2048 \
  -pkeyopt dh_paramgen_type:2 -out /etc/ssl/dhparam-ffdhe2048.pem
```

Or simply download the pre-computed ffdhe2048 PEM from the IETF and use it. The advantage over self-generated primes: no risk of weak or backdoored primes if you're skeptical of your RNG at parameter generation time.

**Set correct permissions:**
```bash
chmod 640 /etc/ssl/dhparam.pem
chown root:ssl-cert /etc/ssl/dhparam.pem
```

---

## 6. OCSP Stapling

### What It Is and Why It Matters

Without OCSP stapling, a browser receiving your certificate must make a separate HTTP request to your CA's OCSP responder to verify the certificate hasn't been revoked. This adds latency (an extra RTT to a third-party server), leaks client browsing behavior to the CA, and fails silently if the OCSP responder is unreachable (most browsers use "soft fail" — they accept the certificate anyway).

OCSP stapling moves the OCSP response from the client's problem to the server's problem. Your server fetches the OCSP response from the CA, caches it, and "staples" it to the TLS handshake. The client validates the stapled response (signed by the CA) without needing to contact the CA at all.

Benefits:
- Eliminates the extra RTT for OCSP lookup on the client side
- Improves privacy — CA doesn't see individual client IPs
- More reliable — your server controls the caching
- Required for HSTS preload when combined with OCSP Must-Staple (optional but good practice)

### nginx OCSP Stapling Configuration

```nginx
# Inside server {} block

# Enable OCSP stapling
ssl_stapling on;

# Verify the OCSP response against the trusted CA chain
ssl_stapling_verify on;

# The full CA chain (including intermediate certs) that signed your certificate
# This allows nginx to verify the OCSP response signature
# If ssl_certificate already includes the full chain, this may be redundant
# but it's explicit and safe to include
ssl_trusted_certificate /etc/ssl/certs/ca-chain-full.crt;

# DNS resolver for nginx to use when querying the OCSP responder
# Use your local resolver for speed, plus public fallbacks
# If behind a corporate network, use internal DNS
resolver 1.1.1.1 1.0.0.1 8.8.8.8 8.8.4.4 valid=300s;
resolver_timeout 5s;
```

**Important: Your ssl_certificate must include the full chain (leaf cert + intermediate(s)), not just the leaf cert.** nginx needs the intermediate to build the OCSP request.

Verify OCSP stapling is working:
```bash
# Connect to your server and check for OCSP response in the handshake
openssl s_client -connect example.com:443 -servername example.com -status < /dev/null 2>&1 | grep -A 20 "OCSP response"

# You should see: OCSP Response Status: successful (0x0)
# If you see "no response sent": nginx hasn't fetched the OCSP response yet
# Wait ~1 minute after restart and try again
```

**OCSP response caching:** nginx caches the OCSP response in memory and re-fetches it before expiry (typically 24-72 hours depending on the CA). The response is shared across all workers via the shared memory segment used for `ssl_session_cache`.

**Troubleshooting common OCSP issues:**

1. **"OCSP response not sent" in testing** — nginx fetches OCSP response lazily on first request. After a restart, the first few connections won't have it. Let-Encrypt OCSP responses are typically valid for 7 days; nginx re-fetches when < half the validity remains.

2. **Resolver not reachable** — If your server can't reach the public DNS resolvers, OCSP fetching fails silently. Check with: `dig @1.1.1.1 ocsp.int-x3.letsencrypt.org`

3. **Missing intermediate in ssl_trusted_certificate** — nginx will log: `ssl_stapling_verify: certificate could not be resolved`. Include the full chain file.

4. **Let's Encrypt + OCSP stapling** — Works well. Certbot typically provides a chain file at `/etc/letsencrypt/live/example.com/chain.pem` for `ssl_trusted_certificate`.

### Apache OCSP Stapling Configuration

```apache
# Global context (httpd.conf or ssl.conf — outside VirtualHost)
SSLUseStapling On
SSLStaplingCache shmcb:/var/run/apache2/ocsp(131072)

# Inside VirtualHost
SSLUseStapling On
SSLStaplingResponderTimeout 5
SSLStaplingReturnResponderErrors Off
# Optional: force specific OCSP responder URL (override what's in the cert)
# SSLStaplingForceURL http://ocsp.example-ca.com/
```

Apache uses `mod_ssl`'s built-in OCSP stapling. The `shmcb` cache is shared memory — allocate enough for your expected number of different certificates (128K is fine for a single cert, scale up for many vhosts).

---

## 7. HSTS Rollout Strategy

### What HSTS Does

HTTP Strict Transport Security (HSTS, RFC 6797) instructs browsers to only communicate with your domain over HTTPS, refusing HTTP connections and preventing SSL stripping attacks. Once a browser has seen the HSTS header, it will upgrade all requests to HTTPS internally — before any network connection is made — for the duration of `max-age`.

### The Risk of Going Too Big Too Fast

If you set `max-age=31536000; includeSubDomains` on day one and then:
- A subdomain isn't yet on HTTPS (e.g., an old internal tool at `legacy.example.com`)
- Your certificate expires and can't be renewed
- You misconfigure TLS and can't fix it immediately

...then users are locked out with no easy recovery path. HSTS is a commitment, not a preference. Roll it out carefully.

### The Rollout Ladder

**Stage 0: Prerequisites (before any HSTS)**

- Ensure ALL subdomains you care about are serving valid HTTPS (if you plan to use `includeSubDomains`)
- Ensure certificate auto-renewal is working (test a dry-run renewal)
- Ensure HTTP→HTTPS redirects are in place for all hosts
- Confirm your TLS configuration passes SSL Labs at A or better

**Stage 1: Short max-age — test the waters**

```nginx
add_header Strict-Transport-Security "max-age=300" always;
```

`max-age=300` is 5 minutes. Leave this for at least a week. Monitor for broken HTTPS access, certificate warnings, mixed-content issues. Check your access logs for HTTP requests that should be HTTPS. Verify subdomains.

**Stage 2: One day**

```nginx
add_header Strict-Transport-Security "max-age=86400" always;
```

Run for a week. Monitor error rates, broken workflows, subdomain issues.

**Stage 3: One month — add includeSubDomains**

```nginx
add_header Strict-Transport-Security "max-age=2592000; includeSubDomains" always;
```

`includeSubDomains` extends HSTS to ALL subdomains. This is the moment where unprotected subdomains become a problem. Audit your DNS zone for every subdomain and verify HTTPS works before this step.

**Stage 4: One year — production target**

```nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

Run here indefinitely unless you're pursuing preload listing.

**Stage 5: Two years + preload directive**

```nginx
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
```

Only add `preload` when you're actually ready to submit to the HSTS preload list (see Section 8). Adding the directive alone doesn't add you to the list — you must actively submit. But the directive must be present before submission.

### The `always` Keyword in nginx

Without `always`, nginx only sends the `Strict-Transport-Security` header on 200 OK responses. With `always`, it's sent on all response codes including redirects, errors, and 304s. Use `always` — you need HSTS to reach the browser regardless of the response status.

### HSTS in Apache

```apache
# Requires mod_headers
# In VirtualHost (HTTPS only — never send HSTS on HTTP)
Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
```

**Critical: Never send HSTS on your HTTP (port 80) VirtualHost.** HSTS on HTTP is ignored by browsers (it only counts when received over HTTPS), but it's confusing and wrong. Your HTTP VirtualHost should only redirect:

```apache
<VirtualHost *:80>
    ServerName example.com
    Redirect permanent / https://example.com/
    # No HSTS header here
</VirtualHost>
```

---

## 8. HSTS Preload

### What Preload Is

The HSTS preload list is a hardcoded list of domains that major browsers (Chrome, Firefox, Safari, Edge, Opera) include at compile time. These domains are delivered as HTTPS-only even before the user has ever visited them — no first-visit HTTP request exists to deliver the HSTS header.

The list is maintained at https://hstspreload.org and Chrome's version at https://chromium.googlesource.com/chromium/src/+/main/net/http/transport_security_state_static.json

### Requirements for Submission

To be accepted into the preload list, your domain must:

1. **Serve a valid HTTPS certificate** — No certificate errors, no self-signed, not expired.
2. **Redirect HTTP to HTTPS** on the same host (example.com:80 → example.com:443).
3. **Serve an HSTS header** on the HTTPS base domain with:
   - `max-age` of at least **31536000** (1 year) — 63072000 (2 years) is preferred
   - `includeSubDomains` directive present
   - `preload` directive present
4. **All subdomains must be HTTPS-capable** — The `includeSubDomains` requirement means every subdomain will be HSTS-preloaded. A subdomain that's HTTP-only will be inaccessible from preloaded browsers.

Checking eligibility:
```bash
# Use the hstspreload.org checker
curl "https://hstspreload.org/api/v2/preloadable?domain=example.com"
# Returns JSON with eligible: true/false and any issues
```

### How to Submit

1. Visit https://hstspreload.org
2. Enter your domain
3. Review the eligibility check results
4. Click "Submit" (requires the `preload` directive to be present in your live HSTS header)
5. The domain enters a pending list. Propagation to Chrome stable typically takes **2-3 months**.

### Propagation Timeline

- Chrome Canary/Dev: Weeks after merge
- Chrome Beta: A few weeks after Canary
- Chrome Stable: **2-3 months** after submission
- Firefox: Syncs from Chrome's list periodically; similar timeline
- Safari: Apple maintains its own list; timing varies (months to a year)
- Edge: Uses Chrome's list (Chromium-based)

**Your domain must remain compliant indefinitely.** Browsers won't remove a preloaded domain just because your HSTS header changes — they rely on the compiled list.

### How to Remove (It Is Hard — Read This Before Submitting)

Removal from the HSTS preload list is intentionally slow and difficult. This is by design — accidental or malicious removal would undermine the security model.

**To request removal:**

1. Visit https://hstspreload.org/removal
2. Submit your domain for removal
3. Removal enters a queue. The actual removal from browsers takes **months to years** — it must ship in a browser release and propagate to all installed browser versions.

**During the removal period (potentially 1+ year):**
- All users on browsers that preloaded your domain before the removal will still enforce HSTS
- There is no way to force an update
- Browsers with older cached versions of the preload list are unaffected

**What "removal" means practically:** Your domain is removed from the preload list source. Over time, as browsers update and the list is recompiled/updated, the domain gradually disappears from the preload list in new browser versions. Existing browser installations gradually stop enforcing it as they update.

**You cannot emergency-remove yourself from preload in any timely way.** This is why the rollout ladder in Section 7 is not optional — treat preload submission as a permanent, near-irreversible decision.

**Only submit to preload if:**
- You are 100% certain ALL subdomains will always be HTTPS
- You have robust certificate renewal automation
- You have no legacy subdomains that are HTTP-only
- You have no plans to retire the domain and let it expire (expired preloaded domains are inaccessible)

---

## 9. Session Tickets vs Session Cache

### The Problem with Session Tickets

TLS session resumption exists to avoid the full handshake on every connection. Two mechanisms exist in TLS 1.2:

1. **Session IDs** (server-side cache): Server stores session state; client sends a session ID; server looks it up.
2. **Session tickets** (client-side state): Server encrypts session state with a server-side ticket key; client stores the ticket; server decrypts it on reconnect. No server-side storage needed.

Session tickets sound convenient but have a serious problem: **the session ticket encryption key is a long-lived symmetric key on the server.** If that key is compromised, an attacker can decrypt all past and future sessions that used tickets encrypted by that key. This undermines perfect forward secrecy.

The attack: An adversary records encrypted traffic. Later, they obtain the session ticket key (via server compromise, key disclosure, or compelled handover). They can now decrypt all recorded sessions that used session resumption. If you have PFS-capable cipher suites but session tickets enabled with a static key, you have nominally "perfect" forward secrecy but practically none for resumed sessions.

**Disable session tickets:**

```nginx
ssl_session_tickets off;
```

```apache
SSLSessionTickets off
```

### Session Cache for Multi-Worker Environments

When you disable tickets, fall back to session IDs with a shared server-side cache. In nginx with multiple worker processes (the normal production setup), each worker has its own memory space. A session cached by worker 1 won't be found by worker 2, making resumption hit-or-miss without a shared cache.

```nginx
# Shared cache: all workers share this 10MB segment (~40,000 sessions)
ssl_session_cache shared:MozSSL:10m;
ssl_session_timeout 1d;
ssl_session_tickets off;
```

The `shared:MozSSL:10m` syntax creates a named shared memory zone (`MozSSL`) of 10 megabytes. The name is arbitrary but must match across the config. Each TLS session consumes roughly 256 bytes, so 10MB holds ~40,000 sessions.

**Session timeout:** `1d` (1 day) is Mozilla's recommendation. Setting it too short defeats resumption; too long unnecessarily extends the window during which a stolen session cache entry could be replayed.

### TLS 1.3 Session Resumption

TLS 1.3 uses a different mechanism: **session tickets are built into the protocol differently** — the server sends a `NewSessionTicket` message post-handshake containing a PSK (Pre-Shared Key) and a ticket nonce. However, TLS 1.3 session tickets in OpenSSL still respect `ssl_session_tickets off` in nginx/Apache — disabling this also disables TLS 1.3 resumption tickets.

**Trade-off:** Disabling all session tickets means every TLS 1.3 connection requires a full 1-RTT handshake with no resumption optimization. For most web services, this is acceptable — the performance impact is minimal because TLS 1.3's 1-RTT handshake is already fast. For very high-throughput or latency-sensitive APIs, consider the trade-off carefully.

**The compromise:** If you must enable tickets for performance, rotate the ticket encryption key frequently (every few hours). nginx doesn't do this automatically unless you implement key rotation via a shared key file that's replaced on a cron schedule. HAProxy and some other load balancers support automatic ticket key rotation.

---

## 10. TLS 1.3 Early Data (0-RTT) Risks

### What 0-RTT Is

TLS 1.3 supports a 0-RTT mode (Early Data) where a client resuming a session can send application data in its very first flight — before the handshake completes. For HTTP, this means the initial GET request can be sent in the same packet as the ClientHello, achieving zero-RTT for resumed sessions.

### The Replay Attack Problem

0-RTT data is **not replay-protected at the TLS layer.** An active attacker (or a misbehaving network middlebox) can take a recorded 0-RTT ClientHello+EarlyData and replay it to the server. The server cannot distinguish a legitimate 0-RTT resumption from a replayed one.

**Impact:** Any state-changing operation carried in 0-RTT data (POST requests, mutations, financial transactions, authentication tokens) can be replayed. The attacker doesn't need to decrypt the data — they just replay the recorded bytes.

Mitigations that help but don't fully solve it:
- Application-level idempotency: If 0-RTT is only allowed for GET requests or truly idempotent operations, the replay is harmless. But enforcing this requires the application (or reverse proxy) to be aware of 0-RTT status.
- Replay nonces: Server maintains a short-lived list of seen nonces from 0-RTT data. Replay detection window is limited (typically the session ticket validity period).
- Single-use tickets: Invalidate a ticket after one 0-RTT use. Complex to implement correctly at scale.

### nginx 0-RTT (ssl_early_data)

nginx 1.15.3+ supports `ssl_early_data`. It is **disabled by default.** Do not enable it unless you have carefully analyzed your application's safety:

```nginx
# DO NOT enable this unless you understand the replay implications
# ssl_early_data on;

# If you do enable it, detect early data requests in the application:
# The $ssl_early_data variable is "1" when the request arrived in early data
# Use it to reject non-idempotent requests:
# proxy_set_header Early-Data $ssl_early_data;
```

### Apache 0-RTT

Apache does not support 0-RTT as of Apache 2.4.x. This is arguably a feature — no config needed.

### Recommendation

**Leave `ssl_early_data off` (the default) for all production services.** The latency gain from 0-RTT (1 RTT → 0 RTT on session resumption) is marginal for typical web workloads. The replay attack surface is real and hard to fully mitigate at the infrastructure layer. Browsers already make the first request faster via other mechanisms (connection coalescing, HTTP/2 multiplexing, TCP Fast Open).

The only use case where 0-RTT is worth the risk: extremely latency-sensitive, read-only APIs where every millisecond counts and replay of any request is provably harmless (truly idempotent GET endpoints with no side effects, no rate limiting bypass potential).

---

## 11. Testing Your Configuration

### SSL Labs (ssllabs.com/ssltest)

The most widely used public TLS testing tool. Provides a letter grade (A+, A, B, C, F) and detailed analysis.

URL: https://www.ssllabs.com/ssltest/analyze.html?d=example.com&hideResults=on

What it checks:
- Protocol support (penalizes TLS 1.0, TLS 1.1, SSLv3)
- Cipher suite strength and ordering
- Certificate chain validity and trust
- Forward secrecy support
- BEAST mitigation
- POODLE (both SSLv3 and TLS)
- Heartbleed
- FREAK, Logjam, DROWN
- OCSP stapling
- Session resumption (tickets and cache)
- HSTS presence and duration
- Public Key Pinning (deprecated, but checked)
- HTTP → HTTPS redirect
- Certificate transparency
- Mixed content (for web pages)

**Limitations:**
- Public test — your domain will appear in the recent results list unless you use `hideResults=on`
- Tests from Qualys's servers — doesn't reflect NAC or IP-restricted endpoints
- Slow (~1 minute per test)
- Doesn't test every cipher in real time — uses a known list

### testssl.sh

A comprehensive command-line TLS tester. Open source, runs locally, works against any host regardless of network restrictions. Far more detailed than SSL Labs for cipher enumeration.

Project: https://testssl.sh / https://github.com/drwetter/testssl.sh

Install:
```bash
git clone --depth 1 https://github.com/drwetter/testssl.sh.git
cd testssl.sh
```

#### Essential testssl.sh Commands

```bash
# Full standard test (equivalent to SSL Labs, but local)
./testssl.sh example.com

# Test a specific port
./testssl.sh example.com:8443

# Test with SNI
./testssl.sh --sni example.com 203.0.113.10

# Test only protocols
./testssl.sh --protocols example.com

# Test only cipher suites
./testssl.sh --ciphers example.com

# Test for specific vulnerabilities
./testssl.sh --vulnerable example.com
# Checks: BEAST, POODLE, FREAK, Logjam, DROWN, LUCKY13, SWEET32, ROBOT,
#          Heartbleed, CCS injection, ticketbleed, CRIME, BREACH, RC4, etc.

# Check individual vulnerabilities
./testssl.sh --poodle example.com
./testssl.sh --beast example.com
./testssl.sh --heartbleed example.com
./testssl.sh --freak example.com
./testssl.sh --logjam example.com
./testssl.sh --sweet32 example.com
./testssl.sh --robot example.com
./testssl.sh --lucky13 example.com

# Check certificate
./testssl.sh --certificate example.com

# Check OCSP stapling
./testssl.sh --stapling example.com

# Check HTTP headers (HSTS, etc.)
./testssl.sh --headers example.com

# Check session resumption (tickets and cache)
./testssl.sh --session example.com

# Output to a log file
./testssl.sh --logfile /tmp/tls-report.txt example.com

# JSON output for programmatic processing
./testssl.sh --jsonfile /tmp/tls-report.json example.com

# Parallel testing for speed
./testssl.sh --parallel example.com

# Test from a specific local IP/interface (useful for multi-homed servers)
./testssl.sh --local-addr 192.168.1.10 example.com

# Test against a local nginx before DNS cutover
./testssl.sh --ip 127.0.0.1 --sni example.com example.com:443
```

#### testssl.sh Interpretation

Key output sections and what to look for:

```
 Testing protocols via sockets

 SSLv2      not offered (OK)
 SSLv3      not offered (OK)
 TLS 1      not offered (OK)
 TLS 1.1    not offered (OK)
 TLS 1.2    offered (OK)
 TLS 1.3    offered (OK)
```

```
 Testing cipher categories

 NULL ciphers (no encryption)                     not offered (OK)
 Anonymous NULL Ciphers (no authentication)       not offered (OK)
 Export ciphers (w/o ADH+NULL)                    not offered (OK)
 LOW: 64 Bit + DES, RC[2,4], MD5 (w/o export)   not offered (OK)
 Triple-DES Ciphers / IDEA                        not offered (OK)
 Obsolete CBC ciphers (AES, CAMELLIA, ARIA etc)  not offered (OK)
 Strong encryption (AEAD ciphers)                 offered (OK)
```

Any `offered` on the deprecated cipher categories or protocol versions is a failure.

### openssl s_client Commands

Direct TLS testing from the command line. Essential for debugging specific issues.

```bash
# Basic connectivity and certificate check
openssl s_client -connect example.com:443 -servername example.com < /dev/null

# Check OCSP stapling
openssl s_client -connect example.com:443 -servername example.com -status < /dev/null 2>&1 | \
  grep -A 20 "OCSP response"

# Force specific TLS version (verify old versions are rejected)
openssl s_client -connect example.com:443 -servername example.com -tls1 < /dev/null
openssl s_client -connect example.com:443 -servername example.com -tls1_1 < /dev/null
# Both should show: no peer certificate available / ssl handshake failure

# Force specific TLS version (verify modern versions work)
openssl s_client -connect example.com:443 -servername example.com -tls1_2 < /dev/null
openssl s_client -connect example.com:443 -servername example.com -tls1_3 < /dev/null

# Enumerate supported cipher suites (brute-force approach)
for cipher in $(openssl ciphers 'ALL:eNULL' | tr ':' ' '); do
  result=$(echo "" | openssl s_client -connect example.com:443 -cipher "$cipher" \
    -servername example.com 2>&1)
  if echo "$result" | grep -q "Cipher is"; then
    echo "ACCEPTED: $cipher"
  fi
done

# Check session ticket behavior
openssl s_client -connect example.com:443 -servername example.com \
  -reconnect < /dev/null 2>&1 | grep -E "Session-ID|TLS session ticket"

# Check certificate chain
openssl s_client -connect example.com:443 -servername example.com \
  -showcerts < /dev/null 2>&1 | grep -E "^s:|^i:"

# Test with specific cipher suite
openssl s_client -connect example.com:443 -servername example.com \
  -cipher ECDHE-RSA-AES256-GCM-SHA384 < /dev/null

# Check HSTS header
curl -sI https://example.com | grep -i strict-transport-security

# Check certificate expiry
openssl s_client -connect example.com:443 -servername example.com < /dev/null 2>&1 | \
  openssl x509 -noout -dates

# Full certificate info
openssl s_client -connect example.com:443 -servername example.com < /dev/null 2>&1 | \
  openssl x509 -noout -text
```

### nmap TLS Audit

```bash
# TLS version and cipher suite audit via nmap scripts
nmap --script ssl-enum-ciphers -p 443 example.com

# Check for specific vulnerabilities
nmap --script ssl-poodle -p 443 example.com
nmap --script ssl-dh-params -p 443 example.com  # Logjam check
nmap --script ssl-heartbleed -p 443 example.com
```

---

## 12. A+ SSL Labs Checklist

SSL Labs grades on multiple dimensions. A+ requires an A grade plus HSTS with `max-age >= 15552000` (6 months). Here is every requirement:

### Certificate

- [ ] Certificate is issued by a trusted CA (not self-signed, not expired)
- [ ] Certificate has correct CN/SAN for the hostname being tested
- [ ] Certificate uses SHA-256 or better signature hash (not SHA-1)
- [ ] Certificate uses RSA 2048+ or ECDSA P-256+
- [ ] Full certificate chain is served (leaf + all intermediates)
- [ ] No chain issues (extra certs, wrong order, self-signed intermediate)
- [ ] Certificate does not have a Trust Anchor Signed error
- [ ] Certificate matches the private key
- [ ] OCSP stapling is enabled and the stapled response is valid

### Protocols

- [ ] SSLv2 is disabled
- [ ] SSLv3 is disabled
- [ ] TLS 1.0 is disabled
- [ ] TLS 1.1 is disabled
- [ ] TLS 1.2 is enabled
- [ ] TLS 1.3 is enabled (not required for A, but improves score and is expected)

### Cipher Suites

- [ ] No NULL ciphers
- [ ] No anonymous ciphers (aNULL)
- [ ] No export ciphers
- [ ] No RC4 ciphers
- [ ] No 3DES / DES / IDEA
- [ ] No MD5 MACs
- [ ] No CBC ciphers without forward secrecy (static RSA + AES-CBC is worst offender)
- [ ] Forward secrecy supported (ECDHE or DHE ciphers present)
- [ ] AEAD ciphers are used (AES-GCM, ChaCha20-Poly1305)

### Key Exchange

- [ ] DH parameter is 2048 bits or larger (if DHE ciphers are used)
- [ ] ECDH curves are strong (P-256 / X25519 minimum, avoid P-192)
- [ ] No Logjam vulnerability (no common/weak DH primes)
- [ ] No FREAK vulnerability (no export-grade RSA key exchange)
- [ ] No DROWN vulnerability (no SSLv2 support on any IP sharing the private key)

### Vulnerabilities (must be "not vulnerable" on all)

- [ ] BEAST: not vulnerable (TLS 1.0 disabled or CBC mitigated)
- [ ] POODLE (SSLv3): not vulnerable (SSLv3 disabled)
- [ ] POODLE (TLS): not vulnerable (no CBC padding oracle)
- [ ] Heartbleed: not vulnerable (patched OpenSSL)
- [ ] CRIME: not vulnerable (TLS compression disabled — `ssl_comp off` or just use OpenSSL without zlib)
- [ ] BREACH: not vulnerable (HTTP compression awareness — BREACH is HTTP-level, not TLS-level; use CSRF tokens or compression exemptions for secrets)
- [ ] ROBOT: not vulnerable (RSA PKCS#1 v1.5 padding oracle — patched in OpenSSL 1.0.2m+)
- [ ] Lucky13: not vulnerable (AEAD ciphers avoid this entirely)
- [ ] SWEET32: not vulnerable (3DES/DES disabled)
- [ ] FREAK: not vulnerable (no export ciphers)
- [ ] Logjam: not vulnerable (DHparam ≥ 2048 bits)
- [ ] Ticketbleed: not vulnerable (only affects F5 devices, not nginx/Apache)

### HTTP Security Headers (for A+ and broader security)

- [ ] HSTS header present with `max-age >= 15552000` (required for A+)
- [ ] HSTS `includeSubDomains` present (recommended)
- [ ] HTTP to HTTPS redirect in place
- [ ] No mixed content (HTTP resources loaded from HTTPS pages)

### Configuration Details

- [ ] `ssl_prefer_server_ciphers off` — let the client pick between AES-GCM and ChaCha20 (Firefox/Chrome will prefer ChaCha20 on mobile, which is fine)
- [ ] No TLS compression (`ssl_comp off` in nginx — it's off by default in modern nginx)
- [ ] Session ticket key rotation (or tickets disabled)
- [ ] Shared session cache configured for multi-worker setups
- [ ] Resolver configured for OCSP stapling

### HSTS for A+ specifically

SSL Labs awards A+ only when `Strict-Transport-Security` is present with:
- `max-age` of at least **15552000** (180 days)
- Sent over HTTPS

```nginx
# Minimum for A+:
add_header Strict-Transport-Security "max-age=15552000" always;

# Recommended (per Mozilla):
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
```

---

## 13. Complete Reference Configs

### nginx — Production Intermediate Profile (Complete Server Block)

```nginx
# /etc/nginx/snippets/ssl-intermediate.conf
# Mozilla Intermediate Profile, nginx 1.18+, OpenSSL 1.1.1+
# Generated reference: https://ssl-config.mozilla.org/

ssl_certificate     /etc/ssl/certs/example.com.fullchain.crt;
ssl_certificate_key /etc/ssl/private/example.com.key;

ssl_session_timeout 1d;
ssl_session_cache shared:MozSSL:10m;
ssl_session_tickets off;

ssl_dhparam /etc/ssl/dhparam.pem;

ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305;
ssl_prefer_server_ciphers off;

ssl_stapling on;
ssl_stapling_verify on;
ssl_trusted_certificate /etc/ssl/certs/ca-chain.crt;
resolver 1.1.1.1 8.8.8.8 valid=300s;
resolver_timeout 5s;
```

```nginx
# /etc/nginx/sites-available/example.com

# HTTP → HTTPS redirect
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;

    # For Let's Encrypt ACME challenge (if using certbot webroot)
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

# HTTPS server
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;

    server_name example.com www.example.com;

    include snippets/ssl-intermediate.conf;

    # HSTS — 2-year max-age with preload
    # Only use preload directive if you've submitted to hstspreload.org
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

    # Additional security headers
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Reduce buffer size to improve TTFB for small TLS records
    ssl_buffer_size 4k;

    root /var/www/example.com;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### Apache — Production Intermediate Profile (Complete VirtualHost)

```apache
# /etc/apache2/sites-available/example.com.conf

# HTTP → HTTPS redirect
<VirtualHost *:80>
    ServerName example.com
    ServerAlias www.example.com

    # Let's Encrypt ACME challenge
    Alias /.well-known/acme-challenge/ /var/www/certbot/.well-known/acme-challenge/
    <Directory /var/www/certbot/.well-known/acme-challenge/>
        Options None
        AllowOverride None
        Require all granted
    </Directory>

    RewriteEngine On
    RewriteCond %{REQUEST_URI} !^/.well-known/acme-challenge/
    RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [R=301,L]
</VirtualHost>

# HTTPS server
<VirtualHost *:443>
    ServerName example.com
    ServerAlias www.example.com

    DocumentRoot /var/www/example.com

    SSLEngine on
    SSLCertificateFile      /etc/ssl/certs/example.com.crt
    SSLCertificateKeyFile   /etc/ssl/private/example.com.key
    SSLCertificateChainFile /etc/ssl/certs/ca-chain.crt

    # Mozilla Intermediate Profile
    SSLProtocol             -all +TLSv1.2 +TLSv1.3
    SSLCipherSuite          ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305
    SSLOpenSSLConfCmd       Ciphersuites "TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256"
    SSLOpenSSLConfCmd       DHParameters "/etc/ssl/dhparam.pem"
    SSLHonorCipherOrder     off
    SSLCompression          off
    SSLSessionTickets       off

    # OCSP Stapling
    SSLUseStapling          On
    SSLStaplingResponderTimeout 5
    SSLStaplingReturnResponderErrors Off

    # HSTS
    Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"

    # Additional security headers
    Header always set X-Content-Type-Options "nosniff"
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set Referrer-Policy "strict-origin-when-cross-origin"
</VirtualHost>

# Global context (apache2.conf or ssl.conf)
SSLUseStapling On
SSLStaplingCache shmcb:/var/run/apache2/ocsp(131072)
```

### Generate DHParam and Set Up — Quick Reference

```bash
# Generate 2048-bit DH parameters (do this once, not on every deployment)
openssl dhparam -out /etc/ssl/dhparam.pem 2048

# Verify the parameters
openssl dhparam -in /etc/ssl/dhparam.pem -check -text -noout | grep -E "DH Parameters|private"

# Set permissions
chmod 640 /etc/ssl/dhparam.pem
chown root:ssl-cert /etc/ssl/dhparam.pem

# Test nginx configuration before reloading
nginx -t

# Reload nginx (graceful — no dropped connections)
nginx -s reload
# or
systemctl reload nginx

# Test Apache configuration
apachectl configtest

# Graceful Apache reload
systemctl reload apache2
```

### Quick Verification After Deployment

```bash
# 1. Verify TLS 1.0 and 1.1 are rejected
openssl s_client -connect example.com:443 -tls1 2>&1 | grep -E "handshake failure|no peer"
openssl s_client -connect example.com:443 -tls1_1 2>&1 | grep -E "handshake failure|no peer"

# 2. Verify TLS 1.2 and 1.3 work
echo "Q" | openssl s_client -connect example.com:443 -tls1_2 2>&1 | grep "Protocol"
echo "Q" | openssl s_client -connect example.com:443 -tls1_3 2>&1 | grep "Protocol"

# 3. Verify OCSP stapling (run a couple times — first may not have response cached)
openssl s_client -connect example.com:443 -servername example.com -status < /dev/null 2>&1 | \
  grep -A 5 "OCSP Response Status"

# 4. Verify HSTS header
curl -sI https://example.com | grep -i strict

# 5. Verify session tickets are disabled
openssl s_client -connect example.com:443 -servername example.com \
  -reconnect < /dev/null 2>&1 | grep -i "ticket"
# Should show: no session ticket

# 6. Check cipher negotiated
echo "Q" | openssl s_client -connect example.com:443 -servername example.com 2>&1 | \
  grep "Cipher is"

# 7. Run testssl.sh full audit
./testssl.sh --full example.com

# 8. Run SSL Labs (command-line trigger, check results in browser)
echo "Manual: https://www.ssllabs.com/ssltest/analyze.html?d=example.com&hideResults=on"
```

---

## Appendix: Common Mistakes and How to Avoid Them

### Sending HSTS on HTTP VirtualHost

HSTS sent over unencrypted HTTP is ignored by browsers and is incorrect per the spec. It also gives a false sense of security. Always send HSTS from the HTTPS server block only.

### Forgetting `always` in nginx add_header

Without `always`, nginx skips the header on non-200 responses. Error pages, redirects, and 5xx responses won't carry HSTS, meaning a redirect through an HTTP intermediate step could expose the user.

### Using `ssl_prefer_server_ciphers on` with Modern Ciphers

This setting made sense when servers needed to enforce AES-GCM over RC4. Today, all the recommended cipher suites are strong. Setting this to `on` prevents clients from selecting ChaCha20-Poly1305 when it would be faster for them (mobile without AES-NI). Set it to `off`.

### Not Including Intermediate Certificates

A certificate chain missing the intermediate will fail for many clients (mobile browsers, curl without the system trust store). Always concatenate: `cat example.com.crt intermediate.crt > fullchain.crt` and use the fullchain file for `ssl_certificate`.

### Generating DHParam on Every Deployment

DHParam generation is slow and only needs to be done once (or once a year for rotation). Generating it in a deploy script or container startup adds unnecessary delay. Pre-generate it and include it in your secrets management system.

### Setting DHParam When Using Only ECDHE Ciphers

If your cipher string contains only `ECDHE-*` ciphers (no `DHE-*`), the `ssl_dhparam` directive is unused. This is not harmful but is misleading. The Mozilla Modern profile doesn't need dhparam at all.

### Not Rotating Session Ticket Keys

If session tickets are enabled (not recommended), the default behavior in nginx is to use a static key that never changes. An attacker who obtains this key can decrypt all sessions. Either disable tickets (`ssl_session_tickets off`) or implement automatic key rotation.

### Using a Public DNS Resolver for OCSP Without Considering Latency

If your server is in a restricted network environment (air-gapped datacenter, VPC with no outbound internet), using `resolver 8.8.8.8` will cause OCSP fetching to silently fail. Use your internal DNS resolver, or ensure outbound access to the CA's OCSP responder is available.

### Testing with SSLv2/SSLv3 Built Into Old OpenSSL

If your `openssl s_client -tls1` command returns an error about "Unknown option" rather than a handshake failure from the server, your local OpenSSL client is too old and doesn't support the `-tls1` flag for version-pinning. Use a modern OpenSSL (1.1.1+) for testing.

---

## References

- Mozilla SSL Configuration Generator: https://ssl-config.mozilla.org/
- Mozilla Server Side TLS Wiki: https://wiki.mozilla.org/Security/Server_Side_TLS
- testssl.sh: https://testssl.sh / https://github.com/drwetter/testssl.sh
- SSL Labs Server Test: https://www.ssllabs.com/ssltest/
- HSTS Preload List: https://hstspreload.org
- RFC 8446 — TLS 1.3: https://www.rfc-editor.org/rfc/rfc8446
- RFC 8996 — Deprecating TLS 1.0 and 1.1: https://www.rfc-editor.org/rfc/rfc8996
- RFC 7465 — Prohibiting RC4: https://www.rfc-editor.org/rfc/rfc7465
- RFC 6797 — HTTP Strict Transport Security: https://www.rfc-editor.org/rfc/rfc6797
- RFC 7919 — Negotiated Finite Field DH Groups: https://www.rfc-editor.org/rfc/rfc7919
- NIST SP 800-52 Rev 2 — TLS Guidelines: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-52r2.pdf
- SWEET32 Attack: https://sweet32.info
- Logjam Attack: https://weakdh.org
- DROWN Attack: https://drownattack.com
