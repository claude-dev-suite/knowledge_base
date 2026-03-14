# Dockerfile Reference

> Official Documentation: https://docs.docker.com/engine/reference/builder/

## Overview

A Dockerfile is a text document containing instructions to assemble a Docker image. Each instruction creates a layer in the image, and Docker caches these layers to speed up subsequent builds.

---

## All Dockerfile Instructions

### FROM - Base Image

The `FROM` instruction initializes a new build stage and sets the base image.

```dockerfile
# Basic usage
FROM ubuntu:22.04

# With platform specification
FROM --platform=linux/amd64 node:20-alpine

# Scratch (empty) image for static binaries
FROM scratch

# Using ARG before FROM
ARG BASE_IMAGE=node:20-alpine
FROM ${BASE_IMAGE}
```

### RUN - Execute Commands

The `RUN` instruction executes commands in a new layer on top of the current image.

```dockerfile
# Shell form (runs in /bin/sh -c)
RUN apt-get update && apt-get install -y curl

# Exec form (no shell processing)
RUN ["apt-get", "install", "-y", "curl"]

# Multi-line with cleanup (best practice)
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        curl \
        ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# With heredoc (BuildKit)
RUN <<EOF
apt-get update
apt-get install -y curl
rm -rf /var/lib/apt/lists/*
EOF

# Mount cache for package managers (BuildKit)
RUN --mount=type=cache,target=/var/cache/apt \
    apt-get update && apt-get install -y curl
```

### CMD - Default Command

The `CMD` instruction provides defaults for executing a container. Only the last CMD takes effect.

```dockerfile
# Exec form (preferred)
CMD ["node", "server.js"]

# Shell form
CMD node server.js

# As parameters to ENTRYPOINT
CMD ["--port", "3000"]
```

### ENTRYPOINT - Main Executable

The `ENTRYPOINT` instruction configures the container to run as an executable.

```dockerfile
# Exec form (preferred)
ENTRYPOINT ["node", "server.js"]

# Shell form (prevents CMD arguments from being passed)
ENTRYPOINT node server.js

# Combined with CMD for default arguments
ENTRYPOINT ["python", "app.py"]
CMD ["--host", "0.0.0.0"]
```

### COPY - Copy Files

The `COPY` instruction copies files from the build context into the image.

```dockerfile
# Basic copy
COPY package.json /app/

# Copy multiple files
COPY package.json package-lock.json /app/

# Copy directory contents
COPY src/ /app/src/

# With ownership (BuildKit)
COPY --chown=node:node . /app/

# With chmod (BuildKit)
COPY --chmod=755 scripts/entrypoint.sh /entrypoint.sh

# From another build stage
COPY --from=builder /app/dist /app/dist

# From external image
COPY --from=nginx:alpine /etc/nginx/nginx.conf /etc/nginx/

# With glob patterns
COPY *.json /app/
COPY src/**/*.js /app/src/
```

### ADD - Copy with Extra Features

The `ADD` instruction copies files with URL support and automatic tar extraction.

```dockerfile
# Copy from URL (not recommended, use curl/wget instead)
ADD https://example.com/file.tar.gz /app/

# Auto-extract tar archives
ADD archive.tar.gz /app/

# Local file (same as COPY)
ADD config.json /app/

# With checksum verification (BuildKit)
ADD --checksum=sha256:abc123... https://example.com/file /app/file
```

### ENV - Environment Variables

The `ENV` instruction sets environment variables that persist in the running container.

```dockerfile
# Single variable
ENV NODE_ENV=production

# Multiple variables (legacy syntax)
ENV NODE_ENV=production PORT=3000

# Multiple variables (recommended)
ENV NODE_ENV=production
ENV PORT=3000
ENV LOG_LEVEL=info

# Using variables in subsequent instructions
ENV APP_HOME=/app
WORKDIR ${APP_HOME}
COPY . ${APP_HOME}
```

### ARG - Build Arguments

The `ARG` instruction defines build-time variables passed via `--build-arg`.

