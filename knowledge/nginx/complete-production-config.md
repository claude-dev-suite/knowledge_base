# NGINX Complete Production Configuration

**Audience:** Senior sysadmins, platform engineers, DevOps leads  
**Scope:** Full nginx.conf from scratch — global tuning, TLS hardening, security headers, compression, rate limiting, upstream keepalive, static caching, structured logging, and post-deploy checklist  
**Philosophy:** Be opinionated. There are dozens of ways to configure nginx. This document picks one correct way for a production TLS-terminated reverse proxy and explains every decision.

---

## Table of Contents

1. [System Prerequisites](#1-system-prerequisites)
2. [Annotated nginx.conf — Global Block](#2-annotated-nginxconf--global-block)
3. [Annotated nginx.conf — Events Block](#3-annotated-nginxconf--events-block)
4. [Annotated nginx.conf — HTTP Block](#4-annotated-nginxconf--http-block)
5. [TLS/HTTPS Server Block](#5-tlshttps-server-block)
6. [Worker Tuning Deep Dive](#6-worker-tuning-deep-dive)
7. [TLS Configuration Rationale](#7-tls-configuration-rationale)
8. [Security Headers Reference](#8-security-headers-reference)
9. [Gzip Compression](#9-gzip-compression)
10. [Rate Limiting](#10-rate-limiting)
11. [Upstream Keepalive](#11-upstream-keepalive)
12. [Static File Caching](#12-static-file-caching)
13. [JSON Log Format](#13-json-log-format)
14. [Post-Config Checklist](#14-post-config-checklist)

---

## 1. System Prerequisites

Before touching nginx.conf, the OS needs to be ready. These are not optional.

### File Descriptor Limits

```bash
# /etc/security/limits.conf  (or a file in /etc/security/limits.d/)
nginx    soft    nofile    65535
nginx    hard    nofile    65535
root     soft    nofile    65535
root     hard    nofile    65535
```

### Kernel Network Tuning

```bash
# /etc/sysctl.d/99-nginx.conf
# Backlog queue for incoming connections (match nginx backlog parameter)
net.core.somaxconn = 65535

# Maximum receive buffer — helps under burst traffic
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216

# TCP SYN backlog — protects against SYN flood
net.ipv4.tcp_max_syn_backlog = 65535

# Reuse TIME_WAIT sockets for new connections
net.ipv4.tcp_tw_reuse = 1

# How many times to retry SYN before giving up (reduces connection timeout)
net.ipv4.tcp_syn_retries = 2

# Avoid FIN_WAIT2 lingering indefinitely
net.ipv4.tcp_fin_timeout = 15
```

```bash
sysctl -p /etc/sysctl.d/99-nginx.conf
```

### TLS Certificate Preparation

```bash
# Generate 2048-bit DH parameters (run once; takes ~30s on modern hardware)
# 2048 is the practical floor; 4096 offers negligible security gain vs 5x cpu cost
openssl dhparam -out /etc/nginx/dhparam.pem 2048

# Install certbot + obtain cert (example: standalone mode)
certbot certonly --standalone -d example.com -d www.example.com
# Certbot drops certs to: /etc/letsencrypt/live/example.com/
```

### Directory Layout

```
/etc/nginx/
├── nginx.conf               # Main config (covered in full below)
├── conf.d/
│   └── example.conf         # Per-vhost config
├── snippets/
│   ├── ssl-params.conf      # TLS directives (reusable across vhosts)
│   └── security-headers.conf
├── dhparam.pem              # DH params (generated above)
└── sites-enabled/           # Symlinks if using Debian-style layout
```

---

## 2. Annotated nginx.conf — Global Block

The global (main) context controls the nginx master process and is the first thing parsed.

```nginx
# =============================================================================
# GLOBAL BLOCK
# =============================================================================

# user directive — the OS user worker processes run as.
# NEVER run workers as root. The 'nginx' user is created by package managers.
# On Debian/Ubuntu the default is www-data; on RHEL/CentOS it's nginx.
# Syntax: user user [group];
user nginx nginx;

# worker_processes — how many worker processes to spawn.
# 'auto' reads /proc/cpuinfo and sets one worker per logical CPU core.
# This is correct for I/O-heavy workloads (reverse proxy, static files).
# For CPU-heavy workloads (SSL termination at extreme scale) you might
# experiment with N-1 to leave one core for the OS, but in practice 'auto'
# is correct in the vast majority of deployments.
# Syntax: worker_processes number | auto;
# Default: 1
worker_processes auto;

# worker_cpu_affinity — pin each worker to a CPU to improve cache locality
# and reduce context-switch overhead. 'auto' lets nginx assign the mask.
# Only effective on Linux. Omit on VMs where CPU pinning has no meaning.
# Syntax: worker_cpu_affinity auto [cpumask];
worker_cpu_affinity auto;

# worker_rlimit_nofile — override the OS ulimit for open file descriptors
# per worker process. Must be >= worker_connections * 2 (each connection
# uses at minimum one fd for the client socket and typically one for the
# upstream socket or file being served).
# Must also be <= the hard limit set in /etc/security/limits.conf above.
# Syntax: worker_rlimit_nofile number;
worker_rlimit_nofile 65535;

# worker_priority — lower nice value = higher CPU scheduling priority.
# -10 gives nginx workers a scheduling edge without taking over the system.
# Range: -20 (highest) to 20 (lowest). Default: 0.
# Only change this if you have measured CPU contention issues.
# Syntax: worker_priority number;
worker_priority -10;

# pcre_jit — enable JIT compilation for PCRE regular expressions.
# Significant speedup for configs with many complex location blocks or
# map blocks. Requires PCRE library built with --enable-jit (verify with:
#   nginx -V 2>&1 | grep -o with-pcre-jit
# ).
# Syntax: pcre_jit on | off;
pcre_jit on;

# error_log — master process and worker error log.
# Levels (increasing verbosity): emerg, alert, crit, error, warn, notice, info, debug.
# 'warn' is appropriate for production: catches real problems without noise.
# 'debug' should NEVER be used in production — it logs every request detail
# and will flood disk I/O under any meaningful load.
# Syntax: error_log file [level];
# Default: logs/error.log error
error_log /var/log/nginx/error.log warn;

# pid — path where nginx writes its master process PID.
# Required by init systems (systemd, upstart) to send signals to nginx.
# Syntax: pid file;
pid /run/nginx.pid;

# timer_resolution — reduces gettimeofday() syscall frequency.
# nginx internally calls gettimeofday() to timestamp logs and compute
# timeouts. Setting this to 100ms means the clock is updated at most 10
# times per second, which is fine for logging. Reduces syscall overhead
# under high connection rates. Omit if sub-millisecond timing accuracy
# is required (unusual).
# Syntax: timer_resolution interval;
timer_resolution 100ms;
```

---

## 3. Annotated nginx.conf — Events Block

The events block controls how each worker process handles connections.

```nginx
# =============================================================================
# EVENTS BLOCK
# =============================================================================
events {

    # use — the I/O event notification mechanism.
    # nginx auto-detects the best available method, so you rarely need this.
    # Specifying 'epoll' explicitly on Linux ensures it is used even if nginx
    # was compiled with alternatives. epoll is O(1) for fd monitoring and
    # the correct choice on all modern Linux systems.
    # FreeBSD equivalent: use kqueue;
    # Syntax: use method;
    use epoll;

    # worker_connections — maximum simultaneous connections per worker.
    # This covers ALL connections: client connections + upstream connections.
    # A reverse proxy connection consumes 2 fds (client + upstream).
    # Total theoretical max connections = worker_processes * worker_connections.
    # Do NOT set higher than worker_rlimit_nofile.
    # 4096 is a sensible production default for a mid-traffic proxy.
    # Raise to 8192-16384 on high-traffic hosts with tuned OS limits.
    # Syntax: worker_connections number;
    # Default: 512
    worker_connections 4096;

    # multi_accept — when enabled, a worker accepts ALL pending connections
    # in one pass rather than one per epoll event iteration.
    # This improves throughput under burst traffic at the cost of slightly
    # uneven distribution across workers (one worker may take a burst alone).
    # For a proxy behind a load balancer where bursts are common, 'on' is
    # the correct choice. If worker distribution evenness matters more than
    # raw throughput, set to 'off'.
    # Syntax: multi_accept on | off;
    # Default: off
    multi_accept on;

}
```

---

## 4. Annotated nginx.conf — HTTP Block

The http block configures all HTTP/HTTPS behavior. Directives here apply globally across all server blocks unless overridden.

```nginx
# =============================================================================
# HTTP BLOCK
# =============================================================================
http {

    # -------------------------------------------------------------------------
    # MIME Types
    # -------------------------------------------------------------------------

    # Load the mime.types file mapping file extensions to Content-Type headers.
    # Without this, nginx sends 'application/octet-stream' for everything.
    include /etc/nginx/mime.types;

    # Default MIME type for files that don't match any type in mime.types.
    # application/octet-stream forces the browser to download rather than
    # render — safe and correct for unknown binary content.
    default_type application/octet-stream;

    # -------------------------------------------------------------------------
    # PERFORMANCE — File Serving
    # -------------------------------------------------------------------------

    # sendfile — use the sendfile(2) syscall to transfer files directly from
    # disk to socket without copying through userspace. This is the single
    # highest-impact performance setting for static file serving.
    # Always 'on' in production. Only turn off for debugging or NFS mounts
    # (where sendfile has historically caused issues).
    # Syntax: sendfile on | off;
    # Default: off
    sendfile on;

    # tcp_nopush — when combined with sendfile, nginx buffers data until it
    # has a full packet (TCP_CORK on Linux). Reduces the number of packets
    # sent, improving throughput for large files. Has no effect without sendfile.
    # Syntax: tcp_nopush on | off;
    # Default: off
    tcp_nopush on;

    # tcp_nodelay — disables Nagle's algorithm for keep-alive connections,
    # sending small packets immediately rather than waiting to batch them.
    # This appears to conflict with tcp_nopush, but they operate at different
    # stages: tcp_nopush handles the file send phase, tcp_nodelay handles the
    # subsequent response phase. Both should be 'on'.
    # Syntax: tcp_nodelay on | off;
    # Default: on
    tcp_nodelay on;

    # -------------------------------------------------------------------------
    # PERFORMANCE — Connection Management
    # -------------------------------------------------------------------------

    # keepalive_timeout — how long an idle client keep-alive connection is
    # held open server-side. 65 seconds is a common CDN/proxy timeout and
    # avoids premature resets. The optional second parameter sets the
    # "Keep-Alive: timeout=N" header, which hints to the client.
    # Setting to 0 disables keep-alive (never do this in production — it
    # eliminates all the performance benefits of HTTP/1.1).
    # Syntax: keepalive_timeout timeout [header_timeout];
    # Default: 75s
    keepalive_timeout 65s;

    # keepalive_requests — maximum requests per keep-alive connection before
    # nginx forces a close. Prevents a single client from monopolizing a
    # connection indefinitely. 1000 (the default since nginx 1.19.10) is
    # appropriate for most workloads.
    # Syntax: keepalive_requests number;
    # Default: 1000
    keepalive_requests 1000;

    # -------------------------------------------------------------------------
    # PERFORMANCE — Timeouts
    # -------------------------------------------------------------------------

    # client_header_timeout — timeout for reading the full request header.
    # If a client sends headers slowly (or not at all), nginx returns 408.
    # 10s is aggressive but eliminates slowloris-style header attacks.
    # Default (60s) is dangerously permissive for public-facing servers.
    # Syntax: client_header_timeout time;
    # Default: 60s
    client_header_timeout 10s;

    # client_body_timeout — timeout between successive reads of the request
    # body. NOT the total upload time. 30s is appropriate for API endpoints
    # with large bodies; tighten to 10s for pure proxies.
    # Syntax: client_body_timeout time;
    # Default: 60s
    client_body_timeout 30s;

    # send_timeout — timeout between successive writes to the client.
    # If the client stops reading (e.g., network drop), nginx closes after
    # this interval. 30s prevents lingering connections from tying up workers.
    # Syntax: send_timeout time;
    # Default: 60s
    send_timeout 30s;

    # reset_timedout_connection — send TCP RST to timed-out clients instead
    # of the normal FIN/ACK sequence. Immediately frees socket memory and
    # prevents FIN_WAIT1 accumulation. Essential on high-traffic servers.
    # Keep-alive connections that close cleanly are NOT affected.
    # Syntax: reset_timedout_connection on | off;
    # Default: off
    reset_timedout_connection on;

    # -------------------------------------------------------------------------
    # PERFORMANCE — Buffer Sizes
    # -------------------------------------------------------------------------

    # client_body_buffer_size — buffer for reading request bodies in memory.
    # If the body exceeds this, nginx spills to a temp file (slow).
    # 16k covers most API JSON payloads. Increase for file upload endpoints.
    # The default is platform-dependent: 8k on 32-bit, 16k on 64-bit.
    # Syntax: client_body_buffer_size size;
    client_body_buffer_size 16k;

    # client_max_body_size — maximum request body size. Requests exceeding
    # this return 413 (Request Entity Too Large). 10m covers most API use
    # cases. For file upload endpoints, override in the specific location block.
    # Set to 0 to disable checking entirely (not recommended).
    # Syntax: client_max_body_size size;
    # Default: 1m
    client_max_body_size 10m;

    # large_client_header_buffers — buffers for large request headers
    # (long URIs, many cookies, JWT tokens). 4 buffers of 8k = 32k total.
    # Requests exceeding one buffer for URI → 414; header field → 400.
    # Syntax: large_client_header_buffers number size;
    # Default: 4 8k
    large_client_header_buffers 4 16k;

    # -------------------------------------------------------------------------
    # PERFORMANCE — Open File Cache
    # -------------------------------------------------------------------------

    # open_file_cache — cache file descriptors, sizes, and modification times
    # for frequently accessed static files. Avoids repeated syscalls (open,
    # stat) for the same file. max=1000 entries, evicting LRU after 30s idle.
    # Syntax: open_file_cache max=N [inactive=time];
    open_file_cache max=1000 inactive=30s;

    # open_file_cache_valid — how often to revalidate cached file metadata.
    # After 30s, nginx re-checks the file on disk. Keeps cache fresh without
    # disabling it entirely.
    open_file_cache_valid 30s;

    # open_file_cache_min_uses — a file must be accessed at least twice
    # within the inactive window to be cached. Prevents one-hit files from
    # polluting the cache.
    open_file_cache_min_uses 2;

    # open_file_cache_errors — also cache negative lookups (file not found).
    # Prevents repeated stat() calls for missing files (e.g., favicon.ico).
    open_file_cache_errors on;

    # -------------------------------------------------------------------------
    # SECURITY — Information Hiding
    # -------------------------------------------------------------------------

    # server_tokens — whether nginx emits its version number in Server headers
    # and error pages. 'off' hides the version, reducing fingerprinting surface.
    # Attackers can still identify nginx from response patterns, but removing
    # the version prevents trivially automated vulnerability scanning.
    # Syntax: server_tokens on | off | build | string;
    # Default: on
    server_tokens off;

    # -------------------------------------------------------------------------
    # HASH TABLE SIZES
    # -------------------------------------------------------------------------

    # types_hash_max_size — size of the MIME types hash table. Increase if
    # nginx logs a warning about hash table collisions. 4096 covers all
    # practical mime.types files.
    # Syntax: types_hash_max_size size;
    # Default: 1024
    types_hash_max_size 4096;

    # -------------------------------------------------------------------------
    # RATE LIMITING ZONES (defined at http level, applied per-server/location)
    # -------------------------------------------------------------------------

    # limit_req_zone — define a shared memory zone for rate limiting.
    # $binary_remote_addr is the client IP in binary form (4 bytes for IPv4,
    # 16 bytes for IPv6) — more compact than $remote_addr (string).
    # zone=req_general:10m — 10MB shared zone named "req_general".
    #   10MB ≈ 160,000 IPv4 states (10MB / 64 bytes per state).
    # rate=20r/s — allow 20 requests per second per IP on average.
    #   This is the "leak rate" of the token bucket, not a hard ceiling.
    #
    # Define multiple zones for different endpoint sensitivity:
    #   - req_general: 20r/s for general API traffic
    #   - req_auth: 5r/s for login/auth endpoints (brute-force mitigation)
    #   - req_static: 100r/s for static assets (CDN miss path)
    # Syntax: limit_req_zone key zone=name:size rate=rate;
    limit_req_zone $binary_remote_addr zone=req_general:10m rate=20r/s;
    limit_req_zone $binary_remote_addr zone=req_auth:10m    rate=5r/s;
    limit_req_zone $binary_remote_addr zone=req_static:10m  rate=100r/s;

    # limit_req_status — HTTP status code returned when rate limit is hit.
    # 429 (Too Many Requests) is semantically correct per RFC 6585.
    # The default (503) misleads clients into thinking the service is down.
    # Syntax: limit_req_status code;
    # Default: 503
    limit_req_status 429;

    # limit_req_log_level — log level for rate limiting events.
    # 'warn' surfaces rate limit events without treating them as errors.
    # Syntax: limit_req_log_level info | notice | warn | error;
    # Default: error
    limit_req_log_level warn;

    # -------------------------------------------------------------------------
    # GZIP COMPRESSION
    # -------------------------------------------------------------------------

    # gzip — enable gzip compression. Off by default; always enable in prod.
    # Syntax: gzip on | off;
    gzip on;

    # gzip_comp_level — compression level 1–9.
    # Level 6 is the optimal trade-off: well past the point of diminishing
    # returns for extra CPU (level 9 is ~2% smaller but ~4x more CPU).
    # Level 1 is fastest but compresses ~40% less than level 6.
    # Use level 4-6 in production. Never use 9.
    # Syntax: gzip_comp_level level;
    # Default: 1
    gzip_comp_level 6;

    # gzip_min_length — minimum response size to trigger compression.
    # Compressing tiny responses (< 256 bytes) costs more CPU than it saves
    # bandwidth. The compressed output can actually be larger than the input
    # for very small payloads. 1024 bytes (1k) is a safe floor.
    # Syntax: gzip_min_length length;
    # Default: 20
    gzip_min_length 1024;

    # gzip_types — MIME types to compress in addition to text/html (which
    # is always compressed when gzip is on). Do not include image types —
    # PNG/JPEG/WebP/GIF are already compressed; re-compressing wastes CPU.
    # Do not include video or audio for the same reason.
    # Syntax: gzip_types mime-type ...;
    # Default: text/html
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/x-javascript
        application/xml
        application/xml+rss
        application/atom+xml
        application/geo+json
        application/manifest+json
        application/rdf+xml
        application/ld+json
        font/otf
        font/ttf
        image/svg+xml;

    # gzip_vary — add "Vary: Accept-Encoding" header to compressed responses.
    # This instructs CDNs and caches to store separate copies for clients
    # that do and do not support gzip. ALWAYS enable this when using gzip.
    # Without it, a CDN might serve a compressed response to a client that
    # sent no Accept-Encoding header.
    # Syntax: gzip_vary on | off;
    # Default: off
    gzip_vary on;

    # gzip_proxied — compress responses for proxied requests based on headers.
    # 'any' compresses all proxied responses regardless of upstream headers.
    # This is correct when nginx terminates TLS from a CDN/load balancer —
    # the upstream response has no Cache-Control that should stop gzip.
    # If your upstream sets meaningful caching headers, use:
    #   gzip_proxied expired no-cache no-store private;
    # Syntax: gzip_proxied off | expired | no-cache | no-store | private |
    #         no_last_modified | no_etag | auth | any;
    # Default: off
    gzip_proxied any;

    # gzip_http_version — minimum HTTP version required to send gzip response.
    # HTTP/1.0 proxies may not handle gzip correctly; 1.1 is the safe floor.
    # Syntax: gzip_http_version 1.0 | 1.1;
    # Default: 1.1
    gzip_http_version 1.1;

    # gzip_disable — disable gzip for matching User-Agent strings.
    # 'msie6' is a built-in alias for the regex matching IE 4-6, which has
    # broken gzip support. In 2025 this is purely defensive; keep it anyway.
    # Syntax: gzip_disable regex;
    gzip_disable "msie6";

    # -------------------------------------------------------------------------
    # LOGGING — JSON Format (defined here, referenced in server blocks)
    # -------------------------------------------------------------------------
    # See Section 13 for full rationale. The log_format must be defined in
    # the http block; access_log directives go in server or location blocks.

    log_format json_combined escape=json
        '{'
            '"time":"$time_iso8601",'
            '"remote_addr":"$remote_addr",'
            '"real_ip":"$http_x_real_ip",'
            '"forwarded_for":"$http_x_forwarded_for",'
            '"method":"$request_method",'
            '"uri":"$uri",'
            '"args":"$args",'
            '"status":$status,'
            '"body_bytes_sent":$body_bytes_sent,'
            '"request_time":$request_time,'
            '"upstream_response_time":"$upstream_response_time",'
            '"upstream_addr":"$upstream_addr",'
            '"upstream_status":"$upstream_status",'
            '"http_referer":"$http_referer",'
            '"http_user_agent":"$http_user_agent",'
            '"http_host":"$http_host",'
            '"server_name":"$server_name",'
            '"ssl_protocol":"$ssl_protocol",'
            '"ssl_cipher":"$ssl_cipher",'
            '"request_id":"$request_id"'
        '}';

    # Route access logs to the JSON format defined above.
    # 'buffer=512k flush=1m' batches writes: up to 512k in memory, flushed
    # every minute. Dramatically reduces write syscalls under high load.
    # gzip compresses log files on the fly (requires nginx built with zlib).
    access_log /var/log/nginx/access.log json_combined buffer=512k flush=1m;

    # -------------------------------------------------------------------------
    # INCLUDE VHOST CONFIGS
    # -------------------------------------------------------------------------

    # Include per-vhost configuration files. Using conf.d/*.conf keeps the
    # main nginx.conf clean and allows vhost management without editing it.
    include /etc/nginx/conf.d/*.conf;

}
```

---

## 5. TLS/HTTPS Server Block

This is the per-vhost configuration file at `/etc/nginx/conf.d/example.conf`. It includes the full HTTPS server block with TLS hardening and security headers.

```nginx
# =============================================================================
# HTTP → HTTPS REDIRECT
# =============================================================================
# Redirect all plain HTTP to HTTPS. Use a permanent redirect (301) so browsers
# cache the redirect and skip the HTTP request entirely after the first visit.
# Do NOT serve any content over HTTP. The only exception is ACME challenges
# for certbot renewals, handled in the location block below.
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;

    # ACME challenge location for Let's Encrypt certificate renewal.
    # This MUST come before the catch-all redirect or renewals will break.
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    # Redirect everything else to HTTPS. $host preserves the original hostname.
    location / {
        return 301 https://$host$request_uri;
    }
}

# =============================================================================
# HTTPS SERVER BLOCK
# =============================================================================
server {
    # listen on port 443 with SSL. 'http2' enables HTTP/2 for this server.
    # HTTP/2 requires TLS; you cannot enable http2 on a plain HTTP listener.
    # '[::]:443' enables IPv6. Both must be present for dual-stack support.
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    server_name example.com www.example.com;

    # -------------------------------------------------------------------------
    # CERTIFICATES
    # -------------------------------------------------------------------------

    # ssl_certificate — full certificate chain (leaf + intermediates) in PEM.
    # Let's Encrypt: fullchain.pem includes the leaf and the CA chain.
    # Never use just the leaf certificate — clients that don't have the
    # intermediate cached will fail TLS verification.
    # Syntax: ssl_certificate file;
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;

    # ssl_certificate_key — private key matching the certificate above.
    # Keep this file at 600 permissions, owned by root.
    # Syntax: ssl_certificate_key file;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # ssl_trusted_certificate — the CA chain used to verify OCSP responses.
    # Required when ssl_stapling_verify is on. Use chain.pem (intermediates
    # only, not the leaf). Let's Encrypt: chain.pem.
    # Syntax: ssl_trusted_certificate file;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;

    # -------------------------------------------------------------------------
    # TLS PROTOCOLS
    # -------------------------------------------------------------------------

    # ssl_protocols — TLS protocol versions to accept.
    # TLSv1.0 and TLSv1.1 are deprecated by RFC 8996 (2021). Do not enable.
    # TLSv1.2 is required for compatibility with older clients (Android 4.x,
    # IE 11, Java 8). TLSv1.3 is preferred for all modern clients.
    # Dropping TLSv1.2 (Modern profile) is only appropriate for services where
    # you control all clients (internal APIs, microservices).
    # Syntax: ssl_protocols [SSLv2] [SSLv3] [TLSv1] [TLSv1.1] [TLSv1.2] [TLSv1.3];
    # Default (nginx >= 1.23.4): TLSv1.2 TLSv1.3
    ssl_protocols TLSv1.2 TLSv1.3;

    # -------------------------------------------------------------------------
    # TLS CIPHERS
    # -------------------------------------------------------------------------

    # ssl_ciphers — the ordered list of TLS 1.2 cipher suites.
    # TLS 1.3 cipher suites are not configurable via this directive; they are
    # always the standardised set (AES-256-GCM-SHA384, etc.).
    #
    # This list is the Mozilla Intermediate profile (October 2024):
    # https://ssl-config.mozilla.org/#server=nginx&config=intermediate
    # It provides broad compatibility (IE11+, Android 4.4.2+) with strong
    # cryptography. It excludes RC4, 3DES, export ciphers, and NULL ciphers.
    #
    # ECDHE provides forward secrecy (PFS): compromise of the server's long-
    # term key does not compromise past session keys.
    # AES-GCM is an authenticated encryption mode (AEAD) — no separate MAC.
    # AES-CBC with SHA256/SHA384 is kept for TLS 1.2 compatibility with
    # clients that lack GCM support (rare but still in the wild).
    #
    # Syntax: ssl_ciphers ciphers;
    # Default: HIGH:!aNULL:!MD5
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305;

    # ssl_prefer_server_ciphers — whether nginx imposes its cipher order over
    # the client's preferred order. With TLS 1.3 this has no effect (the
    # protocol uses negotiation). For TLS 1.2, setting 'off' lets clients
    # choose their preferred cipher from the allowed list — modern clients
    # choose correctly. Mozilla's Intermediate profile recommends 'off'.
    # Only set 'on' if you have a specific compliance reason to enforce order.
    # Syntax: ssl_prefer_server_ciphers on | off;
    # Default: off
    ssl_prefer_server_ciphers off;

    # -------------------------------------------------------------------------
    # DH PARAMETERS
    # -------------------------------------------------------------------------

    # ssl_dhparam — DH parameters file for DHE key exchange.
    # Required if you include DHE- ciphers in ssl_ciphers above.
    # Without this, DHE ciphers silently fall back to the OpenSSL default
    # (1024-bit DH params, considered weak). Generated in Section 1.
    # Syntax: ssl_dhparam file;
    ssl_dhparam /etc/nginx/dhparam.pem;

    # -------------------------------------------------------------------------
    # ELLIPTIC CURVES
    # -------------------------------------------------------------------------

    # ssl_ecdh_curve — the list of elliptic curves for ECDHE key exchange.
    # 'auto' uses OpenSSL's compiled-in defaults (currently X25519, P-256,
    # P-384). Explicitly listing X25519:prime256v1 ensures X25519 is
    # preferred (fastest, no patent concerns) with P-256 as fallback.
    # Syntax: ssl_ecdh_curve curve;
    # Default: auto
    ssl_ecdh_curve X25519:prime256v1:secp384r1;

    # -------------------------------------------------------------------------
    # SESSION RESUMPTION
    # -------------------------------------------------------------------------

    # ssl_session_cache — shared TLS session cache between all worker processes.
    # 'shared:SSL:10m' — 10MB shared zone named "SSL". 1MB ≈ 4000 sessions,
    # so 10MB holds ~40,000 sessions. Shared caches are correct for multi-
    # worker deployments because any worker can resume any client's session.
    # 'builtin' cache is per-worker only — useless with multiple workers.
    # Syntax: ssl_session_cache off | none | [builtin[:size]] [shared:name:size];
    # Default: none
    ssl_session_cache shared:SSL:10m;

    # ssl_session_timeout — how long a TLS session remains resumable.
    # 1d (one day) allows clients to resume without a full handshake all day.
    # Session resumption reduces CPU cost of TLS by ~80% (no asymmetric crypto).
    # Syntax: ssl_session_timeout time;
    # Default: 5m
    ssl_session_timeout 1d;

    # ssl_session_tickets — TLS session tickets allow the server to delegate
    # session state storage to the client (encrypted with a server-side key).
    # DISABLE THIS. Here is why:
    # 1. Ticket keys are rotated only on nginx restart/reload. A key that is
    #    never rotated breaks forward secrecy for sessions resumed with it.
    # 2. In multi-server deployments, ticket keys must be shared and rotated
    #    in sync — this is operationally complex and often not done correctly.
    # 3. The session cache (above) provides equivalent performance gains
    #    without the forward secrecy trade-off.
    # Only re-enable if you implement proper ticket key rotation (STEK).
    # Syntax: ssl_session_tickets on | off;
    # Default: on
    ssl_session_tickets off;

    # ssl_buffer_size — TLS record size. The default 16k sends the largest
    # possible TLS record, optimising throughput for large responses but
    # increasing Time to First Byte (TTFB) for small responses, because nginx
    # must fill a 16k buffer before sending the record.
    # For API servers with small JSON responses, 4k reduces TTFB.
    # For video/download servers, keep the default 16k.
    # Syntax: ssl_buffer_size size;
    # Default: 16k
    ssl_buffer_size 4k;

    # -------------------------------------------------------------------------
    # OCSP STAPLING
    # -------------------------------------------------------------------------

    # ssl_stapling — nginx fetches the OCSP response from the CA and staples
    # it to the TLS handshake. Clients receive proof of certificate validity
    # without making their own OCSP request. Benefits:
    # 1. Faster handshake (no client → CA round trip).
    # 2. Privacy (CA cannot track which clients visit your site).
    # 3. Reliability (OCSP responder outage doesn't block clients).
    # Syntax: ssl_stapling on | off;
    # Default: off
    ssl_stapling on;

    # ssl_stapling_verify — nginx verifies the OCSP response before stapling.
    # Requires ssl_trusted_certificate to be set. Always enable this to
    # prevent serving a tampered or invalid OCSP response.
    # Syntax: ssl_stapling_verify on | off;
    # Default: off
    ssl_stapling_verify on;

    # resolver — DNS resolver for OCSP stapling and any hostname resolution.
    # nginx must resolve the CA's OCSP endpoint. Use your own resolver or a
    # trustworthy public resolver. valid=300s caches DNS results for 5 minutes.
    # ipv6=off avoids issues on hosts where IPv6 DNS is misconfigured.
    resolver 1.1.1.1 8.8.8.8 valid=300s;
    resolver_timeout 5s;

    # -------------------------------------------------------------------------
    # SECURITY HEADERS
    # -------------------------------------------------------------------------
    # IMPORTANT: add_header directives in a server block are NOT inherited by
    # location blocks that define their own add_header directives. If you add
    # add_header in a location block, you must repeat all security headers
    # there too, or use the 'always' parameter and include a snippet.
    # The cleanest solution: define security headers in a snippet and include
    # it in every location block that overrides add_header.

    # HSTS — HTTP Strict Transport Security (RFC 6797).
    # Instructs the browser to only connect to this domain over HTTPS for
    # the duration of 'max-age' (63072000 = 2 years).
    # 'includeSubDomains' — extends the policy to all subdomains. Only add
    # this if you are certain ALL subdomains serve HTTPS correctly.
    # 'preload' — requests inclusion in browser HSTS preload lists (Chrome,
    # Firefox, Safari). Once submitted to https://hstspreload.org, removal
    # takes months. Do NOT add 'preload' unless you are permanently committed.
    # 'always' ensures this header is sent even on error responses.
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;

    # X-Frame-Options — prevents your pages from being embedded in iframes
    # on other domains, mitigating clickjacking attacks.
    # 'SAMEORIGIN' allows your own site to iframe itself (useful for SPAs).
    # 'DENY' is stricter — no framing at all.
    # Note: CSP frame-ancestors supersedes this header in modern browsers,
    # but keep both for IE11 compatibility.
    add_header X-Frame-Options "SAMEORIGIN" always;

    # X-Content-Type-Options — prevents browsers from MIME-sniffing a response
    # away from the declared Content-Type. Without this, a browser might
    # execute a .txt file as JavaScript if the content looks like a script.
    # 'nosniff' is the only valid value.
    add_header X-Content-Type-Options "nosniff" always;

    # Referrer-Policy — controls what is sent in the Referer header.
    # 'strict-origin-when-cross-origin': full URL for same-origin requests,
    # only the origin for cross-origin requests over HTTPS, nothing for HTTP.
    # This is the browser default since ~2021 but set it explicitly for
    # older browsers and to express intentional policy.
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Permissions-Policy (formerly Feature-Policy) — opt out of browser APIs
    # that are never used, reducing attack surface if XSS occurs.
    # This example disables camera, microphone, geolocation, and payment.
    # Tailor to your application's actual API usage.
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=()" always;

    # Content-Security-Policy — the most powerful browser security mechanism.
    # This is a RESTRICTIVE example for a typical web app. Adjust for your
    # application's actual content sources.
    #
    # default-src 'self'     — block everything not explicitly allowed
    # script-src 'self'      — scripts only from own origin (no inline, no eval)
    # style-src 'self' 'unsafe-inline' — inline styles required by most CSS frameworks
    # img-src 'self' data:   — images from own origin + data URIs (base64)
    # font-src 'self'        — fonts from own origin only
    # connect-src 'self'     — XHR/fetch/WebSocket to own origin only
    # media-src 'none'       — no audio/video
    # object-src 'none'      — no plugins (Flash is dead; keep this)
    # frame-ancestors 'none' — equivalent to X-Frame-Options: DENY (modern browsers)
    # base-uri 'self'        — prevents injection of base tags to hijack relative URLs
    # form-action 'self'     — forms may only submit to own origin
    # upgrade-insecure-requests — browser upgrades http:// sub-resources to https://
    #
    # WARNING: Start with report-only mode and collect CSP violation reports before
    # enforcing, especially on existing apps with third-party scripts/styles.
    # Content-Security-Policy-Report-Only: <policy>; report-uri /csp-report;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'; connect-src 'self'; media-src 'none'; object-src 'none'; frame-ancestors 'none'; base-uri 'self'; form-action 'self'; upgrade-insecure-requests;" always;

    # X-Request-ID — expose nginx's $request_id to clients for correlation.
    # Useful for support requests: "include your X-Request-ID in the ticket".
    # $request_id is a 32-hex-char unique ID generated per request (nginx 1.11.0+).
    add_header X-Request-ID $request_id always;

    # -------------------------------------------------------------------------
    # UPSTREAM DEFINITION (see Section 11 for deep dive)
    # -------------------------------------------------------------------------

    # The upstream block is typically defined in the http block (nginx.conf),
    # not inside a server block. It is shown here for completeness.
    # In practice, define upstreams in /etc/nginx/conf.d/upstreams.conf and
    # include that file from the http block.

    # -------------------------------------------------------------------------
    # ROOT / PROXY LOCATION
    # -------------------------------------------------------------------------

    location / {
        # Apply rate limiting using the req_general zone.
        # burst=40 — allow a burst of up to 40 requests above the rate limit
        # before rejecting. These excess requests are queued and processed at
        # the zone's rate (20r/s), effectively adding delay.
        # nodelay — process all burst requests immediately (no added delay).
        # Use 'nodelay' for APIs where latency matters. Omit it (or use
        # 'delay=N') if you want to smooth out bursts by slowing them down.
        # Syntax: limit_req zone=name [burst=number] [nodelay|delay=number];
        limit_req zone=req_general burst=40 nodelay;

        # Proxy to the backend upstream. The upstream name must match the
        # 'upstream' block name.
        proxy_pass http://app_backend;

        # Use HTTP/1.1 for upstream connections. Required for keepalive.
        # HTTP/1.0 (the nginx default) does not support persistent connections.
        proxy_http_version 1.1;

        # Clear the Connection header. HTTP/1.1 may set Connection: close from
        # the client; clearing it prevents nginx from forwarding that to the
        # upstream, which would close the upstream keepalive connection.
        proxy_set_header Connection "";

        # Forward real client information to the backend.
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Request-ID      $request_id;

        # Proxy buffer settings.
        proxy_buffering    on;
        proxy_buffer_size  8k;
        proxy_buffers      8 16k;

        # Proxy timeouts. These are the time allowed for the upstream to
        # respond, not the total request time.
        proxy_connect_timeout 5s;
        proxy_send_timeout    60s;
        proxy_read_timeout    60s;
    }

    # -------------------------------------------------------------------------
    # STATIC FILES — Aggressive Caching
    # -------------------------------------------------------------------------

    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|webp|woff|woff2|ttf|otf|eot)$ {
        # Rate limit static assets at a higher rate than API endpoints.
        limit_req zone=req_static burst=200 nodelay;

        root /var/www/example/public;

        # expires — set the Expires and Cache-Control: max-age headers.
        # '1y' = one year. Appropriate for versioned/fingerprinted assets
        # (e.g., main.a1b2c3d4.js) that change filename when content changes.
        # For non-versioned assets (logo.png), use a shorter TTL (7d or 30d).
        # Syntax: expires [modified] time | epoch | max | off;
        # Default: off
        expires 1y;

        # Cache-Control: public allows CDNs and shared caches to store the
        # response. 'immutable' tells browsers not to revalidate during the
        # max-age window even on a hard reload. Use only with content-addressed
        # (fingerprinted) URLs.
        add_header Cache-Control "public, immutable";

        # Explicitly set security headers here too — they are NOT inherited
        # from the parent server block because we define add_header above.
        add_header X-Content-Type-Options "nosniff" always;

        # serve pre-compressed .gz files if they exist (nginx gzip_static module)
        gzip_static on;

        # Disable logging for static assets to reduce log volume.
        # If you need static asset analytics, use a CDN or keep this on.
        access_log off;
    }

    # -------------------------------------------------------------------------
    # AUTH ENDPOINTS — Strict Rate Limiting
    # -------------------------------------------------------------------------

    location ~ ^/(login|signup|forgot-password|api/auth) {
        # 5r/s with burst of 10 — allow 10 rapid requests then throttle.
        # nodelay omitted — excess requests are delayed, not rejected.
        # This smooths out automated attacks without hard-rejecting scripted
        # single sign-on flows that may legitimately burst.
        limit_req zone=req_auth burst=10;

        proxy_pass http://app_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Request-ID      $request_id;
    }

    # -------------------------------------------------------------------------
    # HEALTH CHECK ENDPOINT — No logging, no rate limiting
    # -------------------------------------------------------------------------

    location = /health {
        access_log off;
        return 200 '{"status":"ok"}';
        add_header Content-Type application/json;
    }

}
```

---

## 6. Worker Tuning Deep Dive

### Calculating the Right worker_connections Value

The formula for total theoretical connections:

```
max_connections = worker_processes * worker_connections
```

Each connection consumes at least one file descriptor. A reverse proxy connection typically consumes **two** (one for the client, one for the upstream). Account for this:

```
worker_connections = (target_max_client_connections / worker_processes) * 2
```

Example: 4 CPU cores, target 10,000 concurrent clients:

```
worker_connections = (10,000 / 4) * 2 = 5,000
```

Round up to the next power of 2 (for alignment): `worker_connections 8192;`

Ensure `worker_rlimit_nofile >= worker_connections` and that the OS ulimit is set accordingly.

### Verifying Current Limits

```bash
# Check nginx worker process open file limit
cat /proc/$(pgrep -o nginx)/limits | grep "open files"

# Check current connections per worker
ss -s

# Check nginx connection statistics
nginx -T 2>/dev/null | grep worker_connections

# Real-time connection state breakdown
ss -ant | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn
```

### epoll vs. Other Event Methods

| Method | OS | Scalability | Notes |
|---|---|---|---|
| `epoll` | Linux | O(1) | Always use on Linux |
| `kqueue` | FreeBSD, macOS | O(1) | Use on FreeBSD |
| `select` | All | O(n) | Nginx fallback; never use if avoidable |
| `poll` | All | O(n) | Slightly better than select; still O(n) |
| `eventport` | Solaris | O(1) | For Solaris only |

On Linux, `epoll` is auto-selected if not specified. Specifying it explicitly (`use epoll;`) is safe and makes the choice visible in the config.

---

## 7. TLS Configuration Rationale

### Protocol Version Decision Tree

```
TLSv1.3 only?
  └─ Use for: internal services, APIs where you control all clients
  └─ Risk: breaks compatibility with Java 8 (< 8u261), old Android

TLSv1.2 + TLSv1.3? ← RECOMMENDED for public-facing services
  └─ Covers: IE11, Android 4.4.2+, Java 8+, all modern clients
  └─ This is Mozilla's "Intermediate" compatibility profile

TLSv1.0/1.1?
  └─ Never. Deprecated RFC 8996 (March 2021). PCI-DSS non-compliant.
```

### Cipher Suite Selection Rationale

The cipher string used above (`ECDHE-ECDSA-AES128-GCM-SHA256:...`) provides:

- **ECDHE** or **DHE** key exchange → Forward Secrecy (PFS)
- **AES-GCM** → Authenticated Encryption (AEAD), not vulnerable to BEAST/POODLE
- **CHACHA20-POLY1305** → Better performance than AES-GCM on CPUs without AES-NI (mobile)
- **No RC4**, **no 3DES**, **no export ciphers**, **no NULL ciphers**

### Session Ticket Key Rotation (if you must enable tickets)

If operationally you need session tickets (e.g., very large farm where shared cache is not feasible), implement key rotation:

```bash
# Generate a new 80-byte ticket key (48 bytes + 16 byte HMAC + 16 byte AES)
openssl rand 80 > /etc/nginx/ticket-$(date +%Y%m%d).key

# nginx does not support runtime key rotation via signals — requires reload
# Automate with a systemd timer or cron:
# 0 */6 * * * root /usr/local/sbin/rotate-nginx-ticket-key.sh
```

### HSTS Preload Commitment

Before adding `preload` to the HSTS header, verify:

1. All subdomains serve valid HTTPS (check with: `curl -I https://subdomain.example.com`)
2. Redirect from HTTP to HTTPS works at the root and all subdomains
3. Your TLS certificate covers all subdomains (wildcard or multi-SAN)
4. You are prepared for a 2-year commitment

Submit at: https://hstspreload.org

---

## 8. Security Headers Reference

### add_header Inheritance Warning

This is the most common nginx misconfiguration that silently breaks security:

```nginx
# server block
add_header X-Frame-Options "SAMEORIGIN" always;

location /api/ {
    # THIS LOCATION HAS ITS OWN add_header
    # The X-Frame-Options header from the server block is NO LONGER SENT
    # for requests matching /api/ — it is silently dropped.
    add_header Content-Type application/json;
    proxy_pass http://api_backend;
}
```

**Solutions:**

**Option A — Repeat all headers in every location block** (verbose but explicit):

```nginx
location /api/ {
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    # ... all other security headers ...
    add_header Content-Type application/json;
}
```

**Option B — Use a snippet** (cleaner):

```nginx
# /etc/nginx/snippets/security-headers.conf
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=()" always;

# In each location block:
location /api/ {
    include snippets/security-headers.conf;
    add_header Content-Type application/json;
}
```

**Option C — nginx >= 1.29.3** — use `add_header_inherit merge;` to merge parent headers into child context.

### Header Validation

```bash
# Check all security headers on your site
curl -sI https://example.com | grep -E "^(Strict|X-|Referrer|Permissions|Content-Security)"

# Or use securityheaders.com for a graded report
```

---

## 9. Gzip Compression

### Why Level 6

Compression levels 1–9 in zlib (which nginx uses) follow a logarithmic return curve:

| Level | Compression Ratio | Relative CPU |
|---|---|---|
| 1 | ~60% of level 9 | 1x |
| 4 | ~87% of level 9 | ~2x |
| 6 | ~95% of level 9 | ~3.5x |
| 9 | 100% (reference) | ~10x |

Level 6 recovers 95% of maximum compression at 35% of maximum CPU cost. Levels 7–9 are wasteful in production.

### What NOT to Compress

- **Images**: JPEG, PNG, WebP, GIF, AVIF are already compressed. Re-compressing adds CPU with no benefit.
- **Audio/Video**: Same reasoning.
- **Pre-compressed files**: .zip, .gz, .br, .zst files expand when gzip-compressed.
- **Very small responses** (< 1 KB): Overhead exceeds benefit; gzip_min_length handles this.

### Pre-compressed Files (gzip_static)

For static assets, pre-compress at build time and serve the `.gz` file directly:

```bash
# Build step: compress each asset
gzip -k -9 /var/www/public/assets/main.js
# Creates main.js.gz alongside main.js
```

```nginx
location ~* \.(js|css|svg)$ {
    gzip_static on;  # Serve .gz file if it exists and client accepts gzip
    expires 1y;
}
```

For Brotli (better compression than gzip, supported by all modern browsers):

```nginx
# Requires ngx_brotli module
brotli on;
brotli_comp_level 6;
brotli_types text/plain text/css application/json application/javascript;
brotli_static on;
```

---

## 10. Rate Limiting

### Token Bucket Algorithm

nginx's rate limiting is a token bucket (via the leaky bucket variant). Understanding this prevents misconfiguration:

- `rate=20r/s` — the bucket drains at 20 requests per second
- `burst=40` — the bucket holds up to 40 tokens (excess requests)
- Without `nodelay`: excess requests are queued and processed as the bucket refills
- With `nodelay`: all burst requests are processed immediately; the bucket still enforces the rate for subsequent requests

### When to Use nodelay vs. delay

```nginx
# Use nodelay for APIs: latency matters, don't add artificial delay to bursts
limit_req zone=req_general burst=40 nodelay;

# Use delay for login endpoints: slow down automated attacks
limit_req zone=req_auth burst=10;
# The first 10 requests after the rate limit is hit are delayed (queued)
# Further requests beyond burst return 429

# Use delay=N to allow some burst immediately then delay the rest
limit_req zone=req_general burst=40 delay=20;
# First 20 excess requests: processed immediately
# Requests 21-40: delayed to match the rate
# Requests 41+: rejected with 429
```

### Multiple Rate Limiting Zones

You can apply multiple zones to a single location:

```nginx
location /api/ {
    # Per-IP limit AND a global limit (using $server_name as key)
    limit_req zone=req_general burst=40 nodelay;
    limit_req zone=req_global  burst=1000 nodelay;
}
```

```nginx
# In http block:
limit_req_zone $binary_remote_addr zone=req_general:10m rate=20r/s;
limit_req_zone $server_name        zone=req_global:1m  rate=500r/s;
```

### Testing Rate Limits

```bash
# Send 50 requests as fast as possible and count the 429s
for i in $(seq 1 50); do
  curl -s -o /dev/null -w "%{http_code}\n" https://example.com/api/resource
done | sort | uniq -c

# With Apache Bench:
ab -n 100 -c 10 https://example.com/api/resource
```

---

## 11. Upstream Keepalive

### Full Upstream Configuration

Define this in the http block (or a separate `conf.d/upstreams.conf`):

```nginx
# /etc/nginx/conf.d/upstreams.conf

upstream app_backend {
    # least_conn balances by active connections, not round-robin.
    # Correct for backends with variable response times (API servers).
    # For uniform, fast backends (cache servers), round-robin is fine.
    least_conn;

    # List backend servers. Add weight for asymmetric hardware.
    server 10.0.1.10:8080 max_fails=3 fail_timeout=30s;
    server 10.0.1.11:8080 max_fails=3 fail_timeout=30s;
    server 10.0.1.12:8080 max_fails=3 fail_timeout=30s weight=2;  # Faster host

    # Backup server — used only when all primaries are unavailable.
    server 10.0.1.99:8080 backup;

    # keepalive — number of idle keepalive connections to upstream servers
    # to keep in each worker process's connection pool.
    # This is NOT a limit on total connections — it is a cache of idle ones.
    # A worker opens more connections as needed; it keeps at most 'keepalive'
    # connections idle after requests complete.
    # Rule of thumb: keepalive = (upstream_servers * expected_concurrency_per_worker)
    # For 3 upstreams, 4 workers, expecting 10 concurrent requests per worker:
    #   keepalive = ceil(3 * 10 / 4) = 8, round up to 16 for headroom.
    # Syntax: keepalive connections;
    keepalive 32;

    # keepalive_requests — maximum requests per keepalive upstream connection
    # before it is closed and replaced. Prevents issues with leaking resources
    # in long-lived connections and ensures backend load-balancing remains even.
    # Syntax: keepalive_requests number;
    # Default: 1000
    keepalive_requests 1000;

    # keepalive_timeout — idle keepalive connection timeout.
    # Must be less than the backend's own keepalive timeout to prevent nginx
    # from sending a request on a connection the backend has just closed.
    # If your backend closes idle connections after 60s, set this to 55s.
    # Syntax: keepalive_timeout timeout;
    # Default: 60s
    keepalive_timeout 55s;
}
```

### Required Proxy Directives for Upstream Keepalive

Both of these are required in the location block for keepalive to work:

```nginx
# Use HTTP/1.1 — HTTP/1.0 (nginx's default to upstreams) does not support
# persistent connections. Without this, nginx closes the connection after
# every request, negating all keepalive benefits.
proxy_http_version 1.1;

# Clear the Connection header. An HTTP/1.1 client may send "Connection: close"
# or "Connection: keep-alive". Forwarding these to the upstream can override
# the keepalive behavior nginx negotiated. Clearing it allows the upstream
# to use its own keepalive default.
proxy_set_header Connection "";
```

### Verifying Upstream Keepalive

```bash
# Enable nginx stub_status module to see connection counts
location = /nginx_status {
    stub_status;
    allow 127.0.0.1;
    deny all;
}

# Then:
curl http://127.0.0.1/nginx_status
# Output:
# Active connections: 142
# server accepts handled requests
#  1234567 1234567 9876543
# Reading: 5 Writing: 23 Waiting: 114
#
# "Waiting" = keep-alive connections waiting for the next request.
# A high Waiting count relative to Active indicates keepalive is working.
```

---

## 12. Static File Caching

### Cache-Control Strategy

```nginx
# Versioned/fingerprinted assets (e.g., main.a1b2c3.js, style.d4e5f6.css)
# Set once, cached forever. Safe because URL changes when content changes.
location ~* \.(js|css|woff2?)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}

# Images without content-addressing: shorter TTL
location ~* \.(png|jpg|jpeg|gif|webp|ico|svg)$ {
    expires 30d;
    add_header Cache-Control "public";
}

# HTML pages: short TTL or no-cache so updates propagate quickly.
# The browser revalidates each time but gets a 304 if unchanged (cheap).
location ~* \.html$ {
    expires 1h;
    add_header Cache-Control "public, must-revalidate";
}

# API responses: typically not cached at nginx (handled by application logic)
# But you can add no-store for sensitive endpoints:
location /api/user/ {
    add_header Cache-Control "private, no-store";
    proxy_pass http://app_backend;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
}
```

### ETag and Last-Modified

nginx sends `ETag` and `Last-Modified` headers for static files by default. Do not disable them — they enable conditional requests (304 Not Modified), which save bandwidth even when caches have expired.

```nginx
# These are on by default; shown for reference
etag on;          # default: on (since nginx 1.3.3)
# Last-Modified is always sent for static files; no directive to disable
```

---

## 13. JSON Log Format

### Why JSON Logs

Plaintext access logs require fragile regex parsing in log aggregation systems (ELK, Loki, Splunk). JSON logs parse reliably with zero configuration in all modern log shippers.

### Complete JSON Log Format

```nginx
# Defined in http block (see Section 4 above)
log_format json_combined escape=json
    '{'
        # Timestamp in ISO 8601 format — parseable by all log systems
        '"time":"$time_iso8601",'

        # Client IP addresses
        '"remote_addr":"$remote_addr",'
        '"real_ip":"$http_x_real_ip",'            # Set by upstream load balancer
        '"forwarded_for":"$http_x_forwarded_for",' # Full proxy chain

        # Request details
        '"method":"$request_method",'
        '"uri":"$uri",'                    # Decoded URI (no query string)
        '"args":"$args",'                  # Query string only
        '"status":$status,'               # Integer, not quoted (for numeric queries)

        # Bytes (integer, not quoted)
        '"body_bytes_sent":$body_bytes_sent,'

        # Timing (float seconds, not quoted)
        '"request_time":$request_time,'
        '"upstream_response_time":"$upstream_response_time",' # Quoted: may be "-"

        # Upstream details (may be "-" if request not proxied)
        '"upstream_addr":"$upstream_addr",'
        '"upstream_status":"$upstream_status",'

        # Headers
        '"http_referer":"$http_referer",'
        '"http_user_agent":"$http_user_agent",'
        '"http_host":"$http_host",'

        # Server context
        '"server_name":"$server_name",'

        # TLS details (empty string on non-TLS connections)
        '"ssl_protocol":"$ssl_protocol",'
        '"ssl_cipher":"$ssl_cipher",'

        # Unique request ID for distributed tracing correlation
        '"request_id":"$request_id"'
    '}';
```

### escape=json

The `escape=json` parameter (nginx 1.11.8+) JSON-encodes special characters in variable values. Without it, a `User-Agent` string containing a quote character would break the JSON. Always use `escape=json` with JSON log formats.

### Log Rotation

```bash
# /etc/logrotate.d/nginx
/var/log/nginx/*.log {
    daily
    missingok
    rotate 30
    compress
    delaycompress
    notifempty
    sharedscripts
    postrotate
        # Send USR1 to nginx to reopen log files after rotation
        if [ -f /run/nginx.pid ]; then
            kill -USR1 $(cat /run/nginx.pid)
        fi
    endscript
}
```

---

## 14. Post-Config Checklist

Run every item before going live and after every significant config change.

### Validation

```bash
# Test configuration syntax without reloading
nginx -t

# Verbose test — shows which files are included
nginx -T

# Check the resolved config (useful for debugging include chains)
nginx -T 2>/dev/null | grep -A1 "ssl_protocols"
```

### TLS Verification

```bash
# Verify TLS handshake and certificate chain
openssl s_client -connect example.com:443 -servername example.com < /dev/null 2>&1 | \
  grep -E "(Protocol|Cipher|Verify|subject|issuer)"

# Check TLS 1.0/1.1 are disabled (should return error)
openssl s_client -tls1 -connect example.com:443 < /dev/null 2>&1 | grep "handshake failure"
openssl s_client -tls1_1 -connect example.com:443 < /dev/null 2>&1 | grep "handshake failure"

# Check TLS 1.2 works
openssl s_client -tls1_2 -connect example.com:443 < /dev/null 2>&1 | grep "Protocol"

# Check TLS 1.3 works
openssl s_client -tls1_3 -connect example.com:443 < /dev/null 2>&1 | grep "Protocol"

# Verify OCSP stapling is working
openssl s_client -connect example.com:443 -status < /dev/null 2>&1 | \
  grep -A5 "OCSP Response"
# Should show: OCSP Response Status: successful (0x0)

# Check certificate expiry
openssl s_client -connect example.com:443 < /dev/null 2>&1 | \
  openssl x509 -noout -dates
```

### Security Header Verification

```bash
# Dump all response headers
curl -sI https://example.com

# Check specific security headers are present
curl -sI https://example.com | grep -iE \
  "(strict-transport|x-frame|x-content-type|referrer-policy|content-security|permissions-policy)"

# Verify HSTS max-age is at least 1 year (31536000 seconds)
curl -sI https://example.com | grep -i strict-transport | grep -oP "max-age=\K[0-9]+"

# External check: https://securityheaders.com
```

### Rate Limit Verification

```bash
# Trigger rate limiting and verify 429 response (not 503)
for i in $(seq 1 100); do
  curl -s -o /dev/null -w "%{http_code}\n" https://example.com/
done | sort | uniq -c

# Verify limit_req_status is 429 (not default 503)
nginx -T | grep limit_req_status
```

### Gzip Verification

```bash
# Verify gzip is active for compressible content
curl -sI -H "Accept-Encoding: gzip" https://example.com/ | grep -i content-encoding
# Expected: content-encoding: gzip

# Verify Vary header is present (required for CDN correctness)
curl -sI -H "Accept-Encoding: gzip" https://example.com/ | grep -i vary
# Expected: vary: Accept-Encoding

# Verify images are NOT compressed
curl -sI -H "Accept-Encoding: gzip" https://example.com/logo.png | grep -i content-encoding
# Expected: no content-encoding header
```

### Performance Baseline

```bash
# Measure TTFB before and after config changes
curl -s -o /dev/null -w "TTFB: %{time_starttransfer}s  Total: %{time_total}s\n" \
  https://example.com/

# TLS handshake time
curl -s -o /dev/null -w "TLS: %{time_appconnect}s  Connect: %{time_connect}s\n" \
  https://example.com/

# With ab (Apache Bench): 1000 requests, 50 concurrent
ab -n 1000 -c 50 -H "Accept-Encoding: gzip" https://example.com/

# With hey (modern replacement for ab)
hey -n 1000 -c 50 https://example.com/
```

### Log Verification

```bash
# Verify JSON logs parse correctly
tail -n 5 /var/log/nginx/access.log | python3 -m json.tool

# Check for any JSON parse errors (malformed log lines)
tail -n 1000 /var/log/nginx/access.log | python3 -c "
import sys, json
for i, line in enumerate(sys.stdin):
    try: json.loads(line)
    except json.JSONDecodeError as e: print(f'Line {i}: {e}')
print('Done')
"

# Verify error log is not filling with unexpected errors
tail -f /var/log/nginx/error.log
```

### Reload vs. Restart

```nginx
# Graceful reload — zero downtime; existing connections finish normally
systemctl reload nginx
# or: nginx -s reload

# Full restart — only if reload fails (e.g., changing listen ports)
systemctl restart nginx
```

### Ongoing: Certificate Renewal Monitoring

```bash
# Verify certbot auto-renewal timer is active
systemctl status certbot.timer

# Dry-run renewal to verify it works
certbot renew --dry-run

# Set up expiry alerting (nagios/icinga/prometheus blackbox_exporter)
# Or use: https://certificatemonitor.org
```

---

## References

- nginx Core Module: https://nginx.org/en/docs/ngx_core_module.html
- nginx HTTP Core Module: https://nginx.org/en/docs/http/ngx_http_core_module.html
- nginx SSL Module: https://nginx.org/en/docs/http/ngx_http_ssl_module.html
- nginx Upstream Module: https://nginx.org/en/docs/http/ngx_http_upstream_module.html
- nginx Limit Req Module: https://nginx.org/en/docs/http/ngx_http_limit_req_module.html
- nginx Gzip Module: https://nginx.org/en/docs/http/ngx_http_gzip_module.html
- nginx Headers Module: https://nginx.org/en/docs/http/ngx_http_headers_module.html
- Mozilla SSL Config Generator: https://ssl-config.mozilla.org/
- HSTS Preload List: https://hstspreload.org/
- RFC 8996 — Deprecating TLS 1.0 and 1.1: https://www.rfc-editor.org/rfc/rfc8996
- RFC 6797 — HTTP Strict Transport Security: https://www.rfc-editor.org/rfc/rfc6797
- RFC 6585 — Additional HTTP Status Codes (429): https://www.rfc-editor.org/rfc/rfc6585
