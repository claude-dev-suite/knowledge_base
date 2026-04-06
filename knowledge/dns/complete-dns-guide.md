# Complete DNS Guide for Senior Sysadmins

A practical, opinionated reference. If you're looking for theory, read RFC 1035. If you're looking for what
actually works in production under pressure at 2am, this is the document.

---

## Table of Contents

1. [DNS Record Types — Complete Reference](#dns-record-types)
2. [TTL Strategy and Migration Playbook](#ttl-strategy)
3. [Cloudflare: Proxy vs DNS-Only](#cloudflare-proxy-vs-dns-only)
4. [Route53 Routing Policies](#route53-routing-policies)
5. [DNSSEC Setup and Validation](#dnssec)
6. [Split-Horizon DNS](#split-horizon-dns)
7. [Propagation Debugging](#propagation-debugging)
8. [Email Authentication: SPF, DKIM, DMARC](#email-authentication)

---

## 1. DNS Record Types — Complete Reference {#dns-record-types}

### A Record

Maps a hostname to an IPv4 address. The bread and butter of DNS.

```
example.com.        300   IN  A  203.0.113.10
www.example.com.    300   IN  A  203.0.113.10
```

**Edge cases:**
- You can have multiple A records for the same name. DNS clients receive all of them but typically use
  the first. This is not load balancing — it is round-robin at best and unreliable at worst. Use a
  proper load balancer or Route53 weighted/multivalue routing instead.
- Trailing dot matters in zone files (fully-qualified). Most UIs strip it; most CLIs require it when
  writing raw zone data.
- `0.0.0.0` is a valid A record value but returns effectively nothing useful — do not use it to
  "disable" a hostname. Use a low TTL and remove the record, or point to a maintenance server.

---

### AAAA Record

Maps a hostname to an IPv6 address. Structurally identical to A, just a different address family.

```
example.com.      300  IN  AAAA  2001:db8::1
www.example.com.  300  IN  AAAA  2001:db8::1
```

**Edge cases:**
- Most resolvers try AAAA first, then fall back to A (Happy Eyeballs, RFC 6555). If your IPv6
  stack is broken, clients will experience delay before falling back. Monitor both address families.
- Do not point AAAA at a host that does not actually have IPv6 connectivity. You will create
  mysterious timeouts that are genuinely hard to debug because many developers do not think to check
  IPv6 first.
- IPv6 compression rules: `2001:0db8:0000:0000:0000:0000:0000:0001` equals `2001:db8::1`. Both
  are valid in DNS. Use the compressed form in your zone file for readability.

---

### CNAME Record

Maps a hostname (alias) to another hostname (canonical name). The resolver then looks up the target
hostname.

```
www.example.com.      300  IN  CNAME  example.com.
blog.example.com.     300  IN  CNAME  mysite.wpengine.com.
cdn.example.com.      300  IN  CNAME  d1234.cloudfront.net.
```

**The CNAME-at-Apex Problem**

This is the most common DNS misconception. RFC 1034 prohibits CNAME records at the zone apex (the
naked domain, e.g., `example.com` itself) because a CNAME must be the only record for that name,
but the apex always has SOA and NS records. Any DNS implementation that allows a CNAME at the apex
is violating the RFC.

This matters in practice when you want:
- `example.com` → `myapp.heroku.com`
- `example.com` → a CDN edge

You cannot do it with a real CNAME. Your options:

**Option A: ALIAS / ANAME records (vendor-specific)**
Route53, Cloudflare, DNSimple, and others support a flattening record type variously called ALIAS
(Route53), ANAME (DNSimple), or just handled transparently (Cloudflare). It behaves like a CNAME
but gets resolved server-side and returned as an A/AAAA record. This is not in any RFC — it is a
proprietary extension.

```
# Route53 syntax (via console/API — no zone file representation)
example.com.  A  ALIAS  d1234.cloudfront.net.

# Cloudflare: just enter a CNAME for the apex — they flatten it automatically
example.com.  CNAME  d1234.cloudfront.net.   ; Cloudflare flattens this to A records
```

**Option B: Hard-code the A records**
If the upstream service gives you static IPs (it usually does not), just use A records. Not ideal
because upstream IPs change without notice.

**Option C: Redirect naked domain to www**
Use an HTTP redirect at the registrar or a dedicated redirect service. Works for web traffic only.
Does not help for email or other protocols.

**CNAME chain limit:** Resolvers typically follow up to 8 CNAME hops before giving up. Keep chains
short (1-2). Never create CNAME loops.

**CNAME and other records:** You cannot have any other record type at the same name as a CNAME
(except DNSSEC records). This means `www.example.com IN CNAME foo.com` and
`www.example.com IN TXT "something"` cannot coexist. This breaks SPF setups on aliases — put
your TXT records on the actual hostname.

---

### MX Record

Specifies mail servers for a domain. Always points to a hostname, never an IP, and — critically —
that hostname must be an A or AAAA record, never a CNAME.

```
example.com.  300  IN  MX  10  mail1.example.com.
example.com.  300  IN  MX  20  mail2.example.com.
example.com.  300  IN  MX  10  aspmx.l.google.com.
```

Syntax: `priority  hostname`. Lower priority value = higher preference.

**Why MX must not point to a CNAME:**

RFC 2181 section 10.3 prohibits it explicitly. The practical reason: when a sending MTA looks up
your MX record, it gets the hostname. It then does an A/AAAA lookup on that hostname. If the
hostname is a CNAME, the chain lookup creates additional round trips and some MTAs (particularly
older or stricter ones, including some anti-spam infrastructure) will outright reject the connection
or fail delivery silently. Postfix, Exim, and sendmail all warn about or reject MX-to-CNAME.

Always create a real A record for your mail hostname:
```
mail1.example.com.  300  IN  A  203.0.113.25
```

**Multiple MX records and failover:**
- Same priority = equal preference (round-robin / load balancing)
- Different priority = fallback. The sending MTA tries the lowest number first; if it fails,
  tries the next.
- If you have a backup MX pointed at a server that does not know your domain, that server will
  accept mail and bounce it. This is worse than having no backup MX. Only add a backup MX if
  it is configured to queue-and-forward.

**Null MX (RFC 7505):**
To explicitly declare a domain sends no mail (blocking backscatter):
```
example.com.  300  IN  MX  0  .
```
The single dot means "no mail server." This prevents spammers from bouncing mail through your domain.

---

### NS Record

Delegates a zone to authoritative nameservers. NS records at the apex are glue — they tell the
parent zone who is authoritative for your zone.

```
example.com.  86400  IN  NS  ns1.cloudflare.com.
example.com.  86400  IN  NS  ns2.cloudflare.com.
```

**Critical:** The NS records in your zone file must exactly match what your registrar has on file.
Mismatch = partial or complete resolution failure depending on which server the resolver hits.
Always verify with `whois example.com` and compare to `dig NS example.com @8.8.8.8`.

**Delegation:** You can create NS records for subdomains to delegate them to a different DNS provider:
```
internal.example.com.  86400  IN  NS  ns1.internal-provider.com.
internal.example.com.  86400  IN  NS  ns2.internal-provider.com.
```
Any query for `*.internal.example.com` will be referred to those nameservers. You may need glue
records at the parent if the NS hostnames are within the delegated zone (circular dependency).

**Do not lower NS TTL during migrations.** NS TTL changes are often ignored by resolvers and can
cause unpredictable behavior. Change your registrar's NS records and wait for propagation (usually
24-48 hours for NS delegation).

---

### SOA Record

Start of Authority — every zone has exactly one. Contains zone metadata.

```
example.com.  3600  IN  SOA  ns1.cloudflare.com.  dns.cloudflare.com. (
    2024040601  ; Serial (YYYYMMDDnn)
    10800       ; Refresh (3 hours)
    3600        ; Retry (1 hour)
    604800      ; Expire (7 days)
    300         ; Negative TTL (NXDOMAIN cache time)
)
```

Fields:
- **MNAME:** Primary authoritative nameserver
- **RNAME:** Responsible email address (@ replaced with .) — `dns.cloudflare.com` = `dns@cloudflare.com`
- **Serial:** Monotonically increasing. Secondary nameservers refresh when this changes. The
  `YYYYMMDDnn` format (e.g., `2024040601`) is convention, not required — but stick to it.
- **Refresh:** How often secondaries poll the primary for changes (seconds)
- **Retry:** How long secondaries wait before retrying a failed refresh
- **Expire:** How long secondaries keep serving zone data if they cannot reach the primary
- **Negative TTL:** How long resolvers cache NXDOMAIN responses (RFC 2308)

Most managed DNS providers handle SOA automatically. You almost never edit it directly. The one
time you care: if you manually manage BIND zones, bump the serial every time you make changes.

---

### TXT Record

Free-form text. Abused for everything from domain verification to email authentication.

```
example.com.              300  IN  TXT  "v=spf1 include:_spf.google.com -all"
_dmarc.example.com.       300  IN  TXT  "v=DMARC1; p=reject; rua=mailto:dmarc@example.com"
google._domainkey.example.com.  300  IN  TXT  "v=DKIM1; k=rsa; p=MIGfMA0..."
```

**Multiple TXT records on the same name:**
DNS allows multiple TXT records at the same name. SPF must have exactly one. DMARC must have
exactly one at `_dmarc.domain`. Domain ownership verification records can stack — each verifier
adds their own. Check with `dig TXT example.com` to see all of them.

**String length:** A single TXT string is limited to 255 characters. For longer values (DKIM
keys, long SPF records), split into multiple strings within the same record:
```
google._domainkey.example.com.  300  IN  TXT  "v=DKIM1; k=rsa; p=MIGfMA0GCS" "qGSIb3DQEBAQ..."
```
DNS concatenates adjacent strings automatically. Most UIs handle this transparently.

---

### PTR Record

Reverse DNS — maps an IP to a hostname. Lives in the `in-addr.arpa.` zone (IPv4) or
`ip6.arpa.` zone (IPv6).

```
; IPv4: octets reversed
10.113.0.203.in-addr.arpa.  300  IN  PTR  mail1.example.com.

; IPv6: nibble-by-nibble reversed
1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.8.b.d.0.1.0.0.2.ip6.arpa.  300  IN  PTR  mail1.example.com.
```

**Important:** PTR zones are delegated by the IP block owner (your ISP, your cloud provider, your
datacenter). You cannot create PTR records for IPs you do not own the reverse zone for. For cloud
VMs, set PTR via the provider's console (AWS Elastic IPs → "Reverse DNS", GCP compute instance
settings, etc.).

**PTR for mail:** Sending mail without a valid PTR record for your sending IP will get you
rejected or spam-filtered by major providers (Gmail, Microsoft 365, etc.). This is non-negotiable.
The PTR does not have to match your mail hostname exactly, but it must forward-confirm:
`PTR(IP) → hostname`, then `A(hostname) → same IP`. This is called forward-confirmed reverse DNS
(FCrDNS).

---

### SRV Record

Service locator — used by protocols that need to discover service endpoints (SIP, XMPP, LDAP,
Kubernetes, etc.).

Syntax: `_service._proto.name.  TTL  IN  SRV  priority  weight  port  target.`

```
_sip._tcp.example.com.     300  IN  SRV  10  60  5060  sip1.example.com.
_sip._tcp.example.com.     300  IN  SRV  10  40  5060  sip2.example.com.
_sip._tcp.example.com.     300  IN  SRV  20   0  5060  sip-backup.example.com.
_xmpp-server._tcp.example.com.  300  IN  SRV  5  0  5269  xmpp.example.com.
_ldap._tcp.example.com.    300  IN  SRV  0  0  389  ldap.example.com.
```

Fields:
- **Priority:** Lower = preferred (same as MX)
- **Weight:** Relative weight among records with the same priority. Higher weight = more traffic.
  Used for weighted load balancing within a priority group. Set to 0 when there is only one
  record at that priority.
- **Port:** TCP/UDP port of the service
- **Target:** Hostname of the server. Must be an A/AAAA, not a CNAME (same rule as MX).

**Real-world use:** Kubernetes uses SRV for headless service discovery. Microsoft Teams/Skype
for Business requires specific SRV records. If your XMPP or SIP setup is not working, start
by verifying your SRV records with `dig SRV _sip._tcp.example.com`.

---

### CAA Record

Certification Authority Authorization — restricts which CAs can issue SSL/TLS certificates for
your domain. Introduced in RFC 6844. Major CAs are required to check CAA before issuance.

```
example.com.  300  IN  CAA  0  issue "letsencrypt.org"
example.com.  300  IN  CAA  0  issue "digicert.com"
example.com.  300  IN  CAA  0  issuewild "letsencrypt.org"
example.com.  300  IN  CAA  0  iodef "mailto:security@example.com"
```

Tags:
- **issue:** Authorizes the CA to issue single-name or SAN certificates for the exact domain
- **issuewild:** Authorizes the CA to issue wildcard certificates (`*.example.com`). If absent
  but `issue` is present, wildcards are not authorized even with an `issue` tag.
- **iodef:** Where the CA should report policy violations (email or URL)

**Flag field:** The first number (0 or 128). 0 = non-critical (CA may proceed if they do not
understand the tag). 128 = critical (CA must refuse if they do not understand the tag). Always
use 0 unless you have a very specific reason.

**To block all issuance:**
```
example.com.  300  IN  CAA  0  issue ";"
```

**Inheritance:** CAA records are inherited by subdomains if no CAA record exists at the subdomain
level. `www.example.com` uses `example.com`'s CAA if no CAA is set at `www.example.com`.

---

### DNSKEY and DS Records

These are DNSSEC records. See the [DNSSEC section](#dnssec) for full setup details.

**DNSKEY:** Published in the zone, contains the public key used to sign zone records.

```
example.com.  3600  IN  DNSKEY  256  3  13  oJMRESz5E4gYzS/q6XDrvU1qMPYIjCWzJaOau8XNEZeqCYKD...
example.com.  3600  IN  DNSKEY  257  3  13  mdsswUyr3DPW132mOi8V9xESWE8jTo0dxCjjnopKl+GqJxpVXck...
```

Flags:
- **256** = Zone Signing Key (ZSK) — signs the actual zone records
- **257** = Key Signing Key (KSK) — signs the DNSKEY RRset, anchors the chain of trust
- Algorithm **13** = ECDSA P-256 with SHA-256 (recommended over RSA for new zones; smaller
  keys, faster validation)

**DS Record:** Published in the *parent* zone (at your registrar), contains a hash of your
KSK. Creates the chain of trust from parent to child zone.

```
example.com.  3600  IN  DS  2371  13  2  1F987CC6583E92DF0890718C42D43B89F179D8B0DA4B70A2A2BA9E4B00C94D4
```

Fields: key tag, algorithm, digest type, digest. You get these values from your DNS provider
after signing your zone; you submit them to your registrar's NS management UI.

---

## 2. TTL Strategy and Migration Playbook {#ttl-strategy}

TTL (Time to Live) is measured in seconds. It tells resolvers and caches how long to hold a
record before re-querying. It does not invalidate existing caches — once cached, a record
lives until its TTL expires, no matter what you change at the authoritative server.

### The Migration Playbook

This is the sequence you follow every time you make a DNS change that matters:

**Step 1: Lower the TTL (at least 24 hours before the change)**

Find the current TTL:
```bash
dig example.com A +noall +answer
# example.com.  86400  IN  A  203.0.113.10
#                ^^^^^
#                current TTL is 86400 (24 hours)
```

Lower it to 300 (5 minutes) or less. Commit this change and wait.

**Step 2: Wait 2x the old TTL**

If the old TTL was 86400 seconds (24 hours), wait 48 hours. This ensures every resolver that
cached the record under the old TTL has expired it and picked up the new (low) TTL. If you skip
this wait and make your change, resolvers that cached the record will still serve the old IP
for up to 24 hours after your change.

```
Old TTL: 86400   →   Wait: 172800 seconds (48 hours)
Old TTL: 3600    →   Wait: 7200 seconds (2 hours)
Old TTL: 300     →   Wait: 600 seconds (10 minutes)
```

**Step 3: Make the change**

Update the A/AAAA/CNAME/MX record to the new value. Now propagation takes at most 300 seconds
(your new low TTL) instead of 86400 seconds. You have dramatically reduced your blast radius.

**Step 4: Verify from multiple vantage points**

```bash
dig @8.8.8.8 example.com A +short      # Google resolver
dig @1.1.1.1 example.com A +short      # Cloudflare resolver
dig @9.9.9.9 example.com A +short      # Quad9 resolver
```

All three should return the new value within 5 minutes. If any still shows the old value after
10 minutes, investigate — either the change did not propagate to all authoritative servers, or
you have a caching issue.

**Step 5: Raise TTL back (after confirming everything works)**

Once you are confident the change is correct and stable (give it 30-60 minutes of monitoring),
restore the TTL to its production value:
- `300` for records that change occasionally (good default for most things)
- `3600` for stable records (www, mail servers)
- `86400` for very stable records (NS, SOA)

Do not leave records at TTL 60 or 300 in production indefinitely. Low TTLs increase load on
your authoritative nameservers and can cause resolution latency spikes.

### Emergency Rollback

If something breaks immediately after a DNS change:
1. Revert the record immediately (you already lowered the TTL, so reverting propagates fast)
2. New caches will pick up the reverted record within TTL seconds
3. Existing sessions are already connected and do not use DNS again — DNS is only for initial
   connection establishment. The breakage affects new connections only.

### Common TTL Mistakes

- **"Just change it and hope it propagates fast"** — No. Cache is king. Know your TTL before
  you touch anything.
- **Lowering TTL 5 minutes before the change** — Too late. You just shortened future cache
  time but did nothing for existing cached entries.
- **Leaving TTL at 60 in production** — Hammers your authoritative servers. For a busy domain,
  this adds up to enormous unnecessary DNS query volume.
- **Forgetting that Cloudflare proxy caches independently** — Even after DNS propagates, edge
  cache may serve old content. Purge Cloudflare cache explicitly after DNS+origin changes.

---

## 3. Cloudflare: Proxy vs DNS-Only {#cloudflare-proxy-vs-dns-only}

### The Two Modes

**DNS-only (Grey Cloud):** Cloudflare acts as a plain authoritative nameserver. Queries return
your actual origin IP. No Cloudflare processing happens on the traffic.

**Proxied (Orange Cloud):** Cloudflare returns its own IP (an Anycast range) instead of your
origin IP. Traffic flows through Cloudflare's edge, which applies WAF rules, CDN caching,
DDoS mitigation, and other features. Your origin IP is hidden from the public internet.

### What the Orange Cloud Gives You

- DDoS mitigation at the edge
- WAF (Web Application Firewall)
- CDN caching for static assets
- HTTP/2 and HTTP/3 (QUIC) termination
- SSL termination at the edge (with Full or Full Strict SSL modes)
- IP reputation and bot management
- Rate limiting
- Page Rules and Transform Rules

### What Breaks Behind the Orange Cloud

**Direct TCP connections:** Cloudflare only proxies HTTP (80) and HTTPS (443) by default. If
you have a service on a non-standard port, it will not be proxied. Cloudflare does support
additional ports (e.g., 2052, 2053, 2082, 2083, 2086, 2087, 2095, 2096, 8080, 8443, 8880)
for HTTP/HTTPS proxying, but anything else — raw TCP on arbitrary ports, UDP services, custom
protocols — will not work through the orange cloud.

**SSH:** Never put your SSH hostname behind the orange cloud. `ssh user@server.example.com`
will hang or get rejected because Cloudflare does not proxy port 22 in DNS proxy mode. Use
a grey-cloud subdomain (`ssh.example.com` in grey cloud) or use Cloudflare Tunnel with
`cloudflared` for zero-trust SSH access.

**Non-HTTP protocols:**
- FTP (21) — not proxied
- SMTP (25, 465, 587) — not proxied. **Never proxy your mail hostname through Cloudflare.**
  Always set `mail.example.com` and any MX target hostnames to grey cloud.
- Custom game server ports — not proxied
- Database connections (MySQL 3306, PostgreSQL 5432, etc.) — not proxied
- SFTP — not proxied

**IP whitelisting on your origin:** When orange cloud is on, your origin server only sees
Cloudflare's IP ranges as the source of connections. If you have firewall rules whitelisting
"trusted" IPs at the origin, you need to allow Cloudflare's IP ranges and rely on the
`CF-Connecting-IP` or `X-Forwarded-For` header for the real client IP. This also means
fail2ban, rate limiting, and IP-based access controls at the origin need adjustment.

**Direct origin access bypasses all protection:** Any attacker who knows your origin IP can
bypass Cloudflare entirely by connecting directly. After enabling Cloudflare proxy, restrict
your origin's firewall to only accept connections from Cloudflare's IP ranges:
- IPv4: https://www.cloudflare.com/ips-v4
- IPv6: https://www.cloudflare.com/ips-v6

### IP Exposure Risks with Grey Cloud

When you set a record to grey cloud (DNS-only), the query response contains your real origin IP.
This IP is then cached by resolvers globally and logged by anyone who queries it. Specifically:

- **Historical DNS databases** (SecurityTrails, Shodan, Censys, etc.) will have your IP indexed.
  Even if you later switch to orange cloud, the IP is already out there.
- **Certificate Transparency logs** may reveal hostnames you did not intend to be public.
- **Reverse IP lookup:** Once an attacker knows your IP, they can enumerate all hostnames
  pointing to it and identify patterns.

Mitigation:
- Use grey cloud only for records that genuinely need it (mail, SSH jump hosts)
- For high-security origins, route all traffic through Cloudflare Tunnel (`cloudflared`) so
  the origin never needs to be publicly reachable at all
- Rotate origin IPs after switching from grey to orange cloud if the origin was publicly
  accessible for a significant period

### Cloudflare Page Rules (Legacy) and Modern Equivalents

Page Rules match URL patterns and apply actions. As of 2024, Cloudflare is phasing out classic
Page Rules in favor of Rules (Transform Rules, Redirect Rules, Cache Rules, Configuration Rules).
Classic Page Rules still work but new projects should use the modern rules.

**Common Page Rule use cases:**

Force HTTPS redirect:
```
Match: http://example.com/*
Action: Always Use HTTPS
```

Cache everything (including HTML):
```
Match: https://example.com/static/*
Action: Cache Level: Cache Everything
        Edge Cache TTL: 1 month
```

Bypass cache for admin paths:
```
Match: https://example.com/admin/*
Action: Cache Level: Bypass
```

Disable security features for known API consumers:
```
Match: https://api.example.com/webhook/*
Action: Security Level: Essentially Off
        Browser Integrity Check: Off
```

**Modern equivalents:**
- Redirect Rules replace "Forwarding URL" page rules
- Cache Rules replace "Cache Level" page rules
- Transform Rules handle header modification
- Configuration Rules replace per-URL security settings

**Important:** Page Rules are processed in priority order. First match wins. Arrange them
carefully — put more specific patterns before broader ones.

---

## 4. Route53 Routing Policies {#route53-routing-policies}

Route53 routing policies control how DNS responses are chosen when multiple records exist for
the same name. They are configured per-record, not per-zone.

### Simple Routing

One record, one response. No health checks, no logic.

**Use when:** Single origin, static configuration, no failover requirements. The default.

```
# API console: Type A, Value: 203.0.113.10, no routing policy set
```

With simple routing you can return multiple values (multiple IPs), and Route53 returns all of
them in random order. The client picks one. This is not real load balancing — use Multivalue
Answer or a load balancer for that.

---

### Weighted Routing

Multiple records with the same name, each with a weight. Traffic distribution is proportional
to weights.

**Use when:**
- A/B testing (90% traffic to v1, 10% to v2)
- Gradual rollouts (start at 5%, ramp to 100%)
- Load balancing across origins in a single region with different capacities

```
# Record 1: 203.0.113.10, weight 90
# Record 2: 203.0.113.20, weight 10
# → 90% to first server, 10% to second
```

**Setting weight to 0** excludes a record from routing without deleting it. Useful for taking
an instance out of rotation without losing the configuration.

**With health checks:** If a record is unhealthy, Route53 redistributes its weight proportionally
among healthy records. If all records are unhealthy, Route53 returns all of them (fail-open).

---

### Latency-Based Routing

Routes to the AWS Region that provides the lowest latency for the resolver's location. Route53
maintains latency maps between regions and resolver locations.

**Use when:**
- Multi-region deployments on AWS
- Optimizing for end-user response time
- You have equivalent infrastructure in multiple regions (us-east-1, eu-west-1, ap-southeast-1)

**Important caveats:**
- Latency is measured from the resolver, not the end user. For ISPs with centralized resolvers,
  latency routing may not accurately represent user latency. Use latency routing with geolocation
  routing together if you need more precision.
- You must have a resource in each region you configure. Route53 does not route to "nearest
  region" — it routes to the specific resource registered for that region.
- Latency data is updated periodically by AWS. Do not expect real-time sensitivity to network
  conditions.

---

### Failover Routing

Active-passive pair. Primary handles all traffic when healthy. Secondary receives traffic only
when the primary fails its health check.

**Use when:**
- DR/HA setup with a warm standby
- Maintenance windows (flip to secondary, do maintenance, flip back)
- Primary SaaS service with a "we're down" page fallback

```
# Primary record: 203.0.113.10, Failover: PRIMARY, health check attached
# Secondary record: 203.0.113.20, Failover: SECONDARY (health check optional)
```

**Health check configuration matters enormously here.** If your health check is too aggressive
(short intervals, low failure threshold), you will get unnecessary failovers. If it is too
lenient (long intervals, high failure threshold), failover will be slow.

Recommended health check settings for production failover:
- Request interval: 10 seconds (fast) or 30 seconds (standard)
- Failure threshold: 3 (flip after 3 consecutive failures)
- This gives you 30-90 second failover time from actual outage to DNS change

Route53 health checks can be:
- Endpoint checks (HTTP, HTTPS, TCP to your origin)
- Calculated health checks (logical AND/OR of other checks)
- CloudWatch alarm-based (check any CloudWatch metric)

---

### Geolocation Routing

Routes based on the geographic location of the DNS resolver (inferred from its IP). This is
not IP geolocation of the end user — it is resolver geolocation.

**Use when:**
- Regulatory compliance (EU users must be served from EU infrastructure)
- Language/locale-specific content
- Blocking access from certain countries

```
# Record for users in Europe: eu-origin.example.com
# Record for users in US: us-origin.example.com
# Default record (catch-all): global-origin.example.com
```

**Always configure a default record.** If you do not, and a resolver comes from a location
with no matching geolocation rule, Route53 returns NOERROR with an empty answer set (NODATA).
This effectively breaks DNS for those users.

**Geolocation is not a security control.** VPNs, proxies, and resolvers that route through
different countries will route around your geolocation rules. Use it for optimization and
content targeting, not for access restriction.

---

### Geoproximity Routing

Like geolocation, but based on the physical location of both the resource and the resolver,
with a configurable bias that allows you to shift the routing boundary.

**Use when:**
- You have resources in custom locations (non-AWS, on-premises)
- You want to shift traffic boundaries — e.g., favor us-east-1 for more than just the US East
- Traffic Engineering between adjacent regions

**Bias:** A value from -99 to +99. Positive bias expands the geographic area that routes to a
resource; negative bias shrinks it. Default is 0 (pure proximity).

Requires Route53 Traffic Flow (additional cost). If you just need basic multi-region routing,
latency-based routing is simpler and cheaper.

---

### IP-Based Routing

Routes based on CIDR blocks of the resolver or client IP. You provide a list of CIDRs and
map them to specific endpoints.

**Use when:**
- You have dedicated corporate IP ranges that should always hit specific infrastructure
- ISPs that peer directly and should use a private endpoint
- Specific cloud provider CIDR ranges that should go to optimized endpoints

This is the most granular routing policy and requires the most maintenance (keeping CIDR lists
up to date). Use it when geolocation and latency routing are not precise enough.

---

### Multivalue Answer Routing

Returns up to 8 healthy A/AAAA records chosen at random. Not a replacement for a load
balancer, but useful for simple distribution across multiple origins.

**Use when:**
- You have 2-8 equivalent origin servers
- You want basic health-check-aware DNS round-robin
- You do not want to run a dedicated load balancer

The key differentiator from Simple routing with multiple values: Multivalue supports health
checks. Unhealthy records are excluded from responses. Simple routing with multiple values
returns all values regardless of health.

**Limitation:** DNS-level load balancing is inherently imprecise. Session persistence,
connection draining, and accurate load distribution are load balancer features. Use ALB/NLB
on AWS for anything requiring those.

---

### Routing Policy Selection Guide

```
Single origin, no HA needed                   → Simple
Gradual rollout / A-B testing                 → Weighted
Multi-region, minimize latency                → Latency-Based
Active-passive DR / warm standby              → Failover
Compliance / content by country               → Geolocation
Optimize by proximity with traffic shifting   → Geoproximity
Corporate / ISP-specific routing              → IP-Based
Basic multi-origin with health checks         → Multivalue Answer
```

Policies can be combined in Route53 Traffic Flow for complex scenarios (e.g., Failover within
a region, Latency-based between regions).

---

## 5. DNSSEC Setup and Validation {#dnssec}

DNSSEC adds cryptographic signatures to DNS records, allowing resolvers to verify that records
have not been tampered with in transit. It does not encrypt DNS traffic — use DNS-over-HTTPS
(DoH) or DNS-over-TLS (DoT) for privacy. DNSSEC provides integrity and authenticity.

### When You Need DNSSEC

- Regulated industries (financial services, government, healthcare)
- When your domain is a high-value target for DNS spoofing/cache poisoning attacks
- When your registrar and DNS provider both support it easily (i.e., you have no excuse not to)

Modern providers (Cloudflare, Route53, Google Cloud DNS) handle DNSSEC key management and
rotation automatically. There is very little reason not to enable it.

### Chain of Trust

```
Root zone (.) → signed with root KSK (managed by IANA)
  ↓ DS record in root points to TLD KSK
TLD zone (com.) → signed with TLD KSK
  ↓ DS record in TLD points to your zone KSK
Your zone (example.com.) → signed with your KSK
  ↓ KSK signs DNSKEY RRset (which contains ZSK)
Your ZSK → signs all other records in the zone
```

A DNSSEC-validating resolver walks this chain from the root. If any link breaks (expired key,
mismatched DS record, signing failure), validation fails and the resolver returns SERVFAIL.

### Setup Steps

**Step 1: Enable DNSSEC at your DNS provider**

On Cloudflare:
1. Dashboard → your domain → DNS → Settings → DNSSEC → Enable
2. Cloudflare generates the keys, signs the zone, and gives you the DS record values.

On Route53:
1. Enable DNSSEC signing on the hosted zone
2. Route53 creates a KSK (you can provide your own via AWS KMS, or let Route53 manage it)
3. Route53 gives you the DS record to add at your registrar

**Step 2: Add DS Record at your registrar**

This is the step most people miss or do incorrectly. The DS record must be published in the
parent zone (the TLD's nameservers), which is controlled by your registrar.

Registrar (e.g., Namecheap, GoDaddy, Google Domains/Squarespace):
1. Navigate to DNSSEC settings for your domain
2. Enter the DS record values provided by your DNS provider:
   - Key tag (a number, e.g., 2371)
   - Algorithm (13 for ECDSA P-256, 8 for RSA SHA-256)
   - Digest type (2 for SHA-256)
   - Digest (the hash string)
3. Save and wait for propagation

**Step 3: Verify the chain of trust**

```bash
# Check that DS record exists in parent zone
dig DS example.com @8.8.8.8 +short

# Full DNSSEC validation check
dig example.com +dnssec @8.8.8.8

# Online verification
# https://dnsviz.net/d/example.com/dnssec/
# https://dnssec-analyzer.verisignlabs.com/example.com
```

A properly configured zone will show `ad` (Authenticated Data) flag in dig responses when
queried against a validating resolver.

**Step 4: Key Rollover**

Keys must be periodically rotated. Managed providers (Cloudflare, Route53) handle this
automatically using KSK and ZSK rollovers per RFC 6781. If you manage your own BIND zone:

- ZSK rollover: every 3-6 months (pre-publish method: add new ZSK, wait for TTL, switch
  signing to new ZSK, remove old ZSK)
- KSK rollover: every 1-2 years (double-signature method: add new KSK, update DS at
  registrar, wait for old DS to expire, remove old KSK)

### DNSSEC Gotchas

**The SERVFAIL trap:** Any validation failure returns SERVFAIL to clients. Common causes:
- Expired RRSIG records (zone stopped being signed, or signing failed silently)
- DS record at registrar does not match current KSK (common after key rollover)
- Negative TTL on DS record in parent zone delays recovery

Monitor DNSSEC validation continuously. An alert on SERVFAIL for your domain is not optional.

**Disabling DNSSEC:** To disable DNSSEC, remove the DS record at your registrar *first*, wait
for the DS TTL to expire (usually 1-24 hours), *then* disable signing at your DNS provider.
If you disable signing first, DNSSEC-validating resolvers will get unsigned responses but still
have a DS record in the parent, causing validation failure = SERVFAIL for all validating
resolvers. This is a serious outage.

---

## 6. Split-Horizon DNS {#split-horizon-dns}

Split-horizon (also called split-brain) DNS returns different answers to the same query
depending on where the query originates. Typically: internal clients get private IPs,
external clients get public IPs.

### Use Cases

- `api.example.com` → `10.0.1.5` internally (private IP, no proxy overhead), `203.0.113.10`
  externally (public IP with load balancer)
- `db.internal.example.com` should not resolve at all from the public internet
- Development environments where you want to override specific hostnames locally

### Method 1: /etc/hosts (Simplest, One Machine)

Not "split-horizon" in a DNS sense, but the fastest way to override a single hostname
on a single machine.

```
# /etc/hosts
10.0.1.5   api.example.com
10.0.1.10  db.example.com  db
```

`/etc/hosts` is checked before DNS (default `nsswitch.conf` order). Changes take effect
immediately, no TTL involved.

**Limitations:** Only affects the local machine. Does not scale. Forgetting to remove an
entry causes mysterious "it works on my machine" issues. Use for local development overrides
and nothing else in production.

### Method 2: Unbound — Internal Resolver with Overrides

Unbound is the right tool for internal DNS with split-horizon on Linux networks.

```conf
# /etc/unbound/unbound.conf

server:
    interface: 0.0.0.0
    access-control: 10.0.0.0/8 allow
    access-control: 0.0.0.0/0 refuse

    # Local zone for internal overrides
    local-zone: "example.com" transparent
    local-data: "api.example.com. 300 IN A 10.0.1.5"
    local-data: "db.example.com. 300 IN A 10.0.1.10"

    # Ensure internal zone does not leak to public
    local-zone: "internal.example.com" static

forward-zone:
    name: "."
    forward-addr: 8.8.8.8
    forward-addr: 1.1.1.1
```

`transparent` zone type: return local data if it exists, forward upstream for everything else.
`static` zone type: only return what is configured locally; NXDOMAIN for anything else.

### Method 3: BIND Views

BIND's `view` directive serves different zone data to different client groups.

```conf
# /etc/named.conf

acl internal_clients {
    10.0.0.0/8;
    192.168.0.0/16;
};

view "internal" {
    match-clients { internal_clients; };

    zone "example.com" IN {
        type master;
        file "/etc/bind/zones/example.com.internal";
    };
};

view "external" {
    match-clients { any; };

    zone "example.com" IN {
        type master;
        file "/etc/bind/zones/example.com.external";
    };
};
```

The internal zone file contains private IPs. The external zone file contains public IPs.
Clients on the internal network get internal answers; everyone else gets public answers.

**DNSSEC and views:** DNSSEC complicates views significantly. Each view needs its own
signing keys, and the zone serial must be kept in sync. In practice, many shops disable
DNSSEC on internal views and only enable it on the external view.

### Method 4: Cloudflare Split Tunnel (Zero Trust)

If you use Cloudflare Zero Trust (formerly Cloudflare for Teams), you can configure DNS
split tunneling within the WARP client:

1. **Cloudflare Zero Trust Dashboard** → Settings → Network → Split Tunnels
2. Add your internal domain (e.g., `internal.example.com`) to the "Exclude" list
3. DNS queries for that domain bypass Cloudflare WARP and go to your internal resolver

This works per-device (wherever the WARP client is installed) and requires your internal
DNS resolver to be reachable (via private network, VPN, or Cloudflare Tunnel).

**Local Domain Fallback:** In WARP client settings, you can configure specific domains to
always resolve via internal DNS, while all other DNS goes through Cloudflare's resolver.
This is cleaner than full VPN split tunnel for DNS-only scenarios.

---

## 7. Propagation Debugging {#propagation-debugging}

DNS problems fall into three categories:
1. **Configuration error** — wrong record, typo, wrong TTL, wrong record type
2. **Propagation delay** — correct config but cache not expired yet
3. **Resolver behavior** — some resolver is caching stale data longer than it should

The tools below let you diagnose all three.

### dig — Your Primary Tool

Install it: `apt install dnsutils` / `brew install bind` / it comes with most Linux distros.

**Basic query:**
```bash
dig example.com A
```

**Short output (just the answer):**
```bash
dig +short example.com A
dig +short example.com MX
dig +short example.com TXT
```

**Query a specific resolver (bypass your local cache):**
```bash
dig @8.8.8.8 example.com A         # Google
dig @1.1.1.1 example.com A         # Cloudflare
dig @9.9.9.9 example.com A         # Quad9
dig @208.67.222.222 example.com A  # OpenDNS
```

This is the first thing you do when you suspect a propagation issue. Query at least two
public resolvers and compare. If they disagree, propagation is still in progress.

**Full recursive resolution with +trace:**
```bash
dig +trace example.com A
```

This bypasses your local resolver and performs the full recursive lookup from the root
servers down. It shows you every step:
1. Queries root servers (`.`)
2. Gets referral to TLD servers (`com.`)
3. Gets referral to your authoritative nameservers
4. Queries your authoritative nameservers directly

This is invaluable for:
- Confirming your authoritative NS is returning the right records
- Checking if your registrar's NS delegation is correct
- Diagnosing DNSSEC chain-of-trust failures

**Check all authoritative nameservers:**
```bash
# Get NS records
dig +short NS example.com

# Query each NS directly
dig @ns1.cloudflare.com example.com A
dig @ns2.cloudflare.com example.com A
```

If one NS returns different data than the others, you have a zone sync issue (common with
self-managed primary/secondary setups).

**Check TTL remaining on cached record:**
```bash
dig @8.8.8.8 example.com A
# Look at the TTL in the answer section — this is how many seconds until
# Google's resolver will re-query for this record.
```

**Check MX records:**
```bash
dig +short MX example.com
# Returns: priority  hostname
# 10 aspmx.l.google.com.
```

**Check TXT records (SPF, DKIM, DMARC):**
```bash
dig +short TXT example.com                                 # SPF record
dig +short TXT _dmarc.example.com                          # DMARC
dig +short TXT google._domainkey.example.com               # DKIM
```

**Check PTR (reverse DNS):**
```bash
dig -x 203.0.113.10 +short
# Should return: mail1.example.com.
```

**Check DNSSEC:**
```bash
dig example.com A +dnssec @8.8.8.8
# Look for:
# - RRSIG record in ANSWER section (zone is signed)
# - "ad" flag in flags section (resolver validated the signature)
```

**Check CAA records:**
```bash
dig CAA example.com +short
```

**Check SRV records:**
```bash
dig SRV _sip._tcp.example.com +short
```

### whois — Registrar NS Check

When DNS is completely broken, check that your registrar has the right NS records. Zone
changes at your DNS provider mean nothing if the registrar is still pointing to the old
nameservers.

```bash
whois example.com | grep -i "name server"
# Name Server: ns1.cloudflare.com
# Name Server: ns2.cloudflare.com
```

Compare this to what the TLD nameservers say:
```bash
dig NS example.com @a.gtld-servers.net +short
```

These must match. If they do not, update your registrar's NS settings.

### Online Tools

- **dnschecker.org** — Check DNS propagation from dozens of geographic locations at once.
  Useful for seeing how widespread propagation is.
- **dnsviz.net** — Visual DNSSEC chain-of-trust diagram. If your DNSSEC is broken, this
  shows exactly where and why.
- **mxtoolbox.com** — MX diagnostics, blacklist checks, SPF/DKIM/DMARC validation.
- **mail-tester.com** — Send a test email to get a full score on your email authentication.
- **toolbox.googleapps.com/apps/dig/** — Google's web-based dig, useful when you do not
  have command-line access.

### Common Debugging Scenarios

**"My DNS change is not working after an hour"**
1. `dig +short example.com @8.8.8.8` — still old value? Check your original TTL. If it was
   86400, you need to wait up to 86400 seconds from when you made the change.
2. `dig +trace example.com` — does your authoritative NS return the new value? If yes,
   it is a caching issue. If no, your change did not save properly.
3. Check your DNS provider's UI/API to confirm the record is actually updated.

**"Works in Chrome but not curl"**
Chrome uses its own DNS resolver with its own cache. `curl` uses the OS resolver.
```bash
dig +short example.com     # OS resolver result
# Compare to what Chrome shows (chrome://net-internals/#dns)
```

**"Works for me but not for clients in Europe"**
Your change propagated to your local resolvers but not everywhere. Use dnschecker.org to
see propagation status globally. The clients in Europe may be hitting resolvers that cached
the record longer than your TTL (misconfigured resolver ignoring TTL). Nothing you can do
but wait or ask them to flush their local DNS.

**"SERVFAIL on everything"**
Almost certainly a DNSSEC issue. Run `dig +trace example.com` and look for the DNSSEC chain.
Check if DS record at registrar matches your current KSK. Check RRSIG expiration:
```bash
dig example.com A +dnssec @8.8.8.8 | grep RRSIG
# Check the expiration timestamp in the RRSIG record
```

---

## 8. Email Authentication: SPF, DKIM, DMARC {#email-authentication}

These three work together. None of them alone is sufficient. All three together, correctly
configured, are the baseline for sending email that does not land in spam.

### SPF (Sender Policy Framework)

SPF specifies which servers are authorized to send mail from your domain. It is a TXT record
at your apex domain.

**Syntax:**
```
v=spf1 [mechanisms] [qualifier]all
```

**Mechanisms:**
- `include:_spf.google.com` — include another domain's SPF (used for SaaS senders)
- `ip4:203.0.113.0/24` — authorize an IPv4 address or range
- `ip6:2001:db8::/32` — authorize an IPv6 range
- `a` — authorize the IP(s) in the domain's A record
- `mx` — authorize the IPs in the domain's MX records
- `a:mail.example.com` — authorize a specific hostname's A record

**All qualifier (end of record):**
- `-all` (hard fail) — **Use this.** Any server not listed is definitively unauthorized.
  Receiving MTAs should reject the message.
- `~all` (soft fail) — Servers not listed are suspicious but not definitively unauthorized.
  Receiving MTAs should accept but mark as suspect. This is the wimpy option; use it only
  during initial rollout.
- `?all` (neutral) — No policy. Effectively useless for security. Never use this.
- `+all` (pass all) — Authorizes any server. Completely defeats the purpose of SPF. Never
  ever use this.

**Use `-all`.** If you are not confident in your authorized senders, use `~all` temporarily
while you audit, then switch to `-all`. Do not leave `~all` permanently — it provides no
meaningful protection.

**Real examples:**

Google Workspace only:
```
example.com.  300  IN  TXT  "v=spf1 include:_spf.google.com -all"
```

Google Workspace + Sendgrid + your own mail server:
```
example.com.  300  IN  TXT  "v=spf1 include:_spf.google.com include:sendgrid.net ip4:203.0.113.25 -all"
```

**The 10 DNS lookup limit:**
SPF allows a maximum of 10 DNS lookups per evaluation (each `include:`, `a`, `mx`, `ptr`
mechanism counts as a lookup, and the included records' lookups count too). Exceeding 10
causes a PermError, which most receiving MTAs treat as a fail. If you use many SaaS senders,
you will hit this limit. Solutions:
- Use a service like dmarcian's SPF flattening or Authority Resolver to pre-resolve includes
- Audit and remove senders you no longer use
- Consolidate through a single outbound relay that has its own SPF

**Never put multiple SPF records at the same name.** Only one `v=spf1` record is valid. If
you have two, receivers are supposed to return PermError. Merge them into one record.

---

### DKIM (DomainKeys Identified Mail)

DKIM adds a cryptographic signature to outgoing email headers. The recipient verifies the
signature using a public key published in DNS. This proves the email was sent by someone with
access to the private key (i.e., your mail system) and that the headers have not been modified
in transit.

**DNS Record:**

Published at `selector._domainkey.domain.`:
```
google._domainkey.example.com.  300  IN  TXT  "v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC..."
```

Fields:
- `v=DKIM1` — version, always DKIM1
- `k=rsa` — key type (rsa or ed25519; ed25519 is modern and smaller)
- `p=...` — Base64-encoded public key
- `t=s` — optional: `s` means strict (subdomain signing not allowed), `y` means testing mode
  (receivers should not reject based on DKIM during testing)

**Selector:** The selector (e.g., `google`, `s1`, `mail`) is specified in the DKIM signature
header of outgoing email. It allows multiple keys per domain (different selectors for different
services). To set up DKIM, your mail provider generates a key pair and gives you the public key
to publish in DNS under their specified selector.

**Key rotation:** DKIM private keys should be rotated periodically (at minimum annually, ideally
every 6 months). Process:
1. Generate new key pair
2. Publish new public key at new selector (`s2._domainkey.example.com`)
3. Update mail system to sign with new key
4. Wait for old selector's TTL to expire
5. Remove old selector record

**2048-bit RSA minimum.** 1024-bit RSA keys are considered weak and rejected by some receivers.
Use 2048-bit RSA or ed25519 for new setups.

---

### DMARC (Domain-based Message Authentication, Reporting, and Conformance)

DMARC ties SPF and DKIM together and specifies what to do when they fail. It also enables
reporting — you receive data about who is sending mail as your domain (including spoofers).

**DNS Record:**

Published at `_dmarc.domain`:
```
_dmarc.example.com.  300  IN  TXT  "v=DMARC1; p=reject; rua=mailto:dmarc-rua@example.com; ruf=mailto:dmarc-ruf@example.com; sp=reject; adkim=s; aspf=s; pct=100"
```

Fields:
- `v=DMARC1` — version
- `p=` — policy: `none`, `quarantine`, or `reject`
- `rua=` — aggregate report destination (daily XML reports of authentication results)
- `ruf=` — forensic report destination (per-failure reports; not all senders send these)
- `sp=` — subdomain policy (defaults to `p=` if absent)
- `adkim=` — DKIM alignment: `r` (relaxed, default) or `s` (strict)
- `aspf=` — SPF alignment: `r` (relaxed, default) or `s` (strict)
- `pct=` — percentage of messages to apply policy to (1-100). Used during rollout only.
  Should always be 100 in steady state.

**Alignment:** DMARC requires either SPF or DKIM to "align" with the From header domain.
- Relaxed alignment: the authenticated domain only needs to share the organizational domain
  (e.g., `mail.example.com` aligns with `example.com`)
- Strict alignment: the authenticated domain must exactly match the From domain

Use relaxed alignment unless you have a specific reason for strict. Strict breaks legitimate
use of subdomains for sending.

### DMARC Rollout Strategy — The Only Correct Approach

Do not set `p=reject` on day one. You will break legitimate mail flows you forgot about.

**Phase 1: Monitor (p=none)**
```
"v=DMARC1; p=none; rua=mailto:dmarc@example.com; pct=100"
```
- Duration: 2-4 weeks minimum
- Action: Set up a DMARC report processor (dmarcian, Valimail, postmaster tools, or parse
  raw XML yourself)
- Goal: Identify all legitimate senders. Look for SPF/DKIM pass rates by source.
  Any legitimate sender with low pass rates needs to be fixed.

**Phase 2: Quarantine (p=quarantine)**
```
"v=DMARC1; p=quarantine; rua=mailto:dmarc@example.com; pct=10"
```
- Start at `pct=10` (10% of failing mail goes to spam instead of inbox)
- Monitor reports daily
- Fix any remaining legitimate senders that are failing
- Gradually increase pct: 10 → 25 → 50 → 100 over 2-4 weeks

**Phase 3: Reject (p=reject)**
```
"v=DMARC1; p=reject; rua=mailto:dmarc@example.com; ruf=mailto:dmarc-ruf@example.com; pct=100"
```
- Only when you are confident all legitimate mail is passing SPF or DKIM
- Monitor reports for the first few weeks — watch for legitimate mail volumes dropping
- This is your final steady-state configuration

**Do not use pct < 100 in steady state.** The `pct` parameter is a rollout tool, not a
permanent configuration. `pct=50` means 50% of failing mail still gets through — that is
not protection, it is theater.

**DMARC reports:** The `rua` reports are aggregate — daily XML files showing authentication
results by sending IP and volume. Essential during rollout; optional but recommended in
steady state to detect compromised accounts or unauthorized senders. Use a report processor;
raw XML is not human-readable at scale.

### Quick Verification Checklist

After configuring SPF, DKIM, and DMARC, verify with:

```bash
# SPF
dig +short TXT example.com | grep spf

# DKIM (replace 'google' with your selector)
dig +short TXT google._domainkey.example.com

# DMARC
dig +short TXT _dmarc.example.com

# Send a test email
# mail-tester.com — send to the address they give you, get a score
# Google Postmaster Tools — see reputation from Google's perspective
# Microsoft SNDS — see reputation for Microsoft/Outlook delivery
```

**Pass/fail signal:** When all three are correctly configured, email headers from receiving
servers will contain:
```
Authentication-Results: mx.google.com;
   dkim=pass header.i=@example.com header.s=google;
   spf=pass (google.com: domain of user@example.com designates 209.85.220.41 as permitted sender);
   dmarc=pass (p=REJECT sp=REJECT dis=NONE) header.from=example.com
```

Any `fail` or `none` in those results requires investigation.

---

## Appendix: Quick Reference

### Common DNS Record TTL Recommendations

| Record Type | Stable TTL | During Migration |
|-------------|-----------|-----------------|
| A / AAAA    | 300-3600  | 60-300          |
| CNAME       | 300-3600  | 60-300          |
| MX          | 3600      | 300             |
| TXT (SPF)   | 300-3600  | 300             |
| TXT (DKIM)  | 3600      | 3600            |
| NS          | 86400     | Do not lower    |
| SOA         | 3600      | 3600            |
| CAA         | 3600      | 3600            |

### Essential dig One-Liners

```bash
# Full DNS resolution trace from root
dig +trace example.com

# Query specific resolver, suppress extra output
dig @8.8.8.8 +short example.com A

# All record types at once (not all resolvers support ANY)
dig example.com ANY

# Check DNSSEC validation
dig +dnssec example.com A @8.8.8.8

# Reverse lookup
dig -x 203.0.113.10 +short

# Check specific port/EDNS buffer (useful for DNSSEC debugging)
dig +bufsize=4096 +dnssec example.com A

# Time the query (check resolver latency)
dig +stats example.com A | grep "Query time"
```

### Cloudflare Proxy Port Support

HTTP/HTTPS traffic is proxied through Cloudflare on these ports:
- HTTP: 80, 8080, 8880, 2052, 2082, 2086, 2095
- HTTPS: 443, 2053, 2083, 2087, 2096, 8443

Any port not on this list: use grey cloud (DNS only) or Cloudflare Spectrum (paid).

### DMARC Policy Decision Matrix

| SPF result | DKIM result | DMARC action at p=reject |
|------------|-------------|--------------------------|
| pass       | pass        | Deliver                  |
| pass       | fail        | Deliver (SPF aligned)    |
| fail       | pass        | Deliver (DKIM aligned)   |
| fail       | fail        | Reject                   |
| fail       | none        | Reject                   |
| none       | fail        | Reject                   |

Only one of SPF or DKIM needs to pass *and* be aligned with the From domain for DMARC to pass.