```dockerfile
# With default value
ARG NODE_VERSION=20

# Without default (must be provided)
ARG GITHUB_TOKEN

# Usage in instructions
ARG BASE_IMAGE=node:20-alpine
FROM ${BASE_IMAGE}

# Convert to ENV for runtime access
ARG VERSION
ENV APP_VERSION=${VERSION}

# Predefined ARGs (no ARG declaration needed)
# HTTP_PROXY, HTTPS_PROXY, FTP_PROXY, NO_PROXY
# TARGETPLATFORM, TARGETOS, TARGETARCH, TARGETVARIANT
# BUILDPLATFORM, BUILDOS, BUILDARCH, BUILDVARIANT
```

### WORKDIR - Working Directory

The `WORKDIR` instruction sets the working directory for subsequent instructions.

```dockerfile
# Absolute path
WORKDIR /app

# Creates directory if it doesn't exist
WORKDIR /app/src/components

# Can use environment variables
ENV APP_HOME=/application
WORKDIR ${APP_HOME}

# Multiple WORKDIR instructions stack
WORKDIR /app
WORKDIR src
WORKDIR components
# Result: /app/src/components
```

### USER - Set User Context

The `USER` instruction sets the user and optionally group for subsequent instructions.

```dockerfile
# By name
USER node

# By UID
USER 1001

# User and group
USER node:node
USER 1001:1001

# Create and switch to non-root user
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
USER appuser

# Alpine Linux
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

### EXPOSE - Document Ports

The `EXPOSE` instruction documents which ports the container listens on.

```dockerfile
# Single port (TCP default)
EXPOSE 3000

# Multiple ports
EXPOSE 80 443

# Explicit protocol
EXPOSE 80/tcp
EXPOSE 53/udp

# Port range
EXPOSE 8000-8010
```

**Note:** EXPOSE does not publish ports. Use `-p` flag when running the container.

### VOLUME - Mount Points

The `VOLUME` instruction creates a mount point for external storage.

```dockerfile
# Single volume
VOLUME /data

# Multiple volumes
VOLUME ["/data", "/logs"]

# Common use cases
VOLUME /var/lib/mysql
VOLUME /var/log/nginx
```

**Note:** Files added to volumes after VOLUME instruction are not persisted in the image.

### LABEL - Metadata

The `LABEL` instruction adds metadata to an image.

```dockerfile
# Single label
LABEL version="1.0"

# Multiple labels
LABEL maintainer="dev@example.com"
LABEL description="Application server"
LABEL org.opencontainers.image.source="https://github.com/org/repo"

# OCI standard labels
LABEL org.opencontainers.image.title="My App"
LABEL org.opencontainers.image.description="My application description"
LABEL org.opencontainers.image.version="1.0.0"
LABEL org.opencontainers.image.vendor="My Company"
LABEL org.opencontainers.image.licenses="MIT"
LABEL org.opencontainers.image.created="2024-01-15T10:00:00Z"
```

### HEALTHCHECK - Container Health

The `HEALTHCHECK` instruction tells Docker how to test container health.

```dockerfile
# HTTP health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

# TCP health check
HEALTHCHECK CMD nc -z localhost 3000 || exit 1

# Custom script
HEALTHCHECK --interval=1m --timeout=10s \
    CMD /app/healthcheck.sh

# Disable health check inherited from base image
HEALTHCHECK NONE
```

**Options:**
- `--interval=DURATION` (default 30s): Time between checks
- `--timeout=DURATION` (default 30s): Maximum time for check
- `--start-period=DURATION` (default 0s): Grace period for startup
- `--retries=N` (default 3): Consecutive failures before unhealthy

### SHELL - Default Shell

The `SHELL` instruction overrides the default shell for shell-form commands.

```dockerfile
# Windows PowerShell
SHELL ["powershell", "-Command"]
RUN Get-ChildItem

# Bash instead of sh
SHELL ["/bin/bash", "-c"]
RUN source ~/.bashrc && echo $PATH

# Reset to default
SHELL ["/bin/sh", "-c"]
```

### STOPSIGNAL - Stop Signal

The `STOPSIGNAL` instruction sets the system call signal to stop the container.

```dockerfile
# Default is SIGTERM
STOPSIGNAL SIGTERM

# Use SIGQUIT for graceful shutdown
STOPSIGNAL SIGQUIT

# Use signal number
STOPSIGNAL 9
```

### ONBUILD - Trigger Instructions

The `ONBUILD` instruction adds triggers executed when image is used as base.

```dockerfile
# Triggered when child image builds
ONBUILD COPY . /app
ONBUILD RUN npm install

