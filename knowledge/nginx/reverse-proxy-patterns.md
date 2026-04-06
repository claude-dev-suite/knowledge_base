# NGINX Reverse Proxy Patterns — Production Reference

> Written from institutional knowledge accumulated running NGINX in production across dozens of services. This is opinionated. Where there are multiple ways to do something, the recommended path is shown first with the reasoning explained. Defaults are often wrong for production; they are called out explicitly.

---

## Table of Contents

1. [Foundational Concepts](#1-foundational-concepts)
2. [Single Application Reverse Proxy](#2-single-application-reverse-proxy)
3. [Multi-App by Subdomain](#3-multi-app-by-subdomain)
4. [Multi-App by Path](#4-multi-app-by-path)
5. [WebSocket Proxying](#5-websocket-proxying)
6. [Static Files with Proxy Fallback](#6-static-files-with-proxy-fallback)
7. [Load Balanced Upstreams](#7-load-balanced-upstreams)
8. [SSL Termination](#8-ssl-termination)
9. [Caching Proxy](#9-caching-proxy)
10. [Rate Limiting](#10-rate-limiting)
11. [IP Allowlisting for Admin Paths](#11-ip-allowlisting-for-admin-paths)
12. [Real IP Passthrough](#12-real-ip-passthrough)
13. [Hardening and Operational Details](#13-hardening-and-operational-details)

---

## 1. Foundational Concepts

### How `proxy_pass` Resolves URIs

This is the single most-misunderstood directive in NGINX. The behaviour changes depending on whether a URI is appended to the upstream address.

**Rule:** If `proxy_pass` includes a URI (even just a trailing `/`), NGINX strips the `location` prefix from the request URI before forwarding. If it does NOT include a URI, the raw normalized request URI is forwarded unchanged.

```nginx
# Case A: No URI on proxy_pass
# Request:  GET /api/v1/users
# Forwarded: GET /api/v1/users
location /api/ {
    proxy_pass http://backend;
}

# Case B: Trailing slash on proxy_pass (URI present)
# Request:  GET /api/v1/users
# Forwarded: GET /v1/users   <-- /api stripped
location /api/ {
    proxy_pass http://backend/;
}

# Case C: Path rewrite on proxy_pass
# Request:  GET /api/v1/users
# Forwarded: GET /app/v1/users
location /api/ {
    proxy_pass http://backend/app/;
}
```

The trailing-slash gotcha destroys a surprising number of production deployments. The rule: **use a trailing slash on `proxy_pass` only when you explicitly intend to rewrite the path prefix**. When in doubt, omit the URI.

Variables in `proxy_pass` are treated as having no URI:

```nginx
# Variables force the "no URI" behaviour — safe
location /api/ {
    proxy_pass http://$backend_host;
}
```

### HTTP Version

Always use HTTP/1.1 for upstream connections. HTTP/1.0 is the historical default on some older builds and causes `Connection: close` semantics that kill keepalive.

```nginx
proxy_http_version 1.1;
```

When keepalives are enabled on the upstream (via `keepalive` in the `upstream` block), you must also clear the `Connection` header, since HTTP/1.0 proxies populate it with hop-by-hop header names. NGINX's default `proxy_set_header Connection close` explicitly tears down keepalives.

```nginx
proxy_set_header Connection "";   # clear hop-by-hop — enables keepalive reuse
```

### The Default Headers NGINX Sends Upstream

NGINX only sends two headers by default:

```
Host: $proxy_host        # the upstream hostname:port, NOT the original Host
Connection: close
```

Both defaults are wrong for nearly every production use case. The `Host` default breaks virtual hosting on the upstream. The `Connection: close` default kills connection reuse. Override both explicitly in every proxy config.

---

## 2. Single Application Reverse Proxy

The canonical production configuration for a single backend application. This is the base every other pattern builds on.

```nginx
# /etc/nginx/conf.d/myapp.conf

upstream myapp_backend {
    server 127.0.0.1:3000;

    # Keep idle connections open — eliminates TCP handshake overhead
    # per request. Size this at (expected RPS / average upstream latency).
    # For 500 RPS with 50 ms average latency: ~25 connections minimum.
    keepalive 32;
    keepalive_requests 1000;
    keepalive_timeout 60s;
}

server {
    listen 80;
    listen [::]:80;
    server_name myapp.example.com;

    # Redirect all HTTP to HTTPS — see SSL section for the HTTPS block
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name myapp.example.com;

    ssl_certificate     /etc/letsencrypt/live/myapp.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/myapp.example.com/privkey.pem;

    # Access log with timing and upstream address for debugging
    access_log /var/log/nginx/myapp.access.log combined_plus_upstream;

    # Client body size — set to match your app's max upload. Default 1m is too low.
    client_max_body_size 50m;

    # Read timeout for long-running requests (file uploads, reports)
    # Default 60s kills legitimate slow requests.
    proxy_read_timeout 120s;

    location / {
        proxy_pass http://myapp_backend;

        # --- Protocol ---
        proxy_http_version 1.1;
        proxy_set_header Connection "";   # enable keepalive

        # --- Identity headers: what the backend needs to know about the client ---
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP        $remote_addr;
        proxy_set_header X-Forwarded-For  $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host  $host;
        proxy_set_header X-Forwarded-Port  $server_port;

        # --- Buffering ---
        # ON by default. With buffering, NGINX reads the full upstream response
        # before sending it to the client. This protects upstream from slow clients.
        # Turn OFF only for streaming responses (SSE, chunked) or if upstream
        # processes responses slowly (e.g., media transcoding).
        proxy_buffering on;
        proxy_buffer_size       16k;   # first-chunk buffer, fits most HTTP headers
        proxy_buffers           8 16k; # total: 8 × 16k = 128k per connection
        proxy_busy_buffers_size 32k;   # max allowed to flush while buffering

        # Spill to disk only if response is enormous. Default 1024m is absurd.
        # For most APIs set to 0 to keep everything in memory buffers.
        proxy_max_temp_file_size 0;

        # --- Timeouts ---
        proxy_connect_timeout 10s;   # TCP connect; fail fast on dead upstream
        proxy_send_timeout    60s;   # time between writes to upstream
        proxy_read_timeout    120s;  # time between reads from upstream

        # --- Error handling ---
        # Retry on network errors and upstream server errors only.
        # Do NOT add non-idempotent methods or you risk double-POSTs.
        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_next_upstream_tries   2;
        proxy_next_upstream_timeout 5s;

        # Pass upstream errors to error_page instead of raw proxy errors.
        # Requires custom error pages to be configured.
        proxy_intercept_errors on;

        # --- Response headers ---
        # Strip internal headers upstream should not expose
        proxy_hide_header X-Powered-By;
        proxy_hide_header Server;

        # Rewrite Location headers from upstream if they contain the internal address
        proxy_redirect http://127.0.0.1:3000/ https://myapp.example.com/;
    }

    # Custom error pages — these serve your own HTML, not NGINX defaults
    error_page 502 503 504 /errors/upstream_error.html;
    location = /errors/upstream_error.html {
        root /var/www/myapp;
        internal;
    }
}
```

### Log Format with Upstream Info

Define this in `nginx.conf` `http` block to get upstream address and timing in access logs:

```nginx
log_format combined_plus_upstream
    '$remote_addr - $remote_user [$time_local] '
    '"$request" $status $body_bytes_sent '
    '"$http_referer" "$http_user_agent" '
    'rt=$request_time '
    'urt="$upstream_response_time" '
    'uaddr="$upstream_addr" '
    'ust="$upstream_status"';
```

`$upstream_response_time` is your application latency. `$request_time` includes client send/receive. The difference is network overhead on the client side.

---

## 3. Multi-App by Subdomain

Different apps on different subdomains on the same NGINX instance. Use `server_name` with patterns.

```nginx
# Shared upstream definitions — can be in a separate upstreams.conf include

upstream app_frontend {
    server 127.0.0.1:3000;
    keepalive 16;
}

upstream app_api {
    server 127.0.0.1:4000;
    keepalive 32;
}

upstream app_admin {
    server 127.0.0.1:5000;
    keepalive 8;
}

upstream app_docs {
    server 127.0.0.1:6000;
    keepalive 4;
}

# Shared proxy header snippet — include in each server block
# /etc/nginx/snippets/proxy-headers.conf
#
#   proxy_http_version 1.1;
#   proxy_set_header Connection "";
#   proxy_set_header Host              $host;
#   proxy_set_header X-Real-IP        $remote_addr;
#   proxy_set_header X-Forwarded-For  $proxy_add_x_forwarded_for;
#   proxy_set_header X-Forwarded-Proto $scheme;

# --- Wildcard catch-all for all *.example.com subdomains ---
# Useful when you have a subdomain-per-tenant model
server {
    listen 443 ssl;
    http2 on;
    server_name ~^(?P<subdomain>.+)\.example\.com$;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # Route based on captured subdomain variable
    location / {
        proxy_pass http://127.0.0.1:3000;  # tenant router app handles dispatch
        include /etc/nginx/snippets/proxy-headers.conf;

        proxy_set_header X-Tenant $subdomain;  # pass subdomain to backend
    }
}

# --- Explicit named virtual hosts (preferred when list is known) ---
server {
    listen 443 ssl;
    http2 on;
    server_name www.example.com example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    location / {
        proxy_pass http://app_frontend;
        include /etc/nginx/snippets/proxy-headers.conf;
        proxy_read_timeout 30s;
    }
}

server {
    listen 443 ssl;
    http2 on;
    server_name api.example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # API has smaller body size, shorter timeouts — tune per subdomain
    client_max_body_size 10m;

    location / {
        proxy_pass http://app_api;
        include /etc/nginx/snippets/proxy-headers.conf;
        proxy_connect_timeout 5s;
        proxy_read_timeout    30s;
    }
}

server {
    listen 443 ssl;
    http2 on;
    server_name admin.example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # Admin panel — restrict by IP at the server level
    # This applies to ALL locations in this server block
    allow 10.0.0.0/8;
    allow 192.168.1.0/24;
    allow 203.0.113.50;   # office static IP
    deny all;

    location / {
        proxy_pass http://app_admin;
        include /etc/nginx/snippets/proxy-headers.conf;
    }
}

server {
    listen 443 ssl;
    http2 on;
    server_name docs.example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    location / {
        proxy_pass http://app_docs;
        include /etc/nginx/snippets/proxy-headers.conf;
    }
}

# --- Wildcard TLS with a single wildcard certificate ---
# Use when you have a *.example.com wildcard cert from your CA.
# server_name can use a leading dot or explicit wildcard:
#
# server_name .example.com;   # matches example.com and *.example.com
# server_name *.example.com;  # matches only subdomains
```

### Snippet: `proxy-headers.conf`

Centralise the repetitive header block. Every server block includes this file to ensure consistency:

```nginx
# /etc/nginx/snippets/proxy-headers.conf
proxy_http_version 1.1;
proxy_set_header Connection "";
proxy_set_header Host              $host;
proxy_set_header X-Real-IP        $remote_addr;
proxy_set_header X-Forwarded-For  $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host  $host;
proxy_set_header X-Forwarded-Port  $server_port;
```

---

## 4. Multi-App by Path

Multiple applications served from path prefixes on a single domain. This is the hardest pattern to get right due to the trailing-slash behaviour documented in §1.

```nginx
upstream svc_auth   { server 127.0.0.1:4001; keepalive 8; }
upstream svc_users  { server 127.0.0.1:4002; keepalive 16; }
upstream svc_orders { server 127.0.0.1:4003; keepalive 16; }
upstream svc_static { server 127.0.0.1:4004; keepalive 4; }

server {
    listen 443 ssl;
    http2 on;
    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # -----------------------------------------------------------------------
    # Path-prefix routing — NO trailing slash on proxy_pass
    # The full path /auth/... is forwarded as-is to the backend.
    # The backend must be mounted at / and handle /auth/ itself, OR
    # you strip the prefix explicitly with a rewrite.
    # -----------------------------------------------------------------------

    # Option A: Forward with prefix intact (backend owns its own prefix)
    # Backend sees:  GET /auth/login
    location /auth/ {
        proxy_pass http://svc_auth;          # no trailing slash
        include /etc/nginx/snippets/proxy-headers.conf;
        proxy_read_timeout 30s;
    }

    # Option B: Strip prefix before forwarding (backend mounted at /)
    # Request:  GET /users/profile
    # Backend sees:  GET /profile
    location /users/ {
        rewrite ^/users/(.*) /$1 break;     # strip /users prefix
        proxy_pass http://svc_users;
        include /etc/nginx/snippets/proxy-headers.conf;
    }

    # Option C: proxy_pass with trailing slash (the "built-in" strip)
    # Request:  GET /orders/123
    # Backend sees:  GET /123   <-- /orders stripped
    # WARNING: This only works cleanly when location and proxy_pass
    # both end with /. Any mismatch causes subtle double-slash bugs.
    location /orders/ {
        proxy_pass http://svc_orders/;       # trailing slash strips /orders
        include /etc/nginx/snippets/proxy-headers.conf;
    }

    # -----------------------------------------------------------------------
    # Recommendation: Use Option A (no trailing slash) unless you have a
    # specific reason to strip the prefix. It is the least surprising.
    # Document which option you chose and why in a comment on every location.
    # -----------------------------------------------------------------------

    # Exact match takes priority over prefix matches
    location = / {
        proxy_pass http://svc_static;
        include /etc/nginx/snippets/proxy-headers.conf;
    }

    # Longest prefix match wins over shorter — NGINX does not use order
    location / {
        proxy_pass http://svc_static;
        include /etc/nginx/snippets/proxy-headers.conf;
    }
}
```

### Location Block Priority Rules (Critical)

NGINX matches locations in this order — order in the config file is irrelevant for most types:

1. `=` exact match — highest priority, stops search immediately
2. `^~` prefix match — if matched, regex search is skipped
3. `~` / `~*` regex match — evaluated in config file order; first match wins
4. Prefix match — longest prefix wins regardless of file order

```nginx
# This block structure demonstrates priority:
location = /health     { return 200 "ok"; }          # 1. exact
location ^~ /static/   { root /var/www; }             # 2. prefix, no regex
location ~ \.php$       { ... }                        # 3. regex (case sensitive)
location ~* \.(jpg|png)$ { ... }                       # 3. regex (case insensitive)
location /api/          { proxy_pass http://api; }     # 4. prefix
location /              { proxy_pass http://frontend; } # 4. catchall
```

### The Trailing-Slash Gotcha in Detail

```nginx
# Bug: double slash when location has no trailing slash
location /api {
    proxy_pass http://backend/;   # request /api/v1 becomes //v1 upstream
}

# Bug: missing prefix strip when both lack trailing slash
location /api {
    proxy_pass http://backend;    # request /api/v1 becomes /api/v1 upstream (correct)
}

# Correct strip: both have trailing slash
location /api/ {
    proxy_pass http://backend/;   # request /api/v1 becomes /v1 upstream (correct)
}

# Correct no-strip: neither has trailing slash on proxy_pass
location /api/ {
    proxy_pass http://backend;    # request /api/v1 becomes /api/v1 upstream (correct)
}
```

---

## 5. WebSocket Proxying

WebSocket upgrades use HTTP/1.1 `Upgrade` and `Connection` headers. These are hop-by-hop headers — NGINX strips them by default unless explicitly configured to pass them through.

The canonical solution uses a `map` block to handle both WebSocket and regular HTTP through the same location.

```nginx
# In the http block (nginx.conf or a conf.d include)
map $http_upgrade $connection_upgrade {
    default  upgrade;   # Upgrade header present → pass upgrade
    ''       close;     # No Upgrade header → normal close
}

upstream ws_backend {
    server 127.0.0.1:8080;
    # WebSocket connections are long-lived — keepalive is less relevant here
    # but still useful for the initial HTTP handshake
    keepalive 8;
}

server {
    listen 443 ssl;
    http2 on;
    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    location /ws/ {
        proxy_pass http://ws_backend;

        # WebSocket requires HTTP/1.1 — HTTP/2 WebSocket is different (RFC 8441)
        proxy_http_version 1.1;

        # Pass the Upgrade header from the client
        proxy_set_header Upgrade    $http_upgrade;

        # Use the map variable: "upgrade" or "close" depending on client
        proxy_set_header Connection $connection_upgrade;

        # Standard identity headers
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP        $remote_addr;
        proxy_set_header X-Forwarded-For  $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # --- CRITICAL TIMEOUT SETTING ---
        # Default proxy_read_timeout is 60s. A WebSocket connection with no
        # activity for 60s will be silently killed by NGINX.
        # Options:
        #   1. Set a very long timeout (e.g., 1 day for persistent connections)
        #   2. Configure the application to send WebSocket ping frames every ~30s
        # Option 2 is strongly preferred — it also detects dead connections.
        proxy_read_timeout 3600s;   # 1 hour; tune to your ping interval + margin

        # Send timeout — time between writes to the upstream
        proxy_send_timeout 3600s;

        # Disable buffering for WebSockets — data must flow in real time
        proxy_buffering off;

        # Increase buffer size for WebSocket frames (default 4k is too small
        # for large messages)
        proxy_buffer_size 64k;
    }

    # Regular HTTP on the same server (proxy_pass works normally)
    location / {
        proxy_pass http://ws_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Why the `map` Block is Required

Without the `map` block, you must hardcode `Connection: upgrade`. This breaks non-WebSocket requests through that location because the `Connection: upgrade` header on a regular GET request is malformed. The map cleanly handles both cases:

```nginx
# Wrong — breaks regular HTTP through this location
proxy_set_header Connection "upgrade";

# Right — conditional via map
proxy_set_header Connection $connection_upgrade;
```

### Server-Side Ping Configuration

Configure your WebSocket server to send pings every 25–30 seconds. This resets the NGINX `proxy_read_timeout` timer and lets you set a tight read timeout that still catches dead connections:

```nginx
# With server-side pings every 30s, this timeout catches dead connections
# without being so long it wastes resources
proxy_read_timeout 75s;
```

### Mixed HTTP/WebSocket on One Location

Some frameworks upgrade from HTTP to WebSocket on the same path:

```nginx
location /app/ {
    proxy_pass http://ws_backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade    $http_upgrade;
    proxy_set_header Connection $connection_upgrade;
    proxy_set_header Host       $host;
    proxy_set_header X-Real-IP  $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    # Timeout must accommodate WebSocket idle periods
    proxy_read_timeout 3600s;
    proxy_buffering off;
}
```

---

## 6. Static Files with Proxy Fallback

Serve static files directly from disk; fall through to the backend only for dynamic content. This pattern eliminates backend load for assets and is critical for performance.

```nginx
server {
    listen 443 ssl;
    http2 on;
    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    root /var/www/myapp/public;

    # --- Pure static file serving ---
    # Aggressive caching for versioned/hashed assets
    location ~* \.(js|css|woff2?|ttf|eot|svg|ico)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;  # skip logging for static assets
        # try_files serves from disk; 404 if missing (do NOT fallback to proxy)
        try_files $uri =404;
    }

    # Images — shorter cache than code assets (might change without hash)
    location ~* \.(jpg|jpeg|png|gif|webp|avif)$ {
        expires 30d;
        add_header Cache-Control "public";
        access_log off;
        try_files $uri =404;
    }

    # --- Static with proxy fallback (SPA pattern) ---
    # Try the file on disk; if missing, pass to the backend.
    # The backend handles routing and API calls.
    location / {
        try_files $uri $uri/ @backend;
    }

    # Named location for proxy fallback — avoids infinite recursion
    location @backend {
        proxy_pass http://myapp_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP        $remote_addr;
        proxy_set_header X-Forwarded-For  $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 30s;
    }

    # --- SPA index fallback (React/Vue/Angular client-side routing) ---
    # Everything that isn't a file → serve index.html for client router
    location / {
        try_files $uri $uri/ /index.html;
    }
    # Note: when using /index.html fallback, API calls must be under a
    # separate location block with proxy_pass, or the SPA will get HTML
    # back for every 404'd API call.

    # --- API calls still go to backend even with SPA pattern ---
    location /api/ {
        proxy_pass http://myapp_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP        $remote_addr;
        proxy_set_header X-Forwarded-For  $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### `try_files` Argument Semantics

```nginx
try_files $uri $uri/ @backend;
```

- `$uri` — look for the exact file on disk
- `$uri/` — look for a directory (serves index within it per autoindex/index)
- `@backend` — named location fallback (a proxy or a return statement)
- `/index.html` — serve a specific file as final fallback (SPA pattern)
- `=404` — return 404 if not found (prevents falling through to anything else)

The final argument is the fallback. If it is a URI (starts with `/`), it causes an internal redirect. If it is a named location (`@name`), it is an internal forward. Do not use `=404` in a catch-all location — it will break the fallback chain.

---

## 7. Load Balanced Upstreams

### Round-Robin (Default)

Requests are distributed evenly across servers in order. Weight adjusts the ratio.

```nginx
upstream backend_rr {
    # Round-robin is implicit — no directive needed
    server 10.0.1.10:3000 weight=3;  # gets 3x more requests than weight=1
    server 10.0.1.11:3000 weight=1;
    server 10.0.1.12:3000 weight=1;

    # Failure detection: mark unavailable after 3 failures in 30 seconds
    # Server stays unavailable for the fail_timeout duration (30s here)
    # then NGINX tries it again with a single probe request
    server 10.0.1.10:3000 weight=3 max_fails=3 fail_timeout=30s;

    keepalive 32;
    keepalive_requests 1000;
    keepalive_timeout  60s;
}
```

### Least Connections

Best for backends with variable request duration (e.g., mixed API + file upload paths):

```nginx
upstream backend_lc {
    least_conn;

    server 10.0.1.10:3000;
    server 10.0.1.11:3000;
    server 10.0.1.12:3000;

    keepalive 32;
}
```

### IP Hash (Session Affinity)

Pins each client IP to a specific backend. Use when your application stores session state in memory (which you should avoid, but legacy systems exist). The first three octets of IPv4 are hashed; for IPv6 the entire address is used.

```nginx
upstream backend_iphash {
    ip_hash;

    server 10.0.1.10:3000;
    server 10.0.1.11:3000;
    # Use 'down' not 'weight=0' when taking a server offline under ip_hash.
    # 'down' preserves the hash distribution; removing the server shifts
    # all mappings and routes existing sessions to wrong backends.
    server 10.0.1.12:3000 down;

    keepalive 16;
}
```

### URI Hash (Consistent Hashing)

Routes requests for the same URI to the same backend — useful for cache locality:

```nginx
upstream backend_hash {
    hash $request_uri consistent;

    server 10.0.1.10:3000;
    server 10.0.1.11:3000;
    server 10.0.1.12:3000;

    keepalive 16;
}
```

`consistent` uses ketama consistent hashing, which minimises re-mapping when a server is added or removed (only ~1/N keys remap). Without `consistent`, adding a server remaps ~50% of keys.

### Full Production Upstream Block

```nginx
upstream backend_prod {
    least_conn;

    # Primary servers
    server 10.0.1.10:3000 weight=1 max_fails=3 fail_timeout=30s;
    server 10.0.1.11:3000 weight=1 max_fails=3 fail_timeout=30s;
    server 10.0.1.12:3000 weight=1 max_fails=3 fail_timeout=30s;

    # Backup: only used when ALL primary servers are unavailable
    # Not used in normal load balancing rotation
    server 10.0.1.20:3000 backup;

    # Keepalive connection pool
    # Number should be (workers × expected concurrent upstream conns) / servers
    keepalive         64;
    keepalive_requests 1000;
    keepalive_timeout  75s;
    keepalive_time     1h;
}

server {
    listen 443 ssl;
    http2 on;
    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    location / {
        proxy_pass http://backend_prod;

        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP        $remote_addr;
        proxy_set_header X-Forwarded-For  $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 10s;
        proxy_send_timeout    60s;
        proxy_read_timeout    60s;

        # Retry on these conditions — ONLY safe for idempotent requests
        # Add http_502/503/504 for upstream application errors
        proxy_next_upstream         error timeout http_502 http_503 http_504;
        proxy_next_upstream_tries   3;    # try at most 3 servers
        proxy_next_upstream_timeout 15s;  # total time budget for all retries

        proxy_buffering on;
        proxy_buffer_size 16k;
        proxy_buffers     8 16k;
    }
}
```

### Failure Detection and Recovery

- `max_fails=3 fail_timeout=30s` means: if 3 requests fail within any 30-second window, mark the server unavailable for 30 seconds. After 30 seconds, NGINX sends one probe request. If it succeeds, the server re-enters rotation. If it fails, the 30-second ban resets.
- Setting `max_fails=0` disables passive health checks entirely. The server is always considered available.
- NGINX Plus has active health checks (`health_check` directive); open-source NGINX only has passive failure detection.

### `proxy_next_upstream` Safety Warning

`proxy_next_upstream` retries failed requests on the next upstream server. This is safe only for idempotent methods (GET, HEAD, PUT, DELETE, OPTIONS). A failed POST may have been processed by the upstream before the connection dropped — retrying it will process it twice. By default, NGINX only retries non-idempotent methods when `non_idempotent` is explicitly listed.

```nginx
# Never add this unless you have idempotency guarantees for POST:
# proxy_next_upstream ... non_idempotent;
```

---

## 8. SSL Termination

NGINX handles TLS and forwards plain HTTP to backends. This is the correct architecture for almost every deployment: keep TLS complexity at the edge, let backends communicate in plaintext on a private network.

```nginx
# /etc/nginx/conf.d/ssl-params.conf — include in every SSL server block
#
# Modern SSL configuration. Source: Mozilla SSL Config Generator (Intermediate)
# https://ssl-config.mozilla.org/

ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
ssl_prefer_server_ciphers off;  # let client choose from cipher list

# Session cache — shared across workers, 10m ≈ 40,000 sessions
ssl_session_cache   shared:SSL:10m;
ssl_session_timeout 1d;
ssl_session_tickets off;  # tickets have forward secrecy issues; disable for compliance

# OCSP stapling — fetch upstream cert status and attach to handshake
ssl_stapling         on;
ssl_stapling_verify  on;
resolver 1.1.1.1 8.8.8.8 valid=300s;
resolver_timeout 5s;

# DH parameters — generate with: openssl dhparam -out /etc/nginx/dhparam.pem 2048
ssl_dhparam /etc/nginx/dhparam.pem;

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name example.com www.example.com;

    include /etc/nginx/conf.d/ssl-params.conf;
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;

    # HSTS — tell browsers to only use HTTPS for 2 years
    # DANGER: Do not enable until you are certain all subdomains have HTTPS.
    # Once a browser caches HSTS, removing it requires waiting for the max-age.
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

    # Additional security headers
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    location / {
        proxy_pass http://backend_prod;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP        $remote_addr;
        proxy_set_header X-Forwarded-For  $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# HTTP → HTTPS redirect — always use a separate server block
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;

    # Allow Let's Encrypt HTTP challenge (must precede the redirect)
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}
```

### Re-Encrypting to Backend (mTLS)

When the backend also requires TLS (zero-trust internal network, PCI-DSS requirement):

```nginx
location / {
    # Use https:// in proxy_pass to encrypt the connection to backend
    proxy_pass https://backend_prod;

    # Verify backend certificate against your internal CA
    proxy_ssl_verify              on;
    proxy_ssl_verify_depth        2;
    proxy_ssl_trusted_certificate /etc/nginx/certs/internal-ca.pem;

    # Present a client certificate to the backend (mTLS)
    proxy_ssl_certificate     /etc/nginx/certs/nginx-client.crt;
    proxy_ssl_certificate_key /etc/nginx/certs/nginx-client.key;

    # Use SNI so the backend can identify which cert to present
    proxy_ssl_server_name on;
    proxy_ssl_name        backend.internal.example.com;

    # Reuse SSL sessions between upstream requests
    proxy_ssl_session_reuse on;

    proxy_ssl_protocols TLSv1.2 TLSv1.3;
}
```

---

## 9. Caching Proxy

NGINX's caching subsystem stores upstream responses on disk. Subsequent requests for the same resource are served from cache without touching the backend.

### Cache Zone Configuration

Defined in the `http` block — `proxy_cache_path` cannot go in `server` or `location`.

```nginx
# /etc/nginx/nginx.conf — inside http { }

# path:          disk location for cache files
# levels=1:2:    two-level directory hierarchy (reduces files per dir)
# keys_zone:     shared memory for cache keys and metadata (not file content)
#                10m ≈ 80,000 keys; size the keys_zone to your expected unique key count
# inactive=60m:  purge entries not accessed within 60 minutes
# max_size=10g:  total disk cap; NGINX auto-evicts LRU when exceeded
# use_temp_path=off: write directly to cache dir, avoid extra copy from temp dir
proxy_cache_path /var/cache/nginx/app
    levels=1:2
    keys_zone=app_cache:10m
    inactive=60m
    max_size=10g
    use_temp_path=off;
```

### Basic Caching Location

```nginx
server {
    listen 443 ssl;
    http2 on;
    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    location / {
        proxy_pass http://backend_prod;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP        $remote_addr;
        proxy_set_header X-Forwarded-For  $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # --- Cache directives ---
        proxy_cache     app_cache;

        # Cache key: default is fine for most use cases.
        # Include $cookie_session_id if responses are user-specific and
        # you MUST cache them (rare; understand the privacy implications).
        proxy_cache_key "$scheme$host$request_uri";

        # Cache by response code with per-code TTLs
        proxy_cache_valid 200 301  1h;   # successful and permanent redirects
        proxy_cache_valid 302       5m;   # temporary redirects (shorter)
        proxy_cache_valid 404       1m;   # cache 404s to reduce backend pressure
        proxy_cache_valid any       10s;  # any other code, very short

        # --- Cache bypass: skip cache for these requests ---
        # Authentication headers mean the response is private
        proxy_cache_bypass $http_authorization;
        # Pragma/Cache-Control: no-cache from client forces cache miss
        proxy_cache_bypass $http_pragma $http_cache_control;
        # Custom bypass header for debugging
        proxy_cache_bypass $http_x_bypass_cache;

        # --- No-store: don't save response to cache ---
        # Same conditions as bypass — if we bypass, we also don't store
        proxy_no_cache $http_authorization;
        proxy_no_cache $http_pragma $http_cache_control;
        proxy_no_cache $http_x_bypass_cache;

        # --- Stale content serving ---
        # Serve stale cached response if backend returns these errors.
        # This is a reliability feature — users get old content rather
        # than an error page during backend outages.
        proxy_cache_use_stale error timeout updating
                              http_500 http_502 http_503 http_504;

        # --- Lock: only one request can populate a cache entry ---
        # Prevents cache stampede (thundering herd) on cache miss
        proxy_cache_lock on;
        proxy_cache_lock_timeout 5s;
        proxy_cache_lock_age     5s;

        # --- Background update ---
        # When cache_use_stale includes 'updating': serve stale while
        # one background request fetches the fresh content
        proxy_cache_background_update on;

        # --- Revalidation ---
        # If upstream sends ETag/Last-Modified, use conditional GET on expiry
        # instead of unconditional re-fetch
        proxy_cache_revalidate on;

        # --- Don't cache if fewer than 2 requests have been made ---
        # Avoids caching one-off requests that won't benefit from caching
        proxy_cache_min_uses 2;

        # Expose cache status in response for debugging
        add_header X-Cache-Status $upstream_cache_status always;
    }
}
```

### Cache Status Header Values

`$upstream_cache_status` will be one of:

| Value | Meaning |
|---|---|
| `MISS` | Not in cache; fetched from upstream |
| `HIT` | Served from cache |
| `EXPIRED` | Was in cache but expired; fetched fresh |
| `STALE` | Served from stale cache (use_stale triggered) |
| `UPDATING` | Stale served; background fetch in progress |
| `REVALIDATED` | Cache revalidated with conditional GET |
| `BYPASS` | Cache bypassed (proxy_cache_bypass triggered) |

### Cache Purge Without NGINX Plus

NGINX open-source has no built-in purge endpoint. Options:

1. **File deletion**: Cache files are at `<cache_path>/MD5(key)`. You can delete them directly.
2. **`proxy_cache_purge` module**: Available in some distributions as a third-party module.
3. **TTL strategy**: Use very short TTLs (`proxy_cache_valid 200 30s`) for volatile content.

```bash
# Manual purge by key: compute the file path from the key hash
# Key: "https://example.com/api/v1/products"
echo -n "https://example.com/api/v1/products" | md5sum
# Result: a1b2c3d4e5f6...
# Cache file: /var/cache/nginx/app/4/d3/a1b2c3d4e5f6...
```

### What NOT to Cache

Never cache:

- Authenticated responses (unless the auth token is part of the cache key and privacy impact is understood)
- POST/PUT/PATCH/DELETE responses — these mutate state
- Real-time data (stock prices, live counts)
- Set-Cookie responses (caching cookies is a session hijacking vulnerability)

NGINX respects upstream `Cache-Control: private` and `Cache-Control: no-store` by default and will not cache those responses.

---

## 10. Rate Limiting

Rate limiting protects backends from abuse and ensures fair resource distribution. NGINX's rate limiting uses the leaky bucket algorithm — requests arriving faster than the defined rate either wait in a queue or are rejected.

```nginx
# Defined in http { } block

# Limit by client IP — good for public endpoints
# 10r/s = 10 requests per second per unique IP
limit_req_zone $binary_remote_addr zone=per_ip:10m rate=10r/s;

# Limit by API key header — better for authenticated API clients
# Allows tracking per-client rather than per-IP (important with shared IPs/NAT)
limit_req_zone $http_x_api_key zone=per_api_key:10m rate=100r/s;

# Limit by IP + URI combined — for specific expensive endpoints
limit_req_zone $binary_remote_addr$uri zone=per_ip_per_uri:10m rate=1r/s;

# For login endpoints — very tight limit to prevent brute force
limit_req_zone $binary_remote_addr zone=login_limit:10m rate=5r/m;

# Limit connections (concurrent, not rate) — useful for WebSockets
limit_conn_zone $binary_remote_addr zone=conn_per_ip:10m;
```

```nginx
server {
    listen 443 ssl;
    http2 on;
    server_name api.example.com;

    ssl_certificate     /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;

    # --- General API rate limit ---
    location /api/ {
        proxy_pass http://app_api;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP        $remote_addr;
        proxy_set_header X-Forwarded-For  $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # burst=20: allow up to 20 requests to queue before rejecting
        # nodelay: process queued requests immediately (don't slow them to the rate)
        # Without nodelay: requests are delayed to enforce the exact rate.
        # With nodelay: burst requests go through immediately; next token required for subsequent.
        limit_req zone=per_ip burst=20 nodelay;

        # Return 429 on rate limit (default is 503 — incorrect semantics)
        limit_req_status 429;

        # Max 50 concurrent connections from a single IP
        limit_conn conn_per_ip 50;
        limit_conn_status 429;
    }

    # --- Login endpoint: strict brute-force protection ---
    location = /api/auth/login {
        proxy_pass http://app_api;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 5 requests per minute, no burst — brute force must wait
        limit_req zone=login_limit;
        limit_req_status 429;
    }

    # --- File upload endpoint: per-URI limit ---
    location /api/upload {
        # Only 1 concurrent upload request per IP
        limit_req zone=per_ip_per_uri burst=2 nodelay;
        limit_req_status 429;

        client_max_body_size 100m;

        proxy_pass http://app_api;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Upload-specific timeouts
        proxy_read_timeout 300s;
        proxy_send_timeout 300s;
        proxy_request_buffering off;  # stream upload to backend, don't buffer on disk
    }

    # --- Webhook endpoint: exclude from rate limiting ---
    # Webhooks from known providers should not be rate-limited by IP
    # because they may batch from a single IP
    location /api/webhook/ {
        # Verify webhook signature in the backend instead of IP-limiting
        proxy_pass http://app_api;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # No limit_req here — intentional
    }
}
```

### Rate Limit Logging

Log rate-limited requests to a separate file for monitoring:

```nginx
# In http { } block
limit_req_log_level warn;   # default error — warn is less noisy

# In server or location
error_log /var/log/nginx/rate_limit.log warn;
```

### `burst` and `nodelay` Explained

```
rate=10r/s, burst=20, nodelay

Timeline (requests arrive in 0.5s):
Request 1-10:  processed immediately (within rate)
Request 11-30: processed immediately (burst queue absorbs them)
Request 31+:   rejected with 429 until burst queue drains

Without nodelay (with delay):
Request 11-30: allowed but each delayed by 1/rate = 100ms
Request 31+:   rejected

nodelay is almost always what you want — users experience no delay
for burst traffic, but sustained abuse is rejected.
```

---

## 11. IP Allowlisting for Admin Paths

Protect sensitive paths with `allow`/`deny` directives. These are evaluated in order; the first match wins.

```nginx
# Define a geo block for dynamic allowlisting based on IP ranges
# geo { } is in the http { } block
geo $is_trusted_ip {
    default          0;
    10.0.0.0/8       1;  # private network
    172.16.0.0/12    1;  # Docker/container networks
    192.168.0.0/16   1;  # local LAN
    203.0.113.50     1;  # office static IP
    2001:db8::/32    1;  # IPv6 office range (example)
}

server {
    listen 443 ssl;
    http2 on;
    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # --- Pattern 1: Hard allow/deny list (static) ---
    location /admin/ {
        allow 10.0.0.0/8;
        allow 192.168.1.0/24;
        allow 203.0.113.50;   # office
        deny  all;            # deny everything else

        proxy_pass http://app_admin;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP        $remote_addr;
        proxy_set_header X-Forwarded-For  $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # --- Pattern 2: Geo variable (dynamic, easier to maintain) ---
    location /internal/ {
        if ($is_trusted_ip = 0) {
            return 403;
        }

        proxy_pass http://app_admin;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # --- Pattern 3: Allowlist + rate limit combined ---
    # Allow from trusted IPs without rate limiting, apply strict limits to others
    location /api/admin/ {
        # Trusted IPs get through without rate limiting
        # External IPs get a very tight rate limit
        limit_req zone=per_ip burst=5 nodelay;

        allow 10.0.0.0/8;
        allow 203.0.113.50;
        # Note: deny all here would block external IPs before rate limiting applies.
        # If you want to allow external IPs (with rate limiting), omit deny.
        # If admin should ONLY be internal, use deny all.
        deny  all;

        proxy_pass http://app_admin;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # --- Pattern 4: Metrics/health endpoints ---
    location /metrics {
        # Prometheus scraper IP — allow only the monitoring server
        allow 10.0.2.100;
        deny  all;

        proxy_pass http://app_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
    }

    location /health {
        # Health checks from load balancer VIPs
        allow 10.0.3.0/24;
        allow 127.0.0.1;
        deny  all;

        access_log off;  # don't pollute access logs with health check noise

        proxy_pass http://app_backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
    }

    # Public API
    location / {
        proxy_pass http://app_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP        $remote_addr;
        proxy_set_header X-Forwarded-For  $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### `allow`/`deny` Evaluation Order

`allow` and `deny` are evaluated in the order they appear, and the first match terminates evaluation. A common mistake is putting `deny all` before specific allows:

```nginx
# WRONG — deny all blocks everyone, allows are never reached
deny  all;
allow 10.0.0.0/8;

# CORRECT — specific allows first, deny all as catchall
allow 10.0.0.0/8;
deny  all;
```

---

## 12. Real IP Passthrough

When NGINX sits behind a CDN, upstream load balancer, or cloud provider, `$remote_addr` contains the proxy's IP, not the client's IP. Restoring the real client IP requires configuring NGINX to trust specific proxy addresses and read the IP from forwarded headers.

### X-Forwarded-For and X-Real-IP

```nginx
# /etc/nginx/conf.d/real-ip.conf — include in http { }

# Trust only specific upstream proxies — do NOT trust all IPs
# Only list IPs that you control or that are guaranteed to set headers correctly

# Cloudflare IP ranges (keep updated — fetch from https://www.cloudflare.com/ips/)
set_real_ip_from 103.21.244.0/22;
set_real_ip_from 103.22.200.0/22;
set_real_ip_from 103.31.4.0/22;
set_real_ip_from 104.16.0.0/13;
set_real_ip_from 104.24.0.0/14;
set_real_ip_from 108.162.192.0/18;
set_real_ip_from 131.0.72.0/22;
set_real_ip_from 141.101.64.0/18;
set_real_ip_from 162.158.0.0/15;
set_real_ip_from 172.64.0.0/13;
set_real_ip_from 173.245.48.0/20;
set_real_ip_from 188.114.96.0/20;
set_real_ip_from 190.93.240.0/20;
set_real_ip_from 197.234.240.0/22;
set_real_ip_from 198.41.128.0/17;
set_real_ip_from 2400:cb00::/32;
set_real_ip_from 2606:4700::/32;
set_real_ip_from 2803:f800::/32;
set_real_ip_from 2405:b500::/32;
set_real_ip_from 2405:8100::/32;
set_real_ip_from 2a06:98c0::/29;
set_real_ip_from 2c0f:f248::/32;

# Use the rightmost non-trusted IP in X-Forwarded-For as the real IP.
# This is the correct approach — leftmost IPs can be spoofed by clients.
real_ip_header X-Forwarded-For;
real_ip_recursive on;

# After these directives, $remote_addr contains the real client IP
# and $proxy_add_x_forwarded_for appends the real client IP correctly
```

### PROXY Protocol (Layer 4 Real IP)

When the upstream proxy speaks the PROXY protocol (HAProxy, AWS NLB, some CDNs), NGINX can extract the real IP at the TCP level — before any HTTP header parsing. This is more reliable than header-based passthrough.

```nginx
server {
    # Accept PROXY protocol on the listener
    listen 443 ssl proxy_protocol;
    listen [::]:443 ssl proxy_protocol;
    http2 on;
    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # Trust PROXY protocol from these upstream addresses only
    set_real_ip_from 10.0.0.0/8;
    real_ip_header   proxy_protocol;

    # $remote_addr is now the real client IP (from PROXY protocol)
    # $proxy_protocol_addr also holds the real IP explicitly

    location / {
        proxy_pass http://backend_prod;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host              $host;

        # Set X-Real-IP from the PROXY protocol address
        proxy_set_header X-Real-IP        $proxy_protocol_addr;
        proxy_set_header X-Forwarded-For  $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Port  $proxy_protocol_port;
    }
}
```

### Security Warning: `real_ip_recursive` and Trust

Without `real_ip_recursive on`, NGINX uses the last IP in `X-Forwarded-For`, which is set by the immediate upstream. A client can inject an arbitrary IP at the beginning of the XFF chain if you are not careful.

```
# Client sends: X-Forwarded-For: 1.2.3.4, 5.6.7.8
# Upstream CDN appends: X-Forwarded-For: 1.2.3.4, 5.6.7.8, <real_client_ip>

# With real_ip_recursive on and the CDN IP trusted:
# NGINX strips the trusted CDN IP and takes the rightmost remaining → <real_client_ip>

# Without real_ip_recursive:
# NGINX takes the last IP → CDN's IP (wrong)
```

Always use `real_ip_recursive on`. Only ever trust IPs you control.

### Forwarding Real IP to Backend

Once NGINX has restored `$remote_addr` via `set_real_ip_from`/`real_ip_header`, the standard header directives work correctly:

```nginx
proxy_set_header X-Real-IP        $remote_addr;            # restored real IP
proxy_set_header X-Forwarded-For  $proxy_add_x_forwarded_for;  # appends real IP
proxy_set_header X-Forwarded-Proto $scheme;                 # http or https
```

`$proxy_add_x_forwarded_for` expands to the existing `X-Forwarded-For` header value plus `, $remote_addr`. After real IP restoration, this produces a correct forwarding chain.

---

## 13. Hardening and Operational Details

### Global `nginx.conf` Tuning

```nginx
worker_processes auto;   # one worker per CPU core
worker_rlimit_nofile 65535;  # match OS ulimit

events {
    worker_connections 4096;  # max (worker_processes × worker_connections) = max connections
    use epoll;                # Linux: epoll is fastest
    multi_accept on;          # accept all pending connections in one call
}

http {
    # Sendfile for static file serving (zero-copy kernel bypass)
    sendfile on;
    tcp_nopush  on;   # batch send (requires sendfile on)
    tcp_nodelay on;   # disable Nagle for keepalive connections

    # Keepalive to clients
    keepalive_timeout  75s;
    keepalive_requests 1000;

    # Client body handling
    client_body_timeout    12s;
    client_header_timeout  12s;
    send_timeout           10s;
    client_body_buffer_size 128k;

    # Hide NGINX version from error pages and Server header
    server_tokens off;

    # Gzip compression for proxied responses
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types
        text/plain text/css text/xml text/javascript
        application/json application/javascript application/xml
        application/rss+xml application/atom+xml
        image/svg+xml;
    gzip_min_length 1024;  # don't compress tiny responses
}
```

### Reload Without Downtime

```bash
# Test config before reload — always do this
nginx -t

# Graceful reload: worker processes finish current requests before replacing
nginx -s reload
```

### Debugging Upstream Issues

```nginx
# Temporarily enable debug logging for a specific upstream
error_log /var/log/nginx/debug.log debug;

# Expose upstream variables in response headers (development only — never production)
add_header X-Upstream-Addr   $upstream_addr always;
add_header X-Upstream-Status $upstream_status always;
add_header X-Upstream-Time   $upstream_response_time always;
add_header X-Cache-Status    $upstream_cache_status always;
```

### Common Pitfalls Checklist

| Problem | Symptom | Fix |
|---|---|---|
| Wrong `Host` header | Backend 404 or redirect loop | `proxy_set_header Host $host` |
| `Connection: close` on keepalive upstream | High upstream latency, port exhaustion | `proxy_set_header Connection ""` |
| HTTP/1.0 to upstream | No keepalive, chunked encoding issues | `proxy_http_version 1.1` |
| Trailing slash mismatch | Double slashes or missing path segments upstream | Match trailing slashes on both `location` and `proxy_pass` |
| 60s WebSocket timeout | Clients disconnected after 1 minute of idle | `proxy_read_timeout 3600s` + server-side pings |
| `$remote_addr` is CDN IP | Rate limiting and geo-blocking apply to CDN | `set_real_ip_from` + `real_ip_header` |
| `proxy_next_upstream` on POST | Duplicate order submissions | Never include `non_idempotent` without application-level idempotency |
| Cache stores authenticated responses | Session data leakage between users | `proxy_no_cache $http_authorization` |
| Buffer size too small | Upstream headers truncated | Increase `proxy_buffer_size` |
| `max_fails=0` | Unhealthy backend receives traffic forever | Use `max_fails=3 fail_timeout=30s` minimum |

### Upstream Keepalive Sizing Formula

```
keepalive_connections = ceil(
    (expected_RPS × avg_upstream_latency_seconds) / upstream_server_count
) × 1.25   # 25% headroom
```

Example: 1000 RPS, 50ms average latency, 4 upstream servers:
```
ceil((1000 × 0.050) / 4) × 1.25 = ceil(12.5) × 1.25 ≈ 16 connections per worker
```

With 4 NGINX workers: 64 total keepalive connections — size the upstream `keepalive` directive to 16 per worker.

### Health Check Endpoint Pattern

Expose a dedicated health endpoint that verifies backend connectivity without hitting business logic:

```nginx
location = /nginx-health {
    access_log off;
    return 200 "healthy\n";
    add_header Content-Type text/plain;
}

# Deep health check — proxied to backend
location = /health/ready {
    access_log off;
    allow 10.0.0.0/8;
    allow 127.0.0.1;
    deny  all;

    proxy_pass http://backend_prod/health;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_connect_timeout 2s;
    proxy_read_timeout    5s;
}
```

---

*Last updated: 2026-04-06. Tested against NGINX stable 1.26.x and mainline 1.29.x. Directives and defaults are sourced from the official nginx.org module documentation.*
