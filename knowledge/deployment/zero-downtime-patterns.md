# Zero-Downtime Deployment Patterns

Opinionated guide for production deployments. Written for engineers who have been paged at 2am and have the scars to prove it. Every pattern here has tradeoffs — pick the right tool for the job, not the one that sounds impressive.

---

## Table of Contents

1. [Core Principles](#core-principles)
2. [Blue-Green Deployment](#blue-green-deployment)
3. [Rolling Deployment](#rolling-deployment)
4. [Canary Deployment](#canary-deployment)
5. [Rsync + Symlink Deploy (Capistrano-Style)](#rsync--symlink-deploy-capistrano-style)
6. [GitHub Actions SSH Deploy Job](#github-actions-ssh-deploy-job)
7. [Database Migrations: Expand/Contract Pattern](#database-migrations-expandcontract-pattern)
8. [Rollback Runbook Template](#rollback-runbook-template)

---

## Core Principles

Before diving into specific patterns, internalize these rules. They apply regardless of which strategy you choose.

**Health checks are the only truth.** Your deploy script does not know if the application is healthy — only a health check does. Never assume success because a process started. Never skip the health check because "it's just a config change." Every deploy that skips a health check is a future incident.

**Automate the rollback, not just the deploy.** If rolling back requires more than 3 manual steps, you will not do it cleanly under pressure. If the rollback procedure lives only in someone's head, you're one bad day away from a disaster. The rollback path must be tested at least quarterly.

**Immutable artifacts.** Build once, deploy the artifact. Never run `git pull` on a production server and never run `npm install` in production. If a package registry is unavailable during your deploy, you should not find out the hard way. Build your artifact in CI, store it, and push the same artifact to every environment.

**Decouple deploy from release.** Deploying code and enabling a feature are separate events. Use feature flags. A deploy failing at 4pm on Friday is an inconvenience. A broken feature flag at 4pm on Friday is an incident.

**Drain connections before killing processes.** `SIGKILL` is not a deploy strategy. Use `SIGTERM`, wait for in-flight requests to finish, then confirm the process is gone. Set `worker_shutdown_timeout` in whatever process manager you use. Give it enough time — at least 30 seconds more than your P99 request latency.

---

## Blue-Green Deployment

Blue-green is the safest strategy for stateless applications. You maintain two identical environments. One is live (blue), one is idle (green). You deploy to idle, test it, then flip traffic. If something goes wrong, you flip back.

**When to use it:** Any stateless web app where you can afford to run two full environments. Ideal when deploys are infrequent but correctness is critical.

**When NOT to use it:** When your app is stateful and sessions are not externalized (fix that first). When you can't afford two environments. When you have database schema changes that aren't backward-compatible (use expand/contract first).

### Nginx Configuration

The key insight: Nginx's `upstream` block determines where traffic goes. You define both environments and point the active upstream at whichever one is live. The inactive upstream block still exists — it just isn't referenced by any `proxy_pass`.

```nginx
# /etc/nginx/conf.d/app.conf

# Blue environment — port 3001
upstream blue {
    server 127.0.0.1:3001 max_fails=3 fail_timeout=30s;
    keepalive 32;
}

# Green environment — port 3002
upstream green {
    server 127.0.0.1:3002 max_fails=3 fail_timeout=30s;
    keepalive 32;
}

# This file is managed by the deploy script.
# It contains a single line: proxy_pass http://blue;  OR  proxy_pass http://green;
include /etc/nginx/conf.d/active_upstream.conf;

server {
    listen 80;
    server_name app.example.com;

    location /health {
        # Direct health check bypass — never goes through upstream
        proxy_pass http://127.0.0.1:3001/health;
        access_log off;
    }

    location / {
        include /etc/nginx/conf.d/active_upstream.conf;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 5s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        proxy_next_upstream error timeout invalid_header http_502 http_503;
        proxy_next_upstream_tries 2;
    }
}
```

The `active_upstream.conf` file contains exactly one line and is the only thing the deploy script modifies:

```nginx
# /etc/nginx/conf.d/active_upstream.conf
# This file is written by the deploy script — do not edit manually
proxy_pass http://blue;
```

### Deploy Script with Auto-Revert

This script encodes the entire blue-green lifecycle. It determines the inactive environment, deploys there, runs a health check gate, swaps the upstream, runs a post-swap health check, and reverts if anything fails.

```bash
#!/usr/bin/env bash
# /opt/deploy/blue-green-deploy.sh
# Usage: ./blue-green-deploy.sh <image_tag_or_artifact_version>
# Requires: curl, nginx, systemctl (or docker/pm2 — adapt START_CMD/STOP_CMD)

set -euo pipefail

DEPLOY_VERSION="${1:?Usage: $0 <version>}"
NGINX_UPSTREAM_FILE="/etc/nginx/conf.d/active_upstream.conf"
NGINX_APP_CONF="/etc/nginx/conf.d/app.conf"
HEALTH_CHECK_URL="http://127.0.0.1"
HEALTH_CHECK_PATH="/health"
HEALTH_CHECK_RETRIES=20
HEALTH_CHECK_INTERVAL=3   # seconds between retries
BLUE_PORT=3001
GREEN_PORT=3002
LOG_PREFIX="[blue-green]"
DEPLOY_USER="deploy"

# --- Helpers ---

log() { echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) ${LOG_PREFIX} $*"; }
die() { log "FATAL: $*"; exit 1; }

get_active_upstream() {
    grep -oP '(?<=proxy_pass http://)[\w]+(?=;)' "${NGINX_UPSTREAM_FILE}" || die "Cannot read active upstream from ${NGINX_UPSTREAM_FILE}"
}

get_inactive_upstream() {
    local active="$1"
    if [[ "$active" == "blue" ]]; then echo "green"; else echo "blue"; fi
}

get_port_for_env() {
    local env="$1"
    if [[ "$env" == "blue" ]]; then echo "${BLUE_PORT}"; else echo "${GREEN_PORT}"; fi
}

health_check() {
    local port="$1"
    local label="$2"
    local url="${HEALTH_CHECK_URL}:${port}${HEALTH_CHECK_PATH}"
    local attempt=0

    log "Health checking ${label} at ${url} (${HEALTH_CHECK_RETRIES} attempts, ${HEALTH_CHECK_INTERVAL}s interval)"

    while [[ $attempt -lt $HEALTH_CHECK_RETRIES ]]; do
        attempt=$((attempt + 1))
        local http_code
        http_code=$(curl -sf -o /dev/null -w "%{http_code}" --max-time 5 "${url}" 2>/dev/null || echo "000")

        if [[ "$http_code" == "200" ]]; then
            log "Health check passed for ${label} (attempt ${attempt}/${HEALTH_CHECK_RETRIES})"
            return 0
        fi

        log "Health check attempt ${attempt}/${HEALTH_CHECK_RETRIES} for ${label}: HTTP ${http_code}"
        sleep "${HEALTH_CHECK_INTERVAL}"
    done

    log "Health check FAILED for ${label} after ${HEALTH_CHECK_RETRIES} attempts"
    return 1
}

start_app() {
    local env="$1"
    local port="$2"
    local version="$3"
    log "Starting ${env} on port ${port} with version ${version}"
    # Adapt this to your process manager. Examples:
    #   systemctl start "app-${env}"
    #   docker run -d --name "app-${env}" -p "${port}:3000" "myapp:${version}"
    #   pm2 start ecosystem.config.js --only "app-${env}"
    systemctl start "app-${env}"
    log "${env} started"
}

stop_app() {
    local env="$1"
    log "Stopping ${env}"
    # systemctl stop "app-${env}"
    # docker stop "app-${env}" && docker rm "app-${env}"
    systemctl stop "app-${env}"
    log "${env} stopped"
}

set_upstream() {
    local target="$1"
    log "Switching nginx upstream to ${target}"
    echo "proxy_pass http://${target};" > "${NGINX_UPSTREAM_FILE}"
}

reload_nginx() {
    log "Reloading nginx (graceful — no dropped connections)"
    nginx -t || die "nginx config test failed — aborting upstream switch"
    # nginx -s reload sends SIGHUP to the master process.
    # The master spawns new workers with the new config and drains old workers.
    # In-flight connections on old workers are NOT dropped.
    nginx -s reload
    # Give workers a moment to pick up the new upstream
    sleep 1
}

# --- Main ---

log "=== Blue-Green Deploy: version=${DEPLOY_VERSION} ==="

ACTIVE=$(get_active_upstream)
INACTIVE=$(get_inactive_upstream "${ACTIVE}")
INACTIVE_PORT=$(get_port_for_env "${INACTIVE}")

log "Current active: ${ACTIVE} | Deploying to: ${INACTIVE} (port ${INACTIVE_PORT})"

# Step 1: Stop inactive environment (clean slate)
log "--- Step 1: Preparing inactive slot ---"
stop_app "${INACTIVE}" || log "Stop ${INACTIVE} failed or was already stopped — continuing"

# Step 2: Deploy new version to inactive slot
# You provision the new version here — adapt to your stack.
# For systemd: update /etc/app-${INACTIVE}.env with APP_VERSION=${DEPLOY_VERSION}
log "--- Step 2: Deploying version ${DEPLOY_VERSION} to ${INACTIVE} ---"
echo "APP_VERSION=${DEPLOY_VERSION}" > "/etc/app-${INACTIVE}.env"
echo "APP_PORT=${INACTIVE_PORT}" >> "/etc/app-${INACTIVE}.env"
start_app "${INACTIVE}" "${INACTIVE_PORT}" "${DEPLOY_VERSION}"

# Step 3: Health check gate — MUST pass before traffic switches
log "--- Step 3: Pre-swap health check ---"
if ! health_check "${INACTIVE_PORT}" "${INACTIVE}"; then
    log "Pre-swap health check FAILED. Aborting — traffic stays on ${ACTIVE}."
    stop_app "${INACTIVE}" || true
    die "Deploy aborted. ${ACTIVE} is still serving traffic."
fi

# Step 4: Atomically switch upstream
log "--- Step 4: Switching upstream ---"
set_upstream "${INACTIVE}"
reload_nginx

# Step 5: Post-swap health check via nginx (port 80)
log "--- Step 5: Post-swap health check through nginx ---"
# We check via nginx port 80 to confirm the full path works
POST_SWAP_URL="${HEALTH_CHECK_URL}${HEALTH_CHECK_PATH}"
POST_SWAP_CODE=$(curl -sf -o /dev/null -w "%{http_code}" --max-time 5 "${POST_SWAP_URL}" 2>/dev/null || echo "000")

if [[ "$POST_SWAP_CODE" != "200" ]]; then
    log "Post-swap health check FAILED (HTTP ${POST_SWAP_CODE}). REVERTING to ${ACTIVE}."
    set_upstream "${ACTIVE}"
    reload_nginx
    stop_app "${INACTIVE}" || true
    die "Deploy REVERTED. ${ACTIVE} is serving traffic again."
fi

log "--- Step 6: Deploy successful ---"
log "Traffic is now on ${INACTIVE} (version ${DEPLOY_VERSION})"
log "Leaving ${ACTIVE} running as a hot standby for fast rollback"
log ""
log "To roll back: ./blue-green-deploy.sh --rollback"
log "To decommission old slot: systemctl stop app-${ACTIVE}"
```

### Reload vs Restart — Why It Matters

`nginx -s reload` (SIGHUP) is the correct mechanism for upstream swaps. Here is what actually happens:

1. The master process receives SIGHUP.
2. The master validates the new configuration.
3. The master forks new worker processes with the new config.
4. Old workers are told to stop accepting new connections.
5. Old workers finish in-flight requests and exit.
6. At no point is there a gap in service.

`nginx -s restart` (or `systemctl restart nginx`) kills all workers immediately. In-flight requests are dropped. Never use restart for config changes in production. The only time to use restart is when nginx itself fails to start or you need to reload the binary (e.g., after an nginx upgrade).

Test your config before every reload: `nginx -t && nginx -s reload`. If `nginx -t` fails, the reload is aborted and the old config remains active — this is the safety net.

---

## Rolling Deployment

Rolling deployment replaces backends one at a time. You remove a backend from the pool, drain it, update it, health-check it, and return it to the pool before moving to the next one. Unlike blue-green, you don't need double the resources — but you do need enough capacity in the remaining backends to handle full traffic during the update.

**When to use it:** When you have 3+ backends and can't afford to double resources. When deploys are frequent and the risk per deploy is low. Works well with container orchestration.

**Mandatory prerequisite:** Your backends must be stateless (or session-sticky with external session storage). If backend A gets taken down and user sessions were sticky to it, those users get a 500. Fix your session handling first.

**Capacity planning:** If you have N backends and remove 1 for update, the remaining N-1 must handle 100% of traffic comfortably. For N=2 this means 1 backend must handle everything — which is usually fine if traffic is moderate, but validate your P95 latency under that load before you commit to rolling.

### Nginx Dynamic Upstream Manipulation

For rolling deploys, you need to temporarily remove a backend from the upstream without reloading nginx entirely. The cleanest approach is to use the `upstream_hash` or mark backends as `down` via a separate upstream config file per backend.

```nginx
# /etc/nginx/conf.d/app-upstream.conf
# This file is regenerated by the deploy script

upstream app_backend {
    # Backends are added/removed here during rolling deploy
    # include directives let us manage each server line separately
    server 127.0.0.1:3001 max_fails=3 fail_timeout=30s;  # backend-1
    server 127.0.0.1:3002 max_fails=3 fail_timeout=30s;  # backend-2
    server 127.0.0.1:3003 max_fails=3 fail_timeout=30s;  # backend-3
    keepalive 64;
}

server {
    listen 80;
    server_name app.example.com;

    location / {
        proxy_pass http://app_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_connect_timeout 5s;
        proxy_read_timeout 60s;
        proxy_next_upstream error timeout http_502 http_503;
    }

    location /health {
        proxy_pass http://app_backend/health;
        access_log off;
    }
}
```

### Rolling Deploy Script

```bash
#!/usr/bin/env bash
# /opt/deploy/rolling-deploy.sh
# Usage: ./rolling-deploy.sh <version>
#
# Backend definitions: BACKENDS is an array of "host:port:service_name" tuples.
# Adapt to your infrastructure — these could be remote SSH hosts, local ports, etc.

set -euo pipefail

DEPLOY_VERSION="${1:?Usage: $0 <version>}"
NGINX_UPSTREAM_TEMPLATE="/etc/nginx/templates/app-upstream.conf.tmpl"
NGINX_UPSTREAM_CONF="/etc/nginx/conf.d/app-upstream.conf"
HEALTH_CHECK_RETRIES=15
HEALTH_CHECK_INTERVAL=3
DRAIN_WAIT=10  # seconds to wait after removing backend before updating it
LOG_PREFIX="[rolling]"

# Backend list: "local_port:service_name"
# For remote hosts, adapt to "host:port:ssh_target"
declare -a BACKENDS=(
    "3001:app-backend-1"
    "3002:app-backend-2"
    "3003:app-backend-3"
)

log() { echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) ${LOG_PREFIX} $*"; }
die() { log "FATAL: $*"; exit 1; }

health_check_backend() {
    local port="$1"
    local label="$2"
    local attempt=0

    while [[ $attempt -lt $HEALTH_CHECK_RETRIES ]]; do
        attempt=$((attempt + 1))
        local code
        code=$(curl -sf -o /dev/null -w "%{http_code}" --max-time 5 \
            "http://127.0.0.1:${port}/health" 2>/dev/null || echo "000")

        if [[ "$code" == "200" ]]; then
            log "  Health check OK for ${label} (attempt ${attempt})"
            return 0
        fi
        log "  Health check attempt ${attempt}/${HEALTH_CHECK_RETRIES} for ${label}: HTTP ${code}"
        sleep "${HEALTH_CHECK_INTERVAL}"
    done

    return 1
}

generate_upstream_conf() {
    # Generate nginx upstream block from currently active backends
    local -a active_ports=("$@")
    {
        echo "upstream app_backend {"
        for port in "${active_ports[@]}"; do
            echo "    server 127.0.0.1:${port} max_fails=3 fail_timeout=30s;"
        done
        echo "    keepalive 64;"
        echo "}"
    } > "${NGINX_UPSTREAM_CONF}"
}

reload_nginx_safe() {
    nginx -t || die "nginx config test failed"
    nginx -s reload
    sleep 1
    log "nginx reloaded"
}

deploy_backend() {
    local port="$1"
    local service="$2"
    local version="$3"
    log "  Deploying version ${version} to ${service} on port ${port}"
    # Adapt this block to your process manager:
    echo "APP_VERSION=${version}" > "/etc/app-${service}.env"
    echo "APP_PORT=${port}" >> "/etc/app-${service}.env"
    systemctl stop "${service}" 2>/dev/null || true
    systemctl start "${service}"
}

# --- Main ---

log "=== Rolling Deploy: version=${DEPLOY_VERSION} | backends=${#BACKENDS[@]} ==="

# Parse backends into arrays
declare -a ALL_PORTS=()
declare -a ALL_SERVICES=()
for backend in "${BACKENDS[@]}"; do
    IFS=':' read -r port service <<< "${backend}"
    ALL_PORTS+=("${port}")
    ALL_SERVICES+=("${service}")
done

TOTAL=${#BACKENDS[@]}
FAILED_BACKENDS=()

for i in "${!BACKENDS[@]}"; do
    port="${ALL_PORTS[$i]}"
    service="${ALL_SERVICES[$i]}"
    backend_num=$((i + 1))

    log "--- Backend ${backend_num}/${TOTAL}: ${service} (port ${port}) ---"

    # Build list of ports EXCLUDING current backend
    declare -a remaining_ports=()
    for j in "${!ALL_PORTS[@]}"; do
        if [[ "$j" != "$i" ]]; then
            remaining_ports+=("${ALL_PORTS[$j]}")
        fi
    done

    # Step 1: Remove from upstream pool
    log "  Removing ${service} from nginx upstream pool"
    generate_upstream_conf "${remaining_ports[@]}"
    reload_nginx_safe

    # Step 2: Drain in-flight connections
    log "  Draining connections (${DRAIN_WAIT}s)"
    sleep "${DRAIN_WAIT}"

    # Step 3: Deploy new version
    deploy_backend "${port}" "${service}" "${DEPLOY_VERSION}"

    # Step 4: Health check the updated backend (direct, not through nginx)
    log "  Health checking ${service} directly on port ${port}"
    if ! health_check_backend "${port}" "${service}"; then
        log "  Health check FAILED for ${service}. Skipping this backend."
        FAILED_BACKENDS+=("${service}")
        # Re-add old backend with backup version? Or skip entirely?
        # Policy decision: we skip failed backends and continue with the rest.
        # The backend stays out of the pool. Alert on this condition.
        log "  WARNING: ${service} is NOT in the upstream pool. Investigate immediately."
        continue
    fi

    # Step 5: Return to upstream pool
    log "  Adding ${service} back to nginx upstream pool"
    # Add back all ports up to and including current
    declare -a pool_ports=()
    for j in "${!ALL_PORTS[@]}"; do
        if [[ "$j" -le "$i" ]] || [[ "${#FAILED_BACKENDS[@]}" -eq 0 ]]; then
            pool_ports+=("${ALL_PORTS[$j]}")
        fi
    done
    # Simpler: just add all non-failed backends back
    declare -a final_pool=()
    for j in "${!ALL_PORTS[@]}"; do
        svc="${ALL_SERVICES[$j]}"
        is_failed=0
        for failed in "${FAILED_BACKENDS[@]}"; do
            if [[ "$failed" == "$svc" ]]; then is_failed=1; break; fi
        done
        if [[ $is_failed -eq 0 ]]; then
            final_pool+=("${ALL_PORTS[$j]}")
        fi
    done
    generate_upstream_conf "${final_pool[@]}"
    reload_nginx_safe

    log "  ${service} is updated and back in rotation"
done

# --- Final report ---
log ""
log "=== Rolling Deploy Complete ==="
log "Version ${DEPLOY_VERSION} deployed to $((TOTAL - ${#FAILED_BACKENDS[@]}))/${TOTAL} backends"

if [[ ${#FAILED_BACKENDS[@]} -gt 0 ]]; then
    log "FAILED backends (out of pool): ${FAILED_BACKENDS[*]}"
    log "ACTION REQUIRED: Investigate failed backends and redeploy or roll back manually"
    exit 1
fi

log "All backends healthy and in rotation."
```

---

## Canary Deployment

Canary deploys let you route a small percentage of real traffic to a new version before committing fully. This is the right tool when you're uncertain about correctness, performance, or compatibility under production load — which is most non-trivial changes.

**When to use it:** New features that could have unexpected behavior at scale. Performance-sensitive changes (e.g., new query path). Third-party integrations you haven't load-tested. Any time you want to observe real user impact before full rollout.

**The critical discipline:** Canary without monitoring is theater. Define your success metrics before you start the deploy. Know exactly what "healthy" looks like and automate the check. If you're manually eyeballing dashboards, you'll miss the subtle degradations that matter.

### Nginx `split_clients` Configuration

`split_clients` uses Nginx's consistent hashing to deterministically route requests to upstreams based on a variable. Using `$remote_addr` gives per-IP consistency (a user always hits the same backend). Using `$request_id` gives pure statistical randomness per-request.

**Opinion:** Use `$remote_addr$http_user_agent` for canary. This gives consistent per-user experience without requiring session tracking. A user won't flip between versions mid-session.

```nginx
# /etc/nginx/conf.d/app-canary.conf

upstream app_stable {
    server 127.0.0.1:3001 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:3002 max_fails=3 fail_timeout=30s;
    keepalive 32;
}

upstream app_canary {
    server 127.0.0.1:3010 max_fails=3 fail_timeout=30s;
    keepalive 16;
}

# split_clients hashes the first argument and maps it to a variable.
# Percentages must sum to 100% (or use * as the final catch-all).
# This configuration sends 5% to canary, 95% to stable.
split_clients "${remote_addr}${http_user_agent}" $app_upstream {
    5%      app_canary;
    *       app_stable;
}

server {
    listen 80;
    server_name app.example.com;

    location / {
        proxy_pass http://$app_upstream;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Canary $app_upstream;  # Log which upstream served this request
        proxy_connect_timeout 5s;
        proxy_read_timeout 60s;
    }

    location /health {
        proxy_pass http://app_stable/health;
        access_log off;
    }
}
```

To advance the canary from 5% to 25%, edit the `split_clients` block and reload nginx:

```nginx
# 25% canary phase
split_clients "${remote_addr}${http_user_agent}" $app_upstream {
    25%     app_canary;
    *       app_stable;
}
```

```nginx
# 50% canary phase
split_clients "${remote_addr}${http_user_agent}" $app_upstream {
    50%     app_canary;
    *       app_stable;
}
```

```nginx
# 100% — canary IS the new stable (decommission old stable)
split_clients "${remote_addr}${http_user_agent}" $app_upstream {
    *       app_canary;
}
```

### Canary Progression Script

```bash
#!/usr/bin/env bash
# /opt/deploy/canary-advance.sh
# Usage: ./canary-advance.sh <percentage>
# Valid values: 5, 25, 50, 100

set -euo pipefail

TARGET_PCT="${1:?Usage: $0 <percentage> (5|25|50|100)}"
NGINX_CANARY_CONF="/etc/nginx/conf.d/app-canary.conf"
LOG_PREFIX="[canary]"

log() { echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) ${LOG_PREFIX} $*"; }
die() { log "FATAL: $*"; exit 1; }

case "${TARGET_PCT}" in
    5|25|50)
        STABLE_PCT=$((100 - TARGET_PCT))
        SPLIT_BLOCK="    ${TARGET_PCT}%     app_canary;\n    *       app_stable;"
        ;;
    100)
        SPLIT_BLOCK="    *       app_canary;"
        ;;
    0)
        SPLIT_BLOCK="    *       app_stable;"
        ;;
    *)
        die "Invalid percentage: ${TARGET_PCT}. Use 0, 5, 25, 50, or 100."
        ;;
esac

log "Advancing canary to ${TARGET_PCT}%"

# Rewrite the split_clients block in the config
python3 - "${NGINX_CANARY_CONF}" "${SPLIT_BLOCK}" <<'PYEOF'
import sys, re

conf_file = sys.argv[1]
new_block_content = sys.argv[2]

with open(conf_file) as f:
    content = f.read()

# Replace everything between the split_clients { ... } braces
new_content = re.sub(
    r'(split_clients[^{]+\{)[^}]+(\})',
    lambda m: m.group(1) + '\n' + new_block_content + '\n' + m.group(2),
    content
)

with open(conf_file, 'w') as f:
    f.write(new_content)

print(f"Updated {conf_file}")
PYEOF

nginx -t || die "nginx config test failed — config not changed"
nginx -s reload
sleep 1

log "Canary is now at ${TARGET_PCT}%"
log "Monitor error rates and latency for at least 10 minutes before advancing further"
```

### What to Monitor During a Canary

This is where most teams fail. They deploy a canary and watch CPU. That's not enough.

**Error rate by upstream.** Your nginx logs must include `$app_upstream`. Parse logs with a stream processor (filebeat, vector, fluentd) and track 5xx rate per upstream separately. A 2x increase in error rate on canary is a hard stop.

```bash
# Quick error rate comparison from nginx access log
# Assumes log format includes upstream name as last field or in X-Canary header
awk '{
    if ($0 ~ /app_canary/) { canary_total++; if ($9 >= 500) canary_err++ }
    if ($0 ~ /app_stable/) { stable_total++; if ($9 >= 500) stable_err++ }
}
END {
    printf "Canary:  %d/%d errors (%.2f%%)\n", canary_err, canary_total, (canary_err/canary_total)*100
    printf "Stable:  %d/%d errors (%.2f%%)\n", stable_err, stable_total, (stable_err/stable_total)*100
}' /var/log/nginx/access.log
```

**P95/P99 latency per upstream.** Error rate can look fine while latency degrades. If canary P99 is 3x stable P99, stop.

**Business metrics.** For revenue-generating flows: track conversion rate, checkout completions, or whatever your critical KPI is. A 5% canary with a 20% drop in conversions is a 1% overall revenue hit — noticeable and bad. This requires your analytics to be nearly real-time.

**Automatic abort triggers:** Set up alerting rules before you start. If any of the following are true, automatically revert to 0% canary:
- Canary 5xx rate > stable 5xx rate × 2
- Canary P95 latency > stable P95 latency × 1.5
- Any canary 5xx rate > 1% absolute

**Canary soak times (minimum):**
- 5% → wait 10 minutes minimum
- 25% → wait 20 minutes minimum
- 50% → wait 30 minutes minimum
- Never advance during off-peak hours if you need statistical significance

---

## Rsync + Symlink Deploy (Capistrano-Style)

This is the oldest reliable zero-downtime deploy pattern for traditional servers. It predates containers and still works better than containers for many use cases. The core idea: rsync a new release into a timestamped directory, prepare it fully (migrations, builds), then atomically swap a symlink to make it current.

The symlink swap is atomic at the filesystem level — processes reading `current` will either see the old release or the new one, never a partially-written state.

### Directory Structure

```
/var/www/app/
├── current -> releases/20240315-143022/    ← symlink to active release
├── releases/
│   ├── 20240315-143022/                    ← current (most recent)
│   │   ├── app/
│   │   ├── node_modules/                   ← linked from shared or rebuilt
│   │   ├── .env -> /var/www/app/shared/.env
│   │   └── uploads -> /var/www/app/shared/uploads/
│   ├── 20240314-091500/                    ← previous (kept for rollback)
│   ├── 20240313-162200/                    ← older
│   └── 20240312-120000/                    ← will be pruned
├── shared/
│   ├── .env                                ← environment config (never in git)
│   ├── uploads/                            ← user-uploaded files
│   ├── logs/                               ← application logs
│   └── node_modules/                       ← optional: shared node_modules cache
└── repo/                                   ← bare git clone (optional, for git-based deploys)
```

**Why keep old releases:** You can roll back in under 30 seconds by relinking `current`. Keep at least 5 releases. The disk cost is minimal for most apps.

**Shared directories:** Anything that must persist across deploys lives in `shared/`. The deploy script symlinks these into each release. Never put `uploads/` or `.env` inside a release directory — you'll regret it.

### Full Deploy Script

```bash
#!/usr/bin/env bash
# /opt/deploy/symlink-deploy.sh
# Usage: ./symlink-deploy.sh <git_ref_or_artifact_path>
#
# Assumptions:
#   - App root: /var/www/app
#   - App runs under: systemctl app (or pm2 — adapt RELOAD_CMD)
#   - Node.js app, but adapt the build steps for your stack
#   - Deploy user has sudo for systemctl reload app

set -euo pipefail

APP_ROOT="/var/www/app"
RELEASES_DIR="${APP_ROOT}/releases"
SHARED_DIR="${APP_ROOT}/shared"
CURRENT_LINK="${APP_ROOT}/current"
KEEP_RELEASES=5
APP_HEALTH_URL="http://127.0.0.1:3000/health"
HEALTH_CHECK_RETRIES=20
HEALTH_CHECK_INTERVAL=3
LOG_PREFIX="[symlink-deploy]"

# Source artifact or git ref
SOURCE="${1:?Usage: $0 <source_dir_or_git_ref>}"
RELEASE_TIMESTAMP=$(date -u +%Y%m%d-%H%M%S)
RELEASE_DIR="${RELEASES_DIR}/${RELEASE_TIMESTAMP}"

log()  { echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) ${LOG_PREFIX} $*"; }
die()  { log "FATAL: $*"; exit 1; }

# Shared directories to symlink into each release
SHARED_DIRS=("uploads" "logs")
SHARED_FILES=(".env")

# --- Health check ---
health_check() {
    local label="${1:-app}"
    local attempt=0
    log "Health checking ${label} at ${APP_HEALTH_URL}"
    while [[ $attempt -lt $HEALTH_CHECK_RETRIES ]]; do
        attempt=$((attempt + 1))
        local code
        code=$(curl -sf -o /dev/null -w "%{http_code}" --max-time 5 "${APP_HEALTH_URL}" 2>/dev/null || echo "000")
        if [[ "$code" == "200" ]]; then
            log "Health check passed (attempt ${attempt})"
            return 0
        fi
        log "Attempt ${attempt}/${HEALTH_CHECK_RETRIES}: HTTP ${code}"
        sleep "${HEALTH_CHECK_INTERVAL}"
    done
    return 1
}

# --- Rollback ---
revert_symlink() {
    local previous_release="$1"
    if [[ -z "$previous_release" ]]; then
        log "No previous release to revert to."
        return 1
    fi
    log "REVERTING: symlinking current -> ${previous_release}"
    ln -sfn "${previous_release}" "${CURRENT_LINK}"
    reload_app
}

reload_app() {
    log "Reloading application"
    # Use reload (SIGUSR2 for PM2, systemctl reload for systemd with ExecReload)
    # This sends signal to the process manager to gracefully restart workers
    sudo systemctl reload app 2>/dev/null || sudo systemctl restart app
}

# --- Setup ---
mkdir -p "${RELEASES_DIR}" "${SHARED_DIR}"

# Ensure shared dirs and files exist
for dir in "${SHARED_DIRS[@]}"; do
    mkdir -p "${SHARED_DIR}/${dir}"
done
for file in "${SHARED_FILES[@]}"; do
    if [[ ! -f "${SHARED_DIR}/${file}" ]]; then
        die "Shared file ${SHARED_DIR}/${file} does not exist. Create it before deploying."
    fi
done

# --- Step 1: Capture previous release for rollback ---
PREVIOUS_RELEASE=""
if [[ -L "${CURRENT_LINK}" ]]; then
    PREVIOUS_RELEASE=$(readlink -f "${CURRENT_LINK}")
    log "Previous release: ${PREVIOUS_RELEASE}"
fi

# --- Step 2: Rsync new release ---
log "--- Step 1: Syncing release to ${RELEASE_DIR} ---"
mkdir -p "${RELEASE_DIR}"

# If SOURCE is a directory (pre-built artifact), rsync it.
# If SOURCE is a git ref, you'd do a git archive | tar -x here instead.
rsync -az \
    --exclude='.git' \
    --exclude='node_modules' \
    --exclude='.env' \
    --exclude='uploads' \
    --exclude='logs' \
    --delete \
    "${SOURCE}/" "${RELEASE_DIR}/"

log "Rsync complete"

# --- Step 3: Link shared resources ---
log "--- Step 2: Linking shared directories and files ---"
for dir in "${SHARED_DIRS[@]}"; do
    rm -rf "${RELEASE_DIR:?}/${dir}"
    ln -sf "${SHARED_DIR}/${dir}" "${RELEASE_DIR}/${dir}"
    log "  Linked ${dir}"
done
for file in "${SHARED_FILES[@]}"; do
    rm -f "${RELEASE_DIR:?}/${file}"
    ln -sf "${SHARED_DIR}/${file}" "${RELEASE_DIR}/${file}"
    log "  Linked ${file}"
done

# --- Step 4: Install dependencies ---
log "--- Step 3: Installing dependencies ---"
# Use --frozen-lockfile / --ci to guarantee exact versions
# Never run npm install without a lockfile in production
cd "${RELEASE_DIR}"
npm ci --production --silent
log "npm ci complete"

# --- Step 5: Build ---
log "--- Step 4: Building application ---"
npm run build
log "Build complete"

# --- Step 6: Database migrations ---
log "--- Step 5: Running database migrations ---"
# Migrations MUST be backward-compatible with the running version.
# See the Expand/Contract section of this document.
# Run migrations before swapping the symlink — the new schema must work
# with the OLD code still running.
NODE_ENV=production node "${RELEASE_DIR}/scripts/migrate.js" || die "Migrations failed"
log "Migrations complete"

# --- Step 7: Symlink swap ---
log "--- Step 6: Swapping symlink ---"
# ln -sfn: -s = symbolic, -f = force (overwrite existing), -n = treat CURRENT_LINK as a file if it's a symlink to a dir
# This is atomic at the filesystem level.
ln -sfn "${RELEASE_DIR}" "${CURRENT_LINK}"
log "current -> ${RELEASE_DIR}"

# --- Step 8: Reload application ---
log "--- Step 7: Reloading application ---"
reload_app

# --- Step 9: Post-deploy health check ---
log "--- Step 8: Post-deploy health check ---"
if ! health_check "post-deploy"; then
    log "POST-DEPLOY HEALTH CHECK FAILED. Initiating auto-rollback."

    if [[ -n "$PREVIOUS_RELEASE" ]]; then
        revert_symlink "${PREVIOUS_RELEASE}"

        log "Waiting for rollback health check..."
        sleep 5
        if health_check "post-rollback"; then
            log "Rollback successful. Previous release is serving traffic."
        else
            die "ROLLBACK HEALTH CHECK ALSO FAILED. Manual intervention required. Previous release: ${PREVIOUS_RELEASE}"
        fi
    else
        die "No previous release to roll back to. Manual intervention required."
    fi

    # Clean up failed release
    rm -rf "${RELEASE_DIR}"
    die "Deploy FAILED and rolled back to ${PREVIOUS_RELEASE}"
fi

# --- Step 10: Cleanup old releases ---
log "--- Step 9: Pruning old releases (keeping ${KEEP_RELEASES}) ---"
ls -1dt "${RELEASES_DIR}"/*/ | tail -n +$((KEEP_RELEASES + 1)) | while read -r old_release; do
    log "  Removing old release: ${old_release}"
    rm -rf "${old_release}"
done

log ""
log "=== Deploy complete: ${RELEASE_TIMESTAMP} ==="
log "Release: ${RELEASE_DIR}"
log "Previous: ${PREVIOUS_RELEASE:-none}"
log "To roll back: ln -sfn ${PREVIOUS_RELEASE} ${CURRENT_LINK} && systemctl reload app"
```

### Manual Rollback (Symlink)

```bash
# List releases, newest first
ls -1dt /var/www/app/releases/*/

# Roll back to previous release
PREVIOUS=$(ls -1dt /var/www/app/releases/*/ | sed -n '2p')
ln -sfn "${PREVIOUS}" /var/www/app/current
sudo systemctl reload app

# Verify
curl -sf http://127.0.0.1:3000/health && echo "Rollback successful"
```

---

## GitHub Actions SSH Deploy Job

A production-grade GitHub Actions workflow for SSH-based deploys. Opinionated choices explained inline.

**Concurrency group:** Prevents two deploys running simultaneously. `cancel-in-progress: false` is intentional — we do NOT cancel an in-progress deploy. Canceling mid-deploy leaves the server in an unknown state. Let it finish, then fail the queued one. If you want queueing behavior, use `cancel-in-progress: false`. If you're OK losing queued deploys, set it to `true`.

**known_hosts handling:** Never use `StrictHostKeyChecking=no`. That defeats the entire purpose of SSH host verification. Pre-populate known_hosts from a secret that contains the server's host key fingerprint.

**Health poll:** After deploy, poll the health endpoint from GitHub Actions. This catches cases where the deploy script succeeds but the app fails to serve traffic through the load balancer.

**Slack notification:** Only on failure. Success notifications are noise. Failure notifications are actionable. If you need success notifications, add them as a separate step that doesn't block on failure.

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches:
      - main
  workflow_dispatch:
    inputs:
      environment:
        description: "Target environment"
        required: true
        default: "production"
        type: choice
        options:
          - production
          - staging

# IMPORTANT: cancel-in-progress is false.
# Never cancel an in-progress deploy — it leaves servers in partial state.
# Queue the new deploy and let the current one finish.
concurrency:
  group: deploy-${{ github.event.inputs.environment || 'production' }}
  cancel-in-progress: false

env:
  APP_HOST: ${{ secrets.DEPLOY_HOST }}
  DEPLOY_USER: deploy
  APP_ROOT: /var/www/app
  HEALTH_URL: https://app.example.com/health
  ARTIFACT_DIR: /tmp/deploy-artifact-${{ github.sha }}

jobs:
  build:
    name: Build Artifact
    runs-on: ubuntu-latest
    outputs:
      artifact-name: ${{ steps.artifact.outputs.name }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version-file: ".nvmrc"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Build
        run: npm run build
        env:
          NODE_ENV: production

      - name: Package artifact
        id: artifact
        run: |
          ARTIFACT_NAME="app-${{ github.sha }}.tar.gz"
          tar -czf "${ARTIFACT_NAME}" \
            --exclude='.git' \
            --exclude='node_modules' \
            --exclude='*.test.*' \
            --exclude='coverage' \
            dist/ package.json package-lock.json scripts/
          echo "name=${ARTIFACT_NAME}" >> "$GITHUB_OUTPUT"

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: deploy-artifact
          path: ${{ steps.artifact.outputs.name }}
          retention-days: 7
          if-no-files-found: error

  deploy:
    name: Deploy to ${{ github.event.inputs.environment || 'production' }}
    runs-on: ubuntu-latest
    needs: build
    environment: ${{ github.event.inputs.environment || 'production' }}

    steps:
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: deploy-artifact

      - name: Configure SSH
        run: |
          mkdir -p ~/.ssh
          chmod 700 ~/.ssh

          # Write the private key from secrets
          echo "${{ secrets.DEPLOY_SSH_KEY }}" > ~/.ssh/deploy_key
          chmod 600 ~/.ssh/deploy_key

          # Pre-populate known_hosts with the server's host key.
          # This key was captured once by running:
          #   ssh-keyscan -H app.example.com
          # and stored as a secret. Never use StrictHostKeyChecking=no.
          echo "${{ secrets.DEPLOY_HOST_KEY }}" >> ~/.ssh/known_hosts
          chmod 644 ~/.ssh/known_hosts

          # Write SSH config for convenience
          cat >> ~/.ssh/config <<EOF
          Host deploy-target
            HostName ${{ env.APP_HOST }}
            User ${{ env.DEPLOY_USER }}
            IdentityFile ~/.ssh/deploy_key
            StrictHostKeyChecking yes
            ConnectTimeout 10
            ServerAliveInterval 30
            ServerAliveCountMax 3
          EOF
          chmod 600 ~/.ssh/config

      - name: Verify SSH connectivity
        run: ssh deploy-target "echo 'SSH connection successful' && whoami"
        timeout-minutes: 1

      - name: Upload artifact to server
        run: |
          ARTIFACT="${{ needs.build.outputs.artifact-name }}"
          scp "${ARTIFACT}" "deploy-target:/tmp/${ARTIFACT}"
          # Extract to a temp location for the deploy script
          ssh deploy-target "mkdir -p ${{ env.ARTIFACT_DIR }} && tar -xzf /tmp/${ARTIFACT} -C ${{ env.ARTIFACT_DIR }} && rm /tmp/${ARTIFACT}"

      - name: Run deploy script
        id: deploy
        run: |
          ssh deploy-target "
            set -euo pipefail
            /opt/deploy/symlink-deploy.sh '${{ env.ARTIFACT_DIR }}'
          "
        timeout-minutes: 15

      - name: Poll health endpoint
        # This step runs AFTER the deploy script confirms success.
        # We poll from GitHub Actions to confirm the app is reachable externally —
        # the deploy script only checks from localhost.
        run: |
          echo "Polling ${{ env.HEALTH_URL }} for up to 3 minutes..."
          ATTEMPTS=0
          MAX_ATTEMPTS=36  # 36 × 5s = 3 minutes
          until curl -sf --max-time 10 "${{ env.HEALTH_URL }}" > /dev/null; do
            ATTEMPTS=$((ATTEMPTS + 1))
            if [[ $ATTEMPTS -ge $MAX_ATTEMPTS ]]; then
              echo "Health poll timed out after $((MAX_ATTEMPTS * 5)) seconds"
              exit 1
            fi
            echo "Attempt ${ATTEMPTS}/${MAX_ATTEMPTS} — waiting 5s..."
            sleep 5
          done
          echo "Application is healthy at ${{ env.HEALTH_URL }}"
        timeout-minutes: 4

      - name: Clean up artifact on server
        if: always()
        run: |
          ssh deploy-target "rm -rf '${{ env.ARTIFACT_DIR }}'" || true

      - name: Notify Slack on failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": ":red_circle: *Deploy FAILED*",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": ":red_circle: *Deploy to ${{ github.event.inputs.environment || 'production' }} FAILED*\n*Commit:* `${{ github.sha }}`\n*Branch:* `${{ github.ref_name }}`\n*Triggered by:* ${{ github.actor }}\n*Workflow run:* <${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Run>"
                  }
                },
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "The previous release is still active. Check the workflow run for details.\n*Rollback command:*\n```cd /var/www/app && ln -sfn $(ls -1dt releases/*/ | sed -n '2p') current && systemctl reload app```"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_DEPLOY_WEBHOOK }}
          SLACK_WEBHOOK_TYPE: INCOMING_WEBHOOK

  post-deploy-verification:
    name: Post-Deploy Smoke Tests
    runs-on: ubuntu-latest
    needs: deploy
    if: success()

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Run smoke tests against production
        run: |
          # Run a minimal set of critical-path smoke tests.
          # These should be fast (< 60s) and non-destructive.
          # They run against the live production environment.
          npm ci --ignore-scripts
          SMOKE_TARGET=${{ env.HEALTH_URL }} npm run test:smoke
        timeout-minutes: 5

      - name: Notify Slack on smoke test failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": ":warning: *Smoke tests FAILED after deploy*",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": ":warning: *Smoke tests failed after deploy to ${{ github.event.inputs.environment || 'production' }}*\n*Commit:* `${{ github.sha }}`\nThe deploy completed but smoke tests failed. Manual investigation required."
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_DEPLOY_WEBHOOK }}
          SLACK_WEBHOOK_TYPE: INCOMING_WEBHOOK
```

### Required GitHub Secrets

```
DEPLOY_HOST        — IP or hostname of the production server
DEPLOY_SSH_KEY     — Private SSH key for the deploy user (PEM format, no passphrase)
DEPLOY_HOST_KEY    — Server's SSH host key (from ssh-keyscan -H your-server)
SLACK_DEPLOY_WEBHOOK — Slack incoming webhook URL for failure notifications
```

To capture the host key:

```bash
ssh-keyscan -H your-server.example.com 2>/dev/null
# Copy the output and store as DEPLOY_HOST_KEY secret
```

---

## Database Migrations: Expand/Contract Pattern

This is the pattern that separates teams that have production database incidents from teams that don't. The core problem: ALTER TABLE on a large table acquires a table lock. Your application blocks. Users see 500s or timeouts. The solution is to never make a schema change that requires a lock while your old code is running.

The expand/contract pattern makes schema changes in multiple backwards-compatible steps, each of which can be deployed independently.

### Why Standard Migrations Break Production

```sql
-- This is dangerous on a table with millions of rows:
ALTER TABLE users ADD COLUMN display_name VARCHAR(255) NOT NULL DEFAULT '';

-- On MySQL/PostgreSQL < 11, this:
-- 1. Acquires an ACCESS EXCLUSIVE lock on the table
-- 2. Rewrites every row in the table
-- 3. Holds the lock the entire time
-- 4. Your application cannot read OR write users during this time
-- On a 50M row table this can take 20+ minutes
```

Even on PostgreSQL 11+ where `ADD COLUMN ... DEFAULT` is instant, non-trivial changes (adding a NOT NULL constraint, adding an index, renaming a column) still require locks or full rewrites.

### The Expand/Contract Phases

**Phase 1 — Expand (additive only):** Add new structures alongside old ones. Nothing is removed. The old code must continue to work unchanged.

**Phase 2 — Migrate:** Deploy code that writes to both old and new structures simultaneously. Background-fill any existing data.

**Phase 3 — Verify:** Confirm new structure has all data and new code is working correctly in production.

**Phase 4 — Contract:** Remove old structures. Only happens after new code is fully deployed and verified.

### Concrete Example: Renaming a Column

**Scenario:** Rename `users.name` to `users.display_name`.

#### Phase 1: Expand

```sql
-- Migration 001_expand_add_display_name.sql
-- Safe: ADD COLUMN is non-blocking (PostgreSQL, MySQL 8+)
-- Add new column as nullable (NOT NULL would require a full table rewrite or check constraint trick)
ALTER TABLE users ADD COLUMN display_name VARCHAR(255);

-- Add an index if needed — use CONCURRENTLY to avoid locking
CREATE INDEX CONCURRENTLY idx_users_display_name ON users(display_name);

-- No data migration yet. display_name is NULL for all existing rows.
```

Deploy this migration. The old code still reads/writes `name`. The new column exists but is unused.

#### Phase 2: Dual-Write Code Deployment

Deploy application code that writes to BOTH columns on every write:

```python
# models/user.py — Dual-write phase
class User(Base):
    __tablename__ = 'users'

    id = Column(Integer, primary_key=True)
    name = Column(String)           # OLD — still read by old code instances
    display_name = Column(String)   # NEW — written by new code

    def save(self, session):
        # Write to both columns during transition
        # Old code reads 'name', new code reads 'display_name'
        self.display_name = self.display_name or self.name
        session.add(self)
        session.commit()

    @property
    def canonical_name(self):
        # New code reads display_name, falls back to name for unmigrated rows
        return self.display_name or self.name
```

```python
# For reads in new code, always prefer display_name:
# SELECT id, COALESCE(display_name, name) AS display_name FROM users WHERE ...
```

After deploying dual-write code, backfill existing rows:

```sql
-- Migration 002_backfill_display_name.sql
-- Run in batches to avoid long-running transactions
-- Use a loop with LIMIT/OFFSET or keyset pagination

DO $$
DECLARE
    batch_size INT := 1000;
    last_id    INT := 0;
    max_id     INT;
    rows_updated INT;
BEGIN
    SELECT MAX(id) INTO max_id FROM users;

    LOOP
        UPDATE users
        SET display_name = name
        WHERE id > last_id
          AND id <= last_id + batch_size
          AND display_name IS NULL
          AND name IS NOT NULL;

        GET DIAGNOSTICS rows_updated = ROW_COUNT;
        EXIT WHEN last_id >= max_id;

        last_id := last_id + batch_size;

        -- Yield to other transactions between batches
        PERFORM pg_sleep(0.05);
    END LOOP;
END $$;
```

#### Phase 3: Verify

Before contracting, verify data completeness:

```sql
-- Verify no rows have NULL display_name where name is not null
SELECT COUNT(*) FROM users WHERE name IS NOT NULL AND display_name IS NULL;
-- Must be 0

-- Spot-check data consistency
SELECT id, name, display_name
FROM users
WHERE name IS DISTINCT FROM display_name
LIMIT 20;
-- Investigate any discrepancies

-- Check null counts
SELECT
    COUNT(*) FILTER (WHERE name IS NULL) AS null_name,
    COUNT(*) FILTER (WHERE display_name IS NULL) AS null_display_name,
    COUNT(*) AS total
FROM users;
```

Also verify in application monitoring: new code should be reading `display_name` in 100% of cases. No fallbacks to `name` should appear in logs.

#### Phase 4: Contract

After all instances of old code are gone from production (confirmed via deployment), remove the old column:

```sql
-- Migration 003_contract_drop_name.sql
-- Only run this after:
--   1. All app instances use display_name (zero reads from 'name')
--   2. Dual-write has been running long enough to catch all writes
--   3. Backfill is verified complete

-- First, add NOT NULL constraint if required
-- Use a CHECK CONSTRAINT trick (PostgreSQL) for zero-lock NOT NULL enforcement:
ALTER TABLE users
    ADD CONSTRAINT users_display_name_not_null
    CHECK (display_name IS NOT NULL)
    NOT VALID;  -- NOT VALID skips existing row check (non-blocking)

-- Then validate in the background (takes a share lock, not exclusive):
ALTER TABLE users VALIDATE CONSTRAINT users_display_name_not_null;

-- Now make it truly NOT NULL (fast, since constraint is validated):
ALTER TABLE users ALTER COLUMN display_name SET NOT NULL;

-- Drop the now-redundant constraint:
ALTER TABLE users DROP CONSTRAINT users_display_name_not_null;

-- Drop the old column (fast — just removes catalog entry in Postgres):
ALTER TABLE users DROP COLUMN name;
```

### Why This Avoids Locking

- **ADD COLUMN NULL:** In PostgreSQL, adding a nullable column without a default is instant — just a catalog update, no row rewrite.
- **ADD COLUMN with DEFAULT (PostgreSQL 11+):** Also instant for most types.
- **NOT VALID constraints:** Skip validating existing rows. Validation happens separately with a weaker lock.
- **Batched backfill:** Each batch is a small transaction. No row is locked for more than milliseconds. Other queries proceed normally between batches.
- **DROP COLUMN:** In PostgreSQL, dropping a column is fast — it just marks the column as dropped in `pg_attribute`. The space is reclaimed at next VACUUM.

### Zero-Lock Checklist

Before running any migration in production, answer these questions:

- Does this ALTER TABLE add a new NOT NULL column without a default? **Stop.** Use a nullable column, backfill, then add the constraint.
- Does this ALTER TABLE add an index? **Use CONCURRENTLY** (PostgreSQL) or `ALGORITHM=INPLACE, LOCK=NONE` (MySQL 8).
- Does this ALTER TABLE change a column's type? **This is almost always a full table rewrite.** Use a new column instead.
- Is this migration wrapped in a transaction with a long backfill? **Don't.** Backfill outside the transaction.
- Is this migration running during peak traffic hours? **Don't.** Even "safe" migrations take table statistics locks. Run during low-traffic windows.

---

## Rollback Runbook Template

A rollback runbook is not documentation — it is an operational procedure that must work under stress, with sleep-deprived engineers, in a noisy incident bridge. Write it to be executed, not read.

Fill this template in for every application before its first production deploy. Store it in your runbook wiki or as a checked-in file. Review and test it quarterly.

---

### Application: [APP NAME]

**Last reviewed:** YYYY-MM-DD  
**Reviewed by:** [Name]  
**Rollback tested on staging:** Yes/No — [Date]

---

### Section 1: Who Decides to Roll Back

**Rollback authority:** Any on-call engineer may initiate rollback without waiting for approval if any of the following are true:
- Error rate (5xx) exceeds 1% for more than 2 minutes post-deploy
- P95 latency exceeds [YOUR THRESHOLD]ms for more than 5 minutes
- Any critical business flow (login, checkout, [YOUR CRITICAL PATH]) is failing
- The health endpoint returns non-200

**Do NOT wait for:** Engineering management approval, product manager approval, or a full incident review before rolling back. Roll back first. Discuss later.

**Escalation path if rollback fails:**
1. Page [PRIMARY ON-CALL] via PagerDuty
2. Notify [SLACK CHANNEL]
3. If DB migration is involved, page [DBA ON-CALL]

---

### Section 2: Pre-Rollback Checklist

Before executing rollback, collect this information:

```
[ ] What version is currently deployed? _______________
[ ] What version was deployed before this? _______________
[ ] Were database migrations run? YES / NO
[ ] If YES: are the migrations backward-compatible with the previous version? YES / NO / UNKNOWN
[ ] What time did the incident start? _______________
[ ] What is the observed symptom? _______________
[ ] Is the issue definitely caused by the current deploy? (vs. infrastructure, third-party, traffic spike)
```

**If migrations were run and are NOT backward-compatible:** The rollback is more complex. Do NOT just revert the symlink. Engage the DBA on-call immediately. Proceed to Section 5 (Database Rollback).

---

### Section 3: Standard Application Rollback (No DB Changes)

Execute these steps in order. Each step has an expected outcome. If the outcome doesn't match, stop and escalate.

```bash
# Step 1: Identify the previous release
ls -1dt /var/www/app/releases/*/
# Expected: list of timestamped directories, newest first
# The SECOND one is the previous release

PREVIOUS=$(ls -1dt /var/www/app/releases/*/ | sed -n '2p')
echo "Rolling back to: ${PREVIOUS}"
# Expected: a directory path like /var/www/app/releases/20240314-091500/

# Step 2: Check that the previous release exists and is complete
ls "${PREVIOUS}"
# Expected: should contain app files, not be empty

# Step 3: Swap the symlink
ln -sfn "${PREVIOUS}" /var/www/app/current
readlink -f /var/www/app/current
# Expected: output matches ${PREVIOUS}

# Step 4: Reload the application
sudo systemctl reload app
# Expected: no error output, service remains in "active" state

sudo systemctl status app
# Expected: "active (running)"

# Step 5: Health check
sleep 5  # give workers time to restart
curl -sf http://127.0.0.1:3000/health
# Expected: HTTP 200 with JSON body

# Step 6: Check error rate
# Check your monitoring dashboard for the past 5 minutes.
# Expected: error rate returning to pre-deploy baseline

# Step 7: Verify through load balancer
curl -sf https://app.example.com/health
# Expected: HTTP 200
```

**Time target:** Steps 1-7 should complete in under 5 minutes.

---

### Section 4: Blue-Green Rollback

If using blue-green deployment:

```bash
# Step 1: Check which upstream is currently active
cat /etc/nginx/conf.d/active_upstream.conf
# Expected: proxy_pass http://blue;  OR  proxy_pass http://green;

CURRENT_UPSTREAM=$(grep -oP '(?<=proxy_pass http://)[\w]+(?=;)' /etc/nginx/conf.d/active_upstream.conf)
if [[ "$CURRENT_UPSTREAM" == "blue" ]]; then PREVIOUS="green"; else PREVIOUS="blue"; fi

echo "Rolling back to: ${PREVIOUS}"

# Step 2: Verify previous environment is still running
curl -sf "http://127.0.0.1:$([ "$PREVIOUS" == "blue" ] && echo 3001 || echo 3002)/health"
# Expected: HTTP 200
# If NOT 200: the previous environment is not running. Start it before proceeding.
# sudo systemctl start "app-${PREVIOUS}"

# Step 3: Switch nginx upstream
echo "proxy_pass http://${PREVIOUS};" > /etc/nginx/conf.d/active_upstream.conf
nginx -t && nginx -s reload
# Expected: nginx -t completes with "syntax ok" and "test is successful"

# Step 4: Verify
sleep 2
curl -sf https://app.example.com/health
# Expected: HTTP 200

# Step 5: Confirm in nginx access log which upstream is serving
tail -20 /var/log/nginx/access.log
# Expected: no 5xx responses
```

---

### Section 5: Database Rollback (Escalate First)

Database rollbacks are categorically different from application rollbacks. Attempt these only after confirming with the DBA on-call.

**General rule:** If the expand/contract pattern was followed, you do NOT need to revert the database. You only revert the application. The database schema is backward-compatible with the previous application version.

**If the migration was not backward-compatible (violation of the pattern):**

```bash
# DO NOT execute without DBA sign-off
# 1. Application rollback first (previous steps)
# 2. Assess data written by new code to new columns
# 3. If new columns have no data or data is reversible:
#    Run the reverse migration script (migrations/rollback/XXX_rollback.sql)
# 4. If data was written to new columns and cannot be reversed:
#    You may need to accept the new schema and rewrite application code
#    to be compatible with both old and new schema.
```

---

### Section 6: Post-Rollback Communication

Send this within 5 minutes of completing rollback:

**Slack message template:**

```
:rewind: *Rollback completed* for [APP NAME]

• *Rolled back from:* [new version]
• *Rolled back to:* [previous version]
• *Reason:* [one-sentence description of the issue]
• *Rollback completed at:* [time]
• *Current status:* Application healthy | Error rate normal
• *Postmortem:* Will be filed by [date]

cc: [engineering lead] [product manager]
```

**Incident timeline to document:**

```
[TIME] Deploy started
[TIME] Deploy completed
[TIME] Issue first detected (by whom, via what signal)
[TIME] Rollback decision made (by whom)
[TIME] Rollback started
[TIME] Rollback completed
[TIME] Service confirmed healthy
```

---

### Section 7: Postmortem Requirements

A postmortem MUST be filed within 48 hours for any rollback. The postmortem must answer:

1. What was the root cause of the failure?
2. Why didn't tests catch it?
3. Why wasn't it caught in staging?
4. What monitoring change would have caught it sooner?
5. What code/process change prevents recurrence?
6. Were the rollback procedures adequate, or did anything slow down the response?

Blame-free. The system failed, not the person. The fix is in the system.

---

### Appendix: Quick Reference Card

Print this and keep it near your workstation during deploys.

```
BLUE-GREEN ROLLBACK
  1. cat /etc/nginx/conf.d/active_upstream.conf     # find current
  2. echo "proxy_pass http://[other];" > /etc/nginx/conf.d/active_upstream.conf
  3. nginx -t && nginx -s reload
  4. curl -sf https://app.example.com/health

SYMLINK ROLLBACK
  1. ls -1dt /var/www/app/releases/*/               # find previous
  2. ln -sfn /var/www/app/releases/[PREV]/ /var/www/app/current
  3. sudo systemctl reload app
  4. curl -sf http://127.0.0.1:3000/health

ROLLING ROLLBACK
  For each backend in reverse order:
  1. Remove from nginx pool, reload
  2. Deploy previous version to that backend
  3. Health check
  4. Add back to pool, reload

CANARY ABORT
  1. /opt/deploy/canary-advance.sh 0               # 0% to canary
  2. nginx -t && nginx -s reload
  3. Monitor error rate
```

---

*This document should be reviewed and tested at minimum quarterly. A runbook that has never been tested is a hypothesis, not a procedure.*