# Common in base images
ONBUILD ARG NODE_ENV=production
ONBUILD ENV NODE_ENV=${NODE_ENV}
```

---

## Multi-Stage Builds

Multi-stage builds allow you to use multiple FROM statements, reducing final image size.

```dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Test (optional)
FROM builder AS tester
RUN npm run test

# Stage 3: Production
FROM node:20-alpine AS production
WORKDIR /app

# Copy only production dependencies
COPY package*.json ./
RUN npm ci --only=production

# Copy built artifacts from builder
COPY --from=builder /app/dist ./dist

# Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 3000
CMD ["node", "dist/main.js"]
```

### Building Specific Stages

```bash
# Build only the builder stage
docker build --target builder -t myapp:builder .

# Build production stage (default, last stage)
docker build -t myapp:prod .

# Build test stage
docker build --target tester -t myapp:test .
```

### Advanced Multi-Stage Patterns

```dockerfile
# Pattern: Binary from Go
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /app/server

FROM scratch
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]

# Pattern: Parallel builds
FROM node:20-alpine AS frontend-builder
WORKDIR /frontend
COPY frontend/package*.json ./
RUN npm ci
COPY frontend/ .
RUN npm run build

FROM python:3.11-slim AS backend-builder
WORKDIR /backend
COPY backend/requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt
COPY backend/ .

FROM python:3.11-slim AS production
COPY --from=backend-builder /backend /app
COPY --from=frontend-builder /frontend/dist /app/static
WORKDIR /app
CMD ["python", "main.py"]
```

---

## BuildKit Features

BuildKit is Docker's modern build subsystem with enhanced features.

### Enabling BuildKit

```bash
# Environment variable
DOCKER_BUILDKIT=1 docker build .

# Docker daemon configuration (/etc/docker/daemon.json)
{ "features": { "buildkit": true } }

# Using buildx
docker buildx build .
```

### Build Secrets

```dockerfile
# Mount secret during build (never stored in image)
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci

# Usage
docker build --secret id=npmrc,src=.npmrc .
```

### Build Cache Mounts

```dockerfile
# Cache package manager downloads
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# Cache Go modules
RUN --mount=type=cache,target=/go/pkg/mod \
    go build -o /app/server

# Cache pip packages
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Cache apt packages
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y curl
```

### SSH Agent Forwarding

```dockerfile
# Access SSH agent for private repositories
RUN --mount=type=ssh \
    git clone git@github.com:private/repo.git

