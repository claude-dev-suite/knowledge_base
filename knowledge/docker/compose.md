# Docker Compose Complete Reference

> Official Documentation: https://docs.docker.com/compose/compose-file/

Docker Compose is a tool for defining and running multi-container Docker applications using
a YAML configuration file. This reference covers the complete Compose specification.

---

## Table of Contents

1. [Compose File Structure](#compose-file-structure)
2. [Service Configuration](#service-configuration)
3. [Health Checks](#health-checks)
4. [Restart Policies](#restart-policies)
5. [Networks](#networks)
6. [Volumes](#volumes)
7. [Environment Files](#environment-files)
8. [Profiles](#profiles)
9. [Secrets and Configs](#secrets-and-configs)
10. [Resource Limits](#resource-limits)
11. [CLI Commands Reference](#cli-commands-reference)
12. [Extends and Include](#extends-and-include)
13. [Best Practices and Common Pitfalls](#best-practices-and-common-pitfalls)

---

## Compose File Structure

The Compose file defines services, networks, volumes, configs, and secrets.

```yaml
# Top-level elements of a Compose file
name: my-project                    # Optional: project name (overrides directory name)

services:                           # Required: container definitions
  web:
    image: nginx:alpine
  api:
    build: ./api

networks:                           # Optional: custom network definitions
  frontend:
  backend:

volumes:                            # Optional: named volume definitions
  db-data:
  cache-data:

configs:                            # Optional: configuration files
  app-config:
    file: ./config.json

secrets:                            # Optional: sensitive data
  db-password:
    file: ./secrets/db_password.txt
```

### Version (Deprecated)

The `version` key is now obsolete. Modern Docker Compose (v2+) automatically determines
the schema version. You may still see it in older files:

```yaml
# Legacy - no longer required
version: '3.8'

# Modern - just start with services
services:
  app:
    image: nginx
```

### File Naming Conventions

Docker Compose automatically loads these files in order:
1. `compose.yaml` (preferred)
2. `compose.yml`
3. `docker-compose.yaml`
4. `docker-compose.yml`

Override files are loaded automatically:
- `compose.override.yaml` or `docker-compose.override.yml`

---

## Service Configuration

Services define containers and their configurations.

### Build Configuration

```yaml
services:
  # Simple build from current directory
  app-simple:
    build: .

  # Advanced build configuration
  app-advanced:
    build:
      context: ./app                # Build context directory
      dockerfile: Dockerfile.prod   # Custom Dockerfile name
      args:                         # Build-time arguments
        NODE_ENV: production
        BUILD_DATE: "2024-01-15"
      target: production            # Multi-stage build target
      cache_from:                   # Images to use as cache source
        - myregistry/app:cache
      labels:                       # Labels for the built image
        com.example.version: "1.0"
      network: host                 # Network mode during build
      shm_size: 256mb               # Size of /dev/shm
      extra_hosts:                  # Add host entries during build
        - "host.docker.internal:host-gateway"
      platforms:                    # Multi-platform builds
        - linux/amd64
        - linux/arm64
```

### Image Configuration

```yaml
services:
  # Use pre-built image
  db:
    image: postgres:16-alpine

  # Use image from private registry
  api:
    image: myregistry.io/myorg/api:v1.2.3

  # Build and tag image
  web:
    build: ./web
    image: myregistry.io/myorg/web:latest   # Tags built image
```

### Port Mapping

```yaml
services:
  app:
    ports:
      # Short syntax: HOST:CONTAINER
      - "3000:3000"                 # Map port 3000
      - "8080:80"                   # Map host 8080 to container 80
      - "127.0.0.1:3001:3001"       # Bind to localhost only
      - "6060:6060/udp"             # UDP port

      # Long syntax
      - target: 80                  # Container port
        host_ip: 0.0.0.0            # Host IP to bind
        published: "8080"           # Host port (string for range)
        protocol: tcp               # tcp or udp
        mode: host                  # host or ingress (Swarm)

    # Expose ports to linked services only (not to host)
    expose:
      - "3000"
      - "8000"
```

### Volume Mounts

```yaml
services:
  app:
    volumes:
      # Short syntax
      - ./src:/app/src                      # Bind mount (relative path)
      - /var/log/app:/app/logs              # Bind mount (absolute path)
      - app-data:/app/data                  # Named volume
      - /app/node_modules                   # Anonymous volume

      # Long syntax
      - type: bind
        source: ./config
        target: /app/config
        read_only: true

      - type: volume
        source: db-data
        target: /var/lib/postgresql/data
        volume:
          nocopy: true              # Don't copy data from container

      - type: tmpfs                 # In-memory filesystem
        target: /app/tmp
        tmpfs:
          size: 100000000           # 100MB
          mode: 1777
```

### Environment Variables

```yaml
services:
  app:
    # Map syntax (preferred)
    environment:
      NODE_ENV: production
      DATABASE_URL: postgres://user:pass@db:5432/mydb
      DEBUG: "true"                 # Quote boolean-like values

    # List syntax
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgres://user:pass@db:5432/mydb

    # Pass through from host (value from shell)
    environment:
      - API_KEY                     # Uses host's $API_KEY
```

### depends_on - Service Dependencies

```yaml
services:
  api:
    depends_on:
      # Simple: just wait for container start
      - db
      - redis

  worker:
    depends_on:
      # Conditional: wait for specific conditions
      db:
        condition: service_healthy  # Wait for health check to pass
        restart: true               # Restart if dependency restarts
      redis:
        condition: service_started  # Default: just started
      migrations:
        condition: service_completed_successfully  # One-shot service

  db:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
```

### Other Service Options

```yaml
services:
  app:
    container_name: my-app          # Custom container name (avoid with scaling)
    hostname: app-host              # Container hostname
    domainname: example.com         # Container domain name
    working_dir: /app               # Working directory
    user: "1000:1000"               # User to run as (UID:GID)
    privileged: true                # Extended privileges (use cautiously)
    stdin_open: true                # Keep stdin open (docker run -i)
    tty: true                       # Allocate TTY (docker run -t)
    init: true                      # Run init as PID 1

    # Override entrypoint and command
    entrypoint: /app/entrypoint.sh
    command: ["npm", "run", "start:prod"]

    # Alternative command syntaxes
    command: npm run start          # Shell form
    command:
      - npm
      - run
      - start

    # Logging configuration
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

    # Add extra hosts to /etc/hosts
    extra_hosts:
      - "host.docker.internal:host-gateway"
      - "api.local:192.168.1.100"

    # DNS settings
    dns:
      - 8.8.8.8
      - 8.8.4.4
    dns_search:
      - example.com
```

---

## Health Checks

Health checks determine container readiness and health status.

```yaml
services:
  api:
    healthcheck:
      # Test command (required)
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      # Alternative test formats:
      # test: ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"]
      # test: curl -f http://localhost:3000/health  # Shell form

      interval: 30s       # Time between checks (default: 30s)
      timeout: 10s        # Max time for check to complete (default: 30s)
      retries: 3          # Consecutive failures for unhealthy (default: 3)
      start_period: 40s   # Grace period before counting failures (default: 0s)
      start_interval: 5s  # Interval during start_period (default: same as interval)

  # Disable health check
  legacy-app:
    healthcheck:
      disable: true

  # Common health check patterns
  postgres:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  mongodb:
    image: mongo:7
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 10s
      retries: 5
```

---

## Restart Policies

Control container restart behavior on exit or daemon restart.

```yaml
services:
  app:
    # Options: "no" | "always" | "on-failure" | "unless-stopped"
    restart: unless-stopped

  # Detailed examples
  web:
    restart: "no"           # Never restart (default)

  api:
    restart: always         # Always restart, including on daemon start

  worker:
    restart: on-failure     # Restart only on non-zero exit code

  background-job:
    restart: unless-stopped # Like always, but not if manually stopped

  # With on-failure, you can limit retries in deploy section
  critical-service:
    restart: on-failure
    deploy:
      restart_policy:
        condition: on-failure
        delay: 5s             # Wait between restarts
        max_attempts: 3       # Maximum restart attempts
        window: 120s          # Time window for max_attempts
```

---

## Networks

Networks enable container communication and isolation.

### Network Types

```yaml
services:
  web:
    networks:
      - frontend
      - backend

  api:
    networks:
      backend:
        aliases:              # Additional hostnames on this network
          - api-service
          - backend-api
        ipv4_address: 172.28.0.10  # Static IP (requires subnet config)

  db:
    networks:
      - backend

networks:
  # Bridge network (default driver)
  frontend:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.host_binding_ipv4: "127.0.0.1"

  # Internal network (no external access)
  backend:
    driver: bridge
    internal: true            # Containers can't access external network

  # Host network (container shares host network stack)
  # Note: defined at service level, not in networks section
  # services:
  #   app:
  #     network_mode: host

  # Custom subnet configuration
  custom:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 172.28.0.0/16
          ip_range: 172.28.5.0/24
          gateway: 172.28.0.1

  # External network (created outside Compose)
  existing-network:
    external: true
    name: my-existing-network   # Actual network name if different
```

### Network Modes

```yaml
services:
  # Use host's network directly
  host-networked:
    image: nginx
    network_mode: host

  # Share network with another container
  sidecar:
    image: datadog/agent
    network_mode: service:app   # Share network namespace with 'app' service

  # No network access
  isolated:
    image: alpine
    network_mode: none

  # Use container's network (legacy)
  legacy-link:
    image: app
    network_mode: container:other_container_id
```

---

## Volumes

Volumes persist data beyond container lifecycle.

### Named Volumes

```yaml
services:
  db:
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  # Simple named volume
  postgres-data:

  # Volume with options
  app-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /data/app-storage

  # External volume (pre-existing)
  shared-data:
    external: true
    name: actual_volume_name    # If name differs from key

  # Volume with labels
  labeled-volume:
    labels:
      com.example.description: "Application data"
      com.example.department: "IT"
```

### Bind Mounts

```yaml
services:
  app:
    volumes:
      # Relative path (relative to Compose file)
      - ./src:/app/src

      # Absolute path
      - /var/log/myapp:/app/logs

      # Read-only bind mount
      - ./config:/app/config:ro

      # Long syntax with options
      - type: bind
        source: ./data
        target: /app/data
        bind:
          create_host_path: true  # Create if doesn't exist
          selinux: z              # SELinux label (z=shared, Z=private)

      # Consistency options (macOS performance)
      - ./src:/app/src:cached     # Host authoritative
      - ./data:/app/data:delegated  # Container authoritative
```

### tmpfs Mounts

```yaml
services:
  app:
    volumes:
      - type: tmpfs
        target: /app/temp
        tmpfs:
          size: 52428800          # 50MB in bytes
          mode: 1777              # Permissions

    # Alternative syntax
    tmpfs:
      - /run
      - /tmp:size=100M,mode=1777
```

---

## Environment Files

Load environment variables from files.

```yaml
services:
  app:
    # Single env file
    env_file: .env

    # Multiple env files (loaded in order)
    env_file:
      - .env                      # Base configuration
      - .env.local                # Local overrides
      - .env.${ENVIRONMENT:-dev}  # Environment-specific

    # With options
    env_file:
      - path: .env
        required: true            # Fail if missing (default: true)
      - path: .env.local
        required: false           # Optional file
```

### Env File Format

```bash
# .env file example
# Comments start with #

# Simple key=value
NODE_ENV=production
API_PORT=3000

# Quoted values (preserves whitespace)
MESSAGE="Hello World"
PATH_VAR='/usr/local/bin'

# Multi-line values (use quotes)
PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA...
-----END RSA PRIVATE KEY-----"

# Empty values
EMPTY_VAR=

# Export syntax also works
export DATABASE_URL=postgres://localhost/db
```

### Variable Substitution

```yaml
services:
  app:
    image: ${REGISTRY:-docker.io}/myapp:${TAG:-latest}
    environment:
      # Required variable (fails if not set)
      API_KEY: ${API_KEY:?API_KEY is required}

      # Default if unset or empty
      NODE_ENV: ${NODE_ENV:-development}

      # Default only if unset (empty string is valid)
      DEBUG: ${DEBUG-false}

      # Escape dollar sign
      PRICE: $$99.99
```

---

## Profiles

Profiles enable selective service activation.

```yaml
services:
  # Always starts (no profile)
  web:
    image: nginx

  api:
    image: myapp/api
    profiles:
      - backend

  # Development-only services
  debug-tools:
    image: debug-tools
    profiles:
      - debug
      - dev

  # Testing services
  test-db:
    image: postgres:16
    profiles:
      - test

  # Multiple profiles
  monitoring:
    image: prometheus
    profiles:
      - monitoring
      - production
```

### Using Profiles

```bash
# Start default services (no profile)
docker compose up

# Start with specific profile
docker compose --profile backend up

# Start with multiple profiles
docker compose --profile backend --profile debug up

# Set via environment variable
COMPOSE_PROFILES=backend,debug docker compose up

# List services in a profile
docker compose --profile debug config --services
```

---

## Secrets and Configs

Manage sensitive data and configuration files.

### Secrets

```yaml
services:
  api:
    secrets:
      # Simple reference
      - db_password

      # With target path and permissions
      - source: api_key
        target: /run/secrets/api_key    # Default: /run/secrets/<secret_name>
        uid: "1000"
        gid: "1000"
        mode: 0400                      # Read-only by owner

secrets:
  # From file
  db_password:
    file: ./secrets/db_password.txt

  # External secret (created outside Compose)
  api_key:
    external: true
    name: production_api_key

  # From environment variable
  jwt_secret:
    environment: JWT_SECRET_VALUE
```

### Configs

```yaml
services:
  web:
    configs:
      - source: nginx_config
        target: /etc/nginx/nginx.conf
        uid: "0"
        gid: "0"
        mode: 0440

  app:
    configs:
      - app_config

configs:
  nginx_config:
    file: ./nginx/nginx.conf

  app_config:
    file: ./config/app.json

  # External config
  remote_config:
    external: true
```

---

## Resource Limits

Control CPU, memory, and other resource allocation.

```yaml
services:
  api:
    deploy:
      resources:
        # Hard limits
        limits:
          cpus: "2.0"             # Max 2 CPUs
          memory: 1G              # Max 1GB RAM
          pids: 100               # Max processes

        # Guaranteed resources (soft limits)
        reservations:
          cpus: "0.5"             # Guaranteed 0.5 CPU
          memory: 256M            # Guaranteed 256MB RAM
          devices:                # GPU allocation
            - driver: nvidia
              count: 1
              capabilities: [gpu]

  # Memory without swap
  worker:
    deploy:
      resources:
        limits:
          memory: 512M
    # Disable swap (requires additional config)
    mem_swappiness: 0

  # OOM priority
  critical-service:
    oom_score_adj: -500           # Less likely to be OOM killed (-1000 to 1000)

  non-critical:
    oom_score_adj: 500            # More likely to be OOM killed
    oom_kill_disable: false       # Allow OOM kill (default)
```

### CPU Configuration

```yaml
services:
  app:
    # CPU shares (relative weight)
    cpu_shares: 1024              # Default is 1024

    # CPU quota and period
    cpu_quota: 50000              # Microseconds per cpu_period
    cpu_period: 100000            # Default 100000 (100ms)

    # Pin to specific CPUs
    cpuset: "0,1"                 # Run on CPU 0 and 1

    # Real-time scheduling
    cpu_rt_runtime: 400000        # Microseconds per period
    cpu_rt_period: 1000000        # 1 second period
```

---

## CLI Commands Reference

### Basic Operations

```bash
# Start services
docker compose up                 # Foreground
docker compose up -d              # Detached (background)
docker compose up --build         # Rebuild images before starting
docker compose up --force-recreate  # Recreate containers
docker compose up --no-deps api   # Start specific service without dependencies

# Stop services
docker compose stop               # Stop containers (keep them)
docker compose down               # Stop and remove containers
docker compose down -v            # Also remove volumes
docker compose down --rmi all     # Also remove images
docker compose down --rmi local   # Remove only locally built images

# Restart services
docker compose restart            # Restart all
docker compose restart api        # Restart specific service
```

### Service Management

```bash
# List services and status
docker compose ps                 # Running containers
docker compose ps -a              # All containers (including stopped)
docker compose ls                 # List Compose projects

# Scale services
docker compose up -d --scale api=3 --scale worker=5

# Execute commands in containers
docker compose exec api sh        # Interactive shell
docker compose exec -T api npm test  # Non-interactive
docker compose exec --user root api bash  # Run as root

# Run one-off commands
docker compose run --rm api npm install  # Creates new container
docker compose run --rm -v $(pwd):/app api npm test

# View logs
docker compose logs               # All services
docker compose logs -f api        # Follow specific service
docker compose logs --tail=100    # Last 100 lines
docker compose logs --since 1h    # Last hour
docker compose logs --timestamps  # Include timestamps
```

### Build and Images

```bash
# Build images
docker compose build              # Build all
docker compose build api          # Build specific service
docker compose build --no-cache   # Build without cache
docker compose build --pull       # Pull base images first
docker compose build --parallel   # Build in parallel

# Push images
docker compose push               # Push all built images
docker compose push api           # Push specific service

# Pull images
docker compose pull               # Pull all images
docker compose pull --ignore-pull-failures  # Continue on failures
```

### Configuration and Debugging

```bash
# Validate and view config
docker compose config             # Validate and show resolved config
docker compose config --services  # List service names
docker compose config --volumes   # List volume names
docker compose config --images    # List image names

# Multiple compose files
docker compose -f docker-compose.yml -f docker-compose.prod.yml up

# Override project name
docker compose -p myproject up

# View resource usage
docker compose top                # Running processes
docker compose stats              # Live resource statistics

# Events
docker compose events             # Stream container events

# Port information
docker compose port api 3000      # Show host port for container port
```

### Maintenance

```bash
# Remove stopped containers
docker compose rm                 # Interactive
docker compose rm -f              # Force, no prompt
docker compose rm -s              # Stop if running, then remove

# Copy files
docker compose cp api:/app/logs ./logs  # From container
docker compose cp ./config api:/app/    # To container

# Pause/unpause
docker compose pause api
docker compose unpause api

# Wait for service
docker compose wait api           # Wait for container to exit
```

---

## Extends and Include

Reuse and compose multiple configuration files.

### Include (Compose 2.20+)

```yaml
# compose.yaml
include:
  - path: ./db/compose.yaml       # Include another compose file
  - path: ./cache/compose.yaml
  - path:
      - ./monitoring/compose.yaml
      - ./monitoring/compose.override.yaml  # With overrides
    project_directory: ./monitoring
    env_file: ./monitoring/.env

services:
  api:
    image: myapi
    depends_on:
      - db    # From included file
      - redis # From included file
```

### Extends

```yaml
# common.yaml
services:
  base-app:
    build:
      context: .
    environment:
      LOG_LEVEL: info
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3

# compose.yaml
services:
  api:
    extends:
      file: common.yaml
      service: base-app
    ports:
      - "3000:3000"
    environment:
      LOG_LEVEL: debug    # Override base value

  worker:
    extends:
      file: common.yaml
      service: base-app
    command: ["npm", "run", "worker"]
```

### Multiple Files Pattern

```bash
# Base configuration
# docker-compose.yml

# Development overrides
# docker-compose.override.yml (auto-loaded)

# Production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up

# CI/CD
docker compose -f docker-compose.yml -f docker-compose.ci.yml up
```

```yaml
# docker-compose.yml (base)
services:
  api:
    build: .
    environment:
      NODE_ENV: production

# docker-compose.override.yml (development - auto-loaded)
services:
  api:
    build:
      target: development
    volumes:
      - ./src:/app/src
    environment:
      NODE_ENV: development
      DEBUG: "true"
    ports:
      - "3000:3000"
      - "9229:9229"   # Debug port

# docker-compose.prod.yml (production)
services:
  api:
    image: myregistry/api:${TAG}
    deploy:
      replicas: 3
      resources:
        limits:
          memory: 1G
```

---

## Best Practices and Common Pitfalls

### Best Practices

```yaml
# 1. Use specific image tags, not 'latest'
services:
  db:
    image: postgres:16.1-alpine   # Good: specific version
    # image: postgres:latest      # Avoid: unpredictable

# 2. Always define health checks for dependencies
services:
  api:
    depends_on:
      db:
        condition: service_healthy  # Wait for actual readiness
  db:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]

# 3. Use named volumes for persistent data
volumes:
  postgres-data:                   # Named volume (managed by Docker)
  # ./data:/var/lib/postgresql    # Avoid bind mounts for DB data

# 4. Separate concerns with networks
networks:
  frontend:                        # Public-facing services
  backend:
    internal: true                 # Database, cache (no external access)

# 5. Use env_file for configuration, secrets for sensitive data
services:
  api:
    env_file: .env                 # Non-sensitive config
    secrets:
      - db_password                # Sensitive data

# 6. Set resource limits
services:
  api:
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"

# 7. Use .dockerignore to reduce build context
# .dockerignore
# node_modules
# .git
# *.md
```

### Common Pitfalls

```yaml
# PITFALL 1: Not handling container startup order correctly
# Wrong: depends_on without condition doesn't wait for readiness
services:
  api:
    depends_on:
      - db                        # Only waits for container start

# Correct: Use health checks
services:
  api:
    depends_on:
      db:
        condition: service_healthy

# PITFALL 2: Using container_name with scaling
# Wrong: Can't scale with fixed container name
services:
  api:
    container_name: my-api        # Prevents scaling

# Correct: Remove container_name or use deploy.replicas

# PITFALL 3: Bind mounting node_modules
# Wrong: Host node_modules overwrites container's
services:
  app:
    volumes:
      - ./:/app                   # Includes node_modules!

# Correct: Exclude node_modules with anonymous volume
services:
  app:
    volumes:
      - ./:/app
      - /app/node_modules         # Anonymous volume preserves container's modules

# PITFALL 4: Hardcoding secrets in compose file
# Wrong: Secrets in plain text
services:
  db:
    environment:
      POSTGRES_PASSWORD: mysecretpassword  # Visible in config!

# Correct: Use secrets or env_file
services:
  db:
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_pass
    secrets:
      - db_pass

# PITFALL 5: Using host network without understanding implications
# Wrong: Exposes all container ports on host
services:
  app:
    network_mode: host            # Security risk, port conflicts

# Correct: Map only needed ports
services:
  app:
    ports:
      - "3000:3000"

# PITFALL 6: Not cleaning up resources
# Remember to clean up
# docker compose down -v          # Remove volumes
# docker compose down --rmi all   # Remove images

# PITFALL 7: Environment variable precedence issues
# Precedence (highest to lowest):
# 1. Compose file environment section
# 2. Shell environment variables
# 3. env_file
# 4. Dockerfile ENV

# PITFALL 8: Forgetting about file permissions with bind mounts
# Container user may not match host user
services:
  app:
    user: "1000:1000"             # Match host user
    volumes:
      - ./data:/app/data
```

### Security Recommendations

```yaml
# 1. Don't run as root
services:
  app:
    user: "1000:1000"

# 2. Use read-only filesystem where possible
services:
  app:
    read_only: true
    volumes:
      - type: tmpfs
        target: /tmp
      - type: tmpfs
        target: /app/cache

# 3. Drop unnecessary capabilities
services:
  app:
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE          # Only what's needed

# 4. Use secrets for sensitive data
secrets:
  api_key:
    file: ./secrets/api_key.txt   # Not in version control!

# 5. Limit network exposure
services:
  db:
    networks:
      - backend                   # Not on frontend network
    # No ports exposed to host
```

---

## Quick Reference

```bash
# Development workflow
docker compose up -d              # Start all services
docker compose logs -f            # Follow logs
docker compose exec api sh        # Shell into container
docker compose restart api        # Restart after changes
docker compose down               # Stop and cleanup

# Production deployment
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
docker compose pull               # Pull latest images
docker compose up -d --no-deps api  # Update single service

# Debugging
docker compose ps -a              # Check container status
docker compose logs --tail=100 api  # Recent logs
docker compose exec api cat /proc/1/environ  # Check env vars
docker compose config             # Validate configuration
```