# Usage
docker build --ssh default .
```

### Heredocs

```dockerfile
# Multi-line scripts
RUN <<EOF
set -e
apt-get update
apt-get install -y curl
rm -rf /var/lib/apt/lists/*
EOF

# Creating files inline
COPY <<EOF /app/config.json
{
    "port": 3000,
    "env": "production"
}
EOF

# Multiple heredocs
RUN <<SCRIPT1 && <<SCRIPT2
echo "Running script 1"
SCRIPT1
echo "Running script 2"
SCRIPT2
```

---

## Layer Caching Strategies

### Optimal Layer Ordering

```dockerfile
FROM node:20-alpine

# Layer 1: System dependencies (rarely change)
RUN apk add --no-cache dumb-init tini

# Layer 2: Create user (rarely changes)
RUN addgroup -S app && adduser -S app -G app

# Layer 3: Package manifest (changes with dependencies)
WORKDIR /app
COPY package.json package-lock.json ./

# Layer 4: Install dependencies (cached unless manifest changes)
RUN npm ci --only=production

# Layer 5: Source code (changes frequently)
COPY --chown=app:app . .

USER app
CMD ["dumb-init", "node", "server.js"]
```

### Cache Busting

```dockerfile
# Add cache buster for specific layer
ARG CACHE_BUST=1
RUN echo "Cache bust: ${CACHE_BUST}" && apt-get update

# Version pinning prevents unwanted cache hits
RUN pip install requests==2.31.0

# Use checksums for downloaded files
ADD --checksum=sha256:abc123 https://example.com/file.tar.gz /tmp/
```

### External Cache Sources

```bash
# Use registry cache
docker build --cache-from myregistry/myapp:cache -t myapp .

# Export cache to registry
docker build --cache-to type=registry,ref=myregistry/myapp:cache -t myapp .

# Local cache directory
docker build --cache-from type=local,src=/tmp/cache \
             --cache-to type=local,dest=/tmp/cache -t myapp .
```

---

## .dockerignore

The `.dockerignore` file excludes files from the build context.

```dockerignore
# Dependencies
node_modules/
vendor/
__pycache__/
*.pyc
.venv/

# Build outputs
dist/
build/
*.egg-info/

# Version control
.git/
.gitignore
.svn/

# IDE and editor files
.idea/
.vscode/
*.swp
*.swo
*~

# Environment and secrets
.env
.env.*
*.pem
*.key
credentials.json
secrets/

# Documentation
README.md
docs/
*.md

# Tests
tests/
test/
__tests__/
*.test.js
*.spec.js
coverage/
.nyc_output/

# Docker files (optional)
Dockerfile*
docker-compose*.yml
.dockerignore

# OS files
.DS_Store
Thumbs.db

# Logs
logs/
*.log
npm-debug.log*

# Temporary files
tmp/
temp/
*.tmp

# Exception: include specific files
!config/production.json
```

---

## Security Best Practices

### Non-Root User

```dockerfile
# Debian/Ubuntu
FROM node:20-slim
RUN groupadd -r -g 1001 appgroup \
    && useradd -r -u 1001 -g appgroup -d /app -s /sbin/nologin appuser
WORKDIR /app
COPY --chown=appuser:appgroup . .
USER appuser

# Alpine
FROM node:20-alpine
RUN addgroup -g 1001 -S appgroup \
    && adduser -u 1001 -S appuser -G appgroup -h /app
WORKDIR /app
COPY --chown=appuser:appgroup . .
USER appuser

# Distroless (already non-root)
FROM gcr.io/distroless/nodejs20-debian12
COPY --chown=nonroot:nonroot . /app
USER nonroot
```

### Minimal Images

```dockerfile
# Use slim variants
FROM python:3.11-slim

# Use Alpine variants
FROM node:20-alpine

# Use distroless images
FROM gcr.io/distroless/static-debian12

# Scratch for static binaries
FROM scratch
COPY --from=builder /app/binary /binary
ENTRYPOINT ["/binary"]
```

### Secret Management

```dockerfile
# WRONG: Secrets in ENV or ARG
ENV DATABASE_PASSWORD=secret123  # Visible in image history

# CORRECT: Use BuildKit secrets
RUN --mount=type=secret,id=db_password \
    cat /run/secrets/db_password > /app/.env

# CORRECT: Runtime secrets via environment or files
# Pass at runtime: docker run -e DATABASE_PASSWORD=secret myapp
# Or mount: docker run -v ./secrets:/run/secrets myapp
```

### Image Scanning and Verification

```dockerfile
# Pin to digest for reproducibility
FROM node:20-alpine@sha256:abc123def456...

# Use verified base images
FROM docker.io/library/node:20-alpine

# Add vulnerability scanning in CI
# docker scout cves myimage:tag
# trivy image myimage:tag
```

### Minimize Attack Surface

```dockerfile
FROM node:20-alpine

# Remove unnecessary packages
RUN apk add --no-cache --virtual .build-deps \
        python3 \
        make \
        g++ \
    && npm ci \
    && apk del .build-deps

# Remove shells if not needed (advanced)
RUN rm -rf /bin/sh /bin/bash

# Read-only root filesystem support
# Run with: docker run --read-only myapp
```

---

## Image Optimization

### Alpine-Based Images

```dockerfile
FROM node:20-alpine

# Use apk with --no-cache
RUN apk add --no-cache \
    dumb-init \
    curl

# Alpine uses musl libc (may have compatibility issues)
# Test thoroughly with native dependencies
```

### Slim Images

```dockerfile
FROM python:3.11-slim

# Clean up apt cache
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        curl \
        ca-certificates \
    && rm -rf /var/lib/apt/lists/*
```

### Distroless Images

```dockerfile
# For Node.js
FROM node:20-slim AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

FROM gcr.io/distroless/nodejs20-debian12
WORKDIR /app
COPY --from=builder /app /app
CMD ["server.js"]

# For Go
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o /server

FROM gcr.io/distroless/static-debian12
COPY --from=builder /server /server
ENTRYPOINT ["/server"]

# For Java
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /app
COPY . .
RUN ./gradlew build

FROM gcr.io/distroless/java21-debian12
COPY --from=builder /app/build/libs/*.jar /app.jar
CMD ["app.jar"]
```

### Image Size Comparison

| Base Image | Approximate Size |
|------------|------------------|
| `ubuntu:22.04` | ~77MB |
| `debian:12-slim` | ~74MB |
| `alpine:3.18` | ~7MB |
| `node:20` | ~1GB |
| `node:20-slim` | ~200MB |
| `node:20-alpine` | ~130MB |
| `gcr.io/distroless/nodejs20` | ~130MB |
| `gcr.io/distroless/static` | ~2MB |

---

## Build Arguments vs Environment Variables

### ARG (Build-Time Only)

```dockerfile
# Available only during build
ARG NODE_VERSION=20
FROM node:${NODE_VERSION}-alpine

ARG BUILD_DATE
ARG GIT_COMMIT
LABEL build-date=${BUILD_DATE} git-commit=${GIT_COMMIT}

# Build command
# docker build --build-arg BUILD_DATE=$(date -u +%Y-%m-%d) \
#              --build-arg GIT_COMMIT=$(git rev-parse HEAD) .
```

### ENV (Build and Runtime)

```dockerfile
# Available during build AND runtime
ENV NODE_ENV=production
ENV PORT=3000

# Can be overridden at runtime
# docker run -e PORT=8080 myapp
```

### Converting ARG to ENV

```dockerfile
# Pattern: Use ARG for build, ENV for runtime
ARG APP_VERSION=1.0.0
ENV APP_VERSION=${APP_VERSION}

# Now APP_VERSION is available at runtime
```

### Scope Rules

```dockerfile
# ARG before FROM: only available in FROM
ARG BASE_VERSION=20
FROM node:${BASE_VERSION}-alpine

# ARG must be redeclared after FROM to use in subsequent instructions
ARG BASE_VERSION
RUN echo "Building with Node ${BASE_VERSION}"
```

---

## CMD vs ENTRYPOINT

### CMD Only

```dockerfile
# Default command, easily overridden
FROM node:20-alpine
CMD ["node", "server.js"]

# Override: docker run myapp node other-script.js
```

### ENTRYPOINT Only

```dockerfile
# Fixed executable
FROM alpine
ENTRYPOINT ["curl"]

# docker run myapp https://example.com
# Runs: curl https://example.com
```

### Combined (Recommended Pattern)

```dockerfile
# ENTRYPOINT: the executable
# CMD: default arguments
FROM python:3.11-slim
ENTRYPOINT ["python", "app.py"]
CMD ["--host", "0.0.0.0", "--port", "8000"]

# docker run myapp
# Runs: python app.py --host 0.0.0.0 --port 8000

# docker run myapp --port 3000
# Runs: python app.py --port 3000
```

### Shell vs Exec Form

```dockerfile
# Exec form (preferred) - PID 1, receives signals
ENTRYPOINT ["node", "server.js"]
CMD ["--port", "3000"]

# Shell form - runs in shell, node is child process
ENTRYPOINT node server.js  # PID 1 is /bin/sh, node is child
CMD --port 3000            # Ignored with shell-form ENTRYPOINT
```

### Init Process Pattern

```dockerfile
# Use tini or dumb-init as PID 1
FROM node:20-alpine
RUN apk add --no-cache dumb-init
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "server.js"]
```

---

## COPY vs ADD

### Use COPY (Preferred)

```dockerfile
# Simple, predictable file copying
COPY package.json /app/
COPY src/ /app/src/
COPY --chown=node:node . /app/
```

### Use ADD Only For

```dockerfile
# Auto-extracting local tar archives
ADD archive.tar.gz /app/

# Downloading with checksum verification (BuildKit)
ADD --checksum=sha256:abc123 https://example.com/file /app/
```

### Why Prefer COPY

| Feature | COPY | ADD |
|---------|------|-----|
| Copy local files | Yes | Yes |
| Predictable behavior | Yes | No |
| Auto-extract tar | No | Yes |
| Download URLs | No | Yes |
| Caching | Better | Worse |

---

## Best Practices and Common Pitfalls

### Best Practices

```dockerfile
# 1. Use specific base image tags
FROM node:20.10.0-alpine3.18  # Good
# FROM node:latest            # Bad - unpredictable

# 2. Combine RUN commands to reduce layers
RUN apt-get update \
    && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*

# 3. Order layers by change frequency
COPY package.json .           # Changes less often
RUN npm ci
COPY . .                      # Changes frequently

# 4. Use .dockerignore
# Exclude node_modules, .git, etc.

# 5. Run as non-root user
USER node

# 6. Use multi-stage builds
FROM node:20 AS builder
# ... build steps
FROM node:20-alpine AS production
COPY --from=builder /app/dist /app/dist

# 7. Set proper signals handling
STOPSIGNAL SIGTERM
ENTRYPOINT ["dumb-init", "--"]

# 8. Add health checks
HEALTHCHECK CMD curl -f http://localhost:3000/health || exit 1

# 9. Use explicit WORKDIR
WORKDIR /app  # Not: RUN cd /app

# 10. Document with LABEL
LABEL org.opencontainers.image.source="https://github.com/org/repo"
```

### Common Pitfalls

```dockerfile
# PITFALL 1: Installing unnecessary packages
# Bad
RUN apt-get install -y vim curl wget git
# Good
RUN apt-get install -y --no-install-recommends curl

# PITFALL 2: Not cleaning up in same layer
# Bad (cache persists in layer)
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*
# Good
RUN apt-get update \
    && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*

# PITFALL 3: Copying secrets into image
# Bad
COPY .env /app/.env
ENV API_KEY=secret123
# Good - use runtime secrets
# docker run -e API_KEY=secret myapp

# PITFALL 4: Running as root
# Bad
FROM node:20
COPY . /app
CMD ["node", "app.js"]
# Good
FROM node:20
RUN useradd -r -u 1001 appuser
USER appuser
COPY --chown=appuser . /app
CMD ["node", "app.js"]

# PITFALL 5: Using latest tag
# Bad
FROM node:latest
# Good
FROM node:20.10.0-alpine3.18

# PITFALL 6: Large build context
# Bad - sending entire project including node_modules
# Good - use .dockerignore

# PITFALL 7: Shell form preventing signal handling
# Bad
CMD node server.js
# Good
CMD ["node", "server.js"]

# PITFALL 8: Not using multi-stage for compiled languages
# Bad - includes build tools in final image
FROM golang:1.21
COPY . .
RUN go build -o /app
CMD ["/app"]
# Good
FROM golang:1.21 AS builder
COPY . .
RUN go build -o /app
FROM scratch
COPY --from=builder /app /app
CMD ["/app"]
```

### Debugging Dockerfiles

```bash
# Build with progress output
docker build --progress=plain .

# Build specific stage
docker build --target builder -t myapp:debug .

# Run intermediate image
docker run -it --rm myapp:debug sh

# Inspect image layers
docker history myapp:latest

# Check image size
docker images myapp

# View build cache
docker builder prune --dry-run
```

---

## Complete Example: Production Node.js Application

```dockerfile
# syntax=docker/dockerfile:1
ARG NODE_VERSION=20
ARG ALPINE_VERSION=3.18

# Stage 1: Dependencies
FROM node:${NODE_VERSION}-alpine${ALPINE_VERSION} AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

# Stage 2: Build
FROM deps AS builder
COPY . .
RUN npm run build

# Stage 3: Production dependencies
FROM node:${NODE_VERSION}-alpine${ALPINE_VERSION} AS prod-deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --only=production

# Stage 4: Production
FROM node:${NODE_VERSION}-alpine${ALPINE_VERSION} AS production

# Security: Install security updates
RUN apk upgrade --no-cache

# Install init system
RUN apk add --no-cache dumb-init

# Create non-root user
RUN addgroup -g 1001 -S nodejs \
    && adduser -u 1001 -S nodejs -G nodejs

WORKDIR /app

# Copy production dependencies
COPY --from=prod-deps --chown=nodejs:nodejs /app/node_modules ./node_modules

# Copy built application
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --chown=nodejs:nodejs package.json ./

# Set environment
ENV NODE_ENV=production
ENV PORT=3000

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD node -e "require('http').get('http://localhost:3000/health', (r) => process.exit(r.statusCode === 200 ? 0 : 1))"

# Start application
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "dist/main.js"]
```
