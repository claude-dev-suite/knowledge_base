# Docker Best Practices

> Comprehensive guide to Docker best practices based on official Docker documentation and industry standards.

## Table of Contents

1. [Image Building Best Practices](#1-image-building-best-practices)
2. [Multi-Stage Builds](#2-multi-stage-builds)
3. [.dockerignore Configuration](#3-dockerignore-configuration)
4. [Layer Caching Optimization](#4-layer-caching-optimization)
5. [Image Size Optimization](#5-image-size-optimization)
6. [Security Best Practices](#6-security-best-practices)
7. [Base Image Selection](#7-base-image-selection)
8. [Environment Variables and Secrets](#8-environment-variables-and-secrets)
9. [Health Checks](#9-health-checks)
10. [Logging Best Practices](#10-logging-best-practices)
11. [Volume Best Practices](#11-volume-best-practices)
12. [Networking Best Practices](#12-networking-best-practices)
13. [Docker Compose Best Practices](#13-docker-compose-best-practices)
14. [Container Orchestration Considerations](#14-container-orchestration-considerations)
15. [CI/CD Integration](#15-cicd-integration)
16. [Monitoring and Debugging](#16-monitoring-and-debugging)
17. [Common Patterns and Anti-Patterns](#17-common-patterns-and-anti-patterns)

---

## 1. Image Building Best Practices

### 1.1 Use Official Base Images

Always prefer official images from Docker Hub as they are maintained, regularly updated, and follow best practices.

```dockerfile
# Good - Official image
FROM node:20-alpine

# Avoid - Unknown or unmaintained images
FROM random-user/node-custom
```

### 1.2 Pin Image Versions

Never use `latest` tag in production. Always pin to specific versions for reproducibility.

```dockerfile
# Good - Pinned version
FROM python:3.11.4-slim-bookworm

# Avoid - Unpredictable builds
FROM python:latest
```

### 1.3 Use Minimal Base Images

Choose the smallest base image that meets your requirements.

```dockerfile
# Size comparison (approximate):
# node:20           - ~1GB
# node:20-slim      - ~200MB
# node:20-alpine    - ~130MB

FROM node:20-alpine
```

### 1.4 Reduce Number of Layers

Combine related commands to reduce the number of layers.

```dockerfile
# Good - Single layer for related commands
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        wget \
        git && \
    rm -rf /var/lib/apt/lists/*

# Avoid - Multiple layers for related operations
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y wget
RUN apt-get install -y git
```

### 1.5 Order Instructions by Change Frequency

Place instructions that change less frequently at the top for better cache utilization.

```dockerfile
# Good ordering
FROM node:20-alpine

# Rarely changes
WORKDIR /app

# Changes occasionally
COPY package*.json ./
RUN npm ci --only=production

# Changes frequently
COPY . .

CMD ["node", "server.js"]
```

### 1.6 Use COPY Instead of ADD

Prefer `COPY` over `ADD` unless you need tar extraction or URL fetching.

```dockerfile
# Good - Explicit file copy
COPY requirements.txt .

# Only use ADD for specific purposes
ADD https://example.com/file.tar.gz /tmp/
ADD archive.tar.gz /destination/
```

### 1.7 Set Appropriate Working Directory

Always set a working directory to avoid running commands in root.

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY . .

RUN pip install -r requirements.txt

CMD ["python", "app.py"]
```

### 1.8 Use Non-Root Users

Create and use non-root users for security.

```dockerfile
FROM node:20-alpine

WORKDIR /app

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001

COPY --chown=nextjs:nodejs . .

USER nextjs

CMD ["node", "server.js"]
```

---

## 2. Multi-Stage Builds

### 2.1 Basic Multi-Stage Build

Multi-stage builds allow you to use multiple FROM statements, keeping only what you need in the final image.

```dockerfile
# Build stage
FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# Production stage
FROM node:20-alpine AS production

WORKDIR /app

COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules

USER node

EXPOSE 3000

CMD ["node", "dist/server.js"]
```

### 2.2 Multi-Stage Build for Go Applications

```dockerfile
# Build stage
FROM golang:1.21-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

# Production stage - minimal image
FROM scratch

COPY --from=builder /app/main /main

ENTRYPOINT ["/main"]
```

### 2.3 Multi-Stage Build for Java Applications

```dockerfile
# Build stage
FROM maven:3.9-eclipse-temurin-17 AS builder

WORKDIR /app

COPY pom.xml .
RUN mvn dependency:go-offline

COPY src ./src
RUN mvn package -DskipTests

# Production stage
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

COPY --from=builder /app/target/*.jar app.jar

RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 2.4 Multi-Stage Build for Python Applications

```dockerfile
# Build stage
FROM python:3.11-slim AS builder

WORKDIR /app

RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Production stage
FROM python:3.11-slim

WORKDIR /app

COPY --from=builder /opt/venv /opt/venv

ENV PATH="/opt/venv/bin:$PATH"

COPY . .

RUN useradd -m appuser
USER appuser

EXPOSE 8000

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "app:app"]
```

### 2.5 Multi-Stage Build for Frontend Applications

```dockerfile
# Build stage
FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# Production stage with Nginx
FROM nginx:alpine

COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### 2.6 Named Build Stages

Use named stages for clarity and selective building.

```dockerfile
FROM node:20-alpine AS base
WORKDIR /app
COPY package*.json ./

FROM base AS dependencies
RUN npm ci

FROM base AS development
RUN npm install
COPY . .
CMD ["npm", "run", "dev"]

FROM dependencies AS builder
COPY . .
RUN npm run build

FROM node:20-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=dependencies /app/node_modules ./node_modules
USER node
CMD ["node", "dist/server.js"]
```

### 2.7 Build Specific Stages

```bash
# Build only the development stage
docker build --target development -t myapp:dev .

# Build only the production stage
docker build --target production -t myapp:prod .
```

---

## 3. .dockerignore Configuration

### 3.1 Basic .dockerignore

The `.dockerignore` file excludes files from the build context, reducing build time and image size.

```dockerignore
# Version control
.git
.gitignore
.gitattributes

# Documentation
README.md
CHANGELOG.md
LICENSE
docs/

# IDE and editor files
.vscode/
.idea/
*.swp
*.swo
*~

# OS files
.DS_Store
Thumbs.db

# Build artifacts
dist/
build/
*.log
```

### 3.2 Node.js .dockerignore

```dockerignore
# Dependencies
node_modules/
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# Testing
coverage/
.nyc_output/
*.test.js
*.spec.js
__tests__/
__mocks__/

# Development files
.env.local
.env.development
.env.test

# Build tools config
.eslintrc*
.prettierrc*
jest.config.*
webpack.config.js
tsconfig.json

# Documentation
docs/
*.md

# IDE
.vscode/
.idea/

# Git
.git/
.gitignore

# Docker files (prevent recursive builds)
Dockerfile*
docker-compose*
.dockerignore
```

### 3.3 Python .dockerignore

```dockerignore
# Byte-compiled files
__pycache__/
*.py[cod]
*$py.class
*.so

# Virtual environments
venv/
.venv/
env/
.env/

# Testing
.pytest_cache/
.coverage
htmlcov/
.tox/
.nox/

# Distribution
dist/
build/
*.egg-info/
*.egg

# IDE
.vscode/
.idea/
*.swp

# Jupyter
.ipynb_checkpoints/

# Local config
.env
.env.local
*.local

# Documentation
docs/
*.md

# Git
.git/
.gitignore

# Docker
Dockerfile*
docker-compose*
.dockerignore
```

### 3.4 Java/Maven .dockerignore

```dockerignore
# Build output
target/
*.class
*.jar
*.war
*.ear

# IDE
.idea/
*.iml
.project
.classpath
.settings/
.vscode/

# Logs
*.log

# OS
.DS_Store
Thumbs.db

# Maven
.mvn/wrapper/maven-wrapper.jar

# Testing
test-output/

# Documentation
docs/
*.md

# Git
.git/
.gitignore

# Docker
Dockerfile*
docker-compose*
.dockerignore
```

### 3.5 Go .dockerignore

```dockerignore
# Binaries
*.exe
*.exe~
*.dll
*.so
*.dylib

# Test binary
*.test

# Output
vendor/
bin/

# IDE
.idea/
.vscode/
*.swp

# Coverage
coverage.out
coverage.html

# Documentation
docs/
*.md

# Git
.git/
.gitignore

# Docker
Dockerfile*
docker-compose*
.dockerignore
```

### 3.6 .dockerignore Patterns

```dockerignore
# Match any file
*

# Exception - include specific files
!src/
!package.json
!package-lock.json

# Match specific extensions
*.log
*.tmp
*.cache

# Match directories
**/node_modules
**/__pycache__

# Negation patterns
*.md
!README.md

# Match files in subdirectories
**/*.test.js
**/test/
```

---

## 4. Layer Caching Optimization

### 4.1 Understanding Docker Layer Cache

Each instruction in a Dockerfile creates a layer. Docker caches layers and reuses them if nothing has changed.

```dockerfile
# Layer 1: Base image (cached until base image changes)
FROM node:20-alpine

# Layer 2: Workdir (cached until instruction changes)
WORKDIR /app

# Layer 3: Dependencies (cached until package files change)
COPY package*.json ./
RUN npm ci

# Layer 4: Source code (changes frequently, invalidates cache)
COPY . .

# Layer 5: Build (rebuilt when source changes)
RUN npm run build
```

### 4.2 Dependency Caching Strategy

Separate dependency installation from source code copying.

```dockerfile
# Good - Dependencies cached separately
FROM python:3.11-slim

WORKDIR /app

# Install dependencies first (changes less frequently)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy source code last (changes frequently)
COPY . .

CMD ["python", "app.py"]
```

### 4.3 Using BuildKit Cache Mounts

BuildKit provides advanced caching capabilities.

```dockerfile
# syntax=docker/dockerfile:1

FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .

# Use cache mount for pip cache
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

COPY . .

CMD ["python", "app.py"]
```

### 4.4 NPM/Yarn Cache Optimization

```dockerfile
# syntax=docker/dockerfile:1

FROM node:20-alpine

WORKDIR /app

COPY package*.json ./

# Cache npm packages
RUN --mount=type=cache,target=/root/.npm \
    npm ci --only=production

COPY . .

CMD ["node", "server.js"]
```

### 4.5 Go Module Cache Optimization

```dockerfile
# syntax=docker/dockerfile:1

FROM golang:1.21-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./

# Cache Go modules
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

COPY . .

RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -o main .

FROM scratch
COPY --from=builder /app/main /main
ENTRYPOINT ["/main"]
```

### 4.6 Maven Cache Optimization

```dockerfile
# syntax=docker/dockerfile:1

FROM maven:3.9-eclipse-temurin-17 AS builder

WORKDIR /app

COPY pom.xml .

# Cache Maven dependencies
RUN --mount=type=cache,target=/root/.m2 \
    mvn dependency:go-offline

COPY src ./src

RUN --mount=type=cache,target=/root/.m2 \
    mvn package -DskipTests

FROM eclipse-temurin:17-jre-alpine
COPY --from=builder /app/target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 4.7 Cache Invalidation Awareness

Understanding what invalidates the cache:

```dockerfile
# This instruction invalidates cache for all subsequent layers
# if ANY file in the context changes
COPY . .

# Better approach - copy only what's needed
COPY src/ ./src/
COPY config/ ./config/

# ARG values can invalidate cache
ARG VERSION=1.0.0
RUN echo "Building version $VERSION"
```

---

## 5. Image Size Optimization

### 5.1 Choose Minimal Base Images

```dockerfile
# Image size comparison:
# ubuntu:22.04      - ~77MB
# debian:bookworm   - ~117MB
# alpine:3.18       - ~7MB
# distroless        - ~2-20MB

# For compiled languages, use scratch or distroless
FROM scratch
COPY --from=builder /app/binary /binary
ENTRYPOINT ["/binary"]
```

### 5.2 Clean Up in the Same Layer

```dockerfile
# Good - Clean up in same layer
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        build-essential \
        curl && \
    make && \
    apt-get purge -y build-essential && \
    apt-get autoremove -y && \
    rm -rf /var/lib/apt/lists/*

# Bad - Clean up in different layer (space not reclaimed)
RUN apt-get update
RUN apt-get install -y build-essential
RUN make
RUN apt-get purge -y build-essential
RUN rm -rf /var/lib/apt/lists/*
```

### 5.3 Use --no-install-recommends

```dockerfile
# Only install required packages
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        package1 \
        package2 && \
    rm -rf /var/lib/apt/lists/*
```

### 5.4 Remove Unnecessary Files

```dockerfile
RUN pip install --no-cache-dir -r requirements.txt

# Remove documentation and tests
RUN find /usr/local -type d -name __pycache__ -exec rm -rf {} + && \
    find /usr/local -type f -name "*.pyc" -delete && \
    find /usr/local -type f -name "*.pyo" -delete
```

### 5.5 Use Distroless Images

```dockerfile
# Build stage
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o server .

# Production with distroless
FROM gcr.io/distroless/static-debian12
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]
```

### 5.6 Minimize Layers

```dockerfile
# Combine commands
RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        gnupg; \
    mkdir -p /etc/apt/keyrings; \
    curl -fsSL https://example.com/key.gpg | gpg --dearmor -o /etc/apt/keyrings/example.gpg; \
    echo "deb [signed-by=/etc/apt/keyrings/example.gpg] https://example.com/repo stable main" > /etc/apt/sources.list.d/example.list; \
    apt-get update; \
    apt-get install -y --no-install-recommends example-package; \
    rm -rf /var/lib/apt/lists/*
```

### 5.7 Analyze Image Size

```bash
# View image layers and sizes
docker history myimage:latest

# Use dive tool for detailed analysis
dive myimage:latest

# Check image size
docker images myimage:latest --format "{{.Size}}"
```

### 5.8 Squash Layers (Optional)

```bash
# Build with squash (experimental)
docker build --squash -t myimage:squashed .
```

---

## 6. Security Best Practices

### 6.1 Use Non-Root Users

```dockerfile
FROM node:20-alpine

WORKDIR /app

# Create non-root user
RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup

# Set ownership
COPY --chown=appuser:appgroup . .

# Switch to non-root user
USER appuser

CMD ["node", "server.js"]
```

### 6.2 Scan Images for Vulnerabilities

```bash
# Docker Scout (built-in)
docker scout cves myimage:latest

# Trivy
trivy image myimage:latest

# Snyk
snyk container test myimage:latest

# Clair
clair-scanner myimage:latest
```

### 6.3 Keep Images Updated

```dockerfile
# Always use specific versions and update regularly
FROM node:20.10.0-alpine3.18

# Update base image packages
RUN apk update && apk upgrade --no-cache
```

### 6.4 Don't Store Secrets in Images

```dockerfile
# NEVER do this
ENV API_KEY=secret123
COPY .env /app/.env

# Instead, use runtime secrets
# Pass secrets at runtime:
# docker run -e API_KEY=secret123 myimage
# Or use Docker secrets (Swarm mode)
# Or use external secret management
```

### 6.5 Use Read-Only Root Filesystem

```yaml
# docker-compose.yml
services:
  app:
    image: myimage:latest
    read_only: true
    tmpfs:
      - /tmp
      - /var/run
```

```bash
# Docker run
docker run --read-only --tmpfs /tmp myimage:latest
```

### 6.6 Limit Container Capabilities

```yaml
# docker-compose.yml
services:
  app:
    image: myimage:latest
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    security_opt:
      - no-new-privileges:true
```

### 6.7 Use Security Scanning in CI

```yaml
# GitHub Actions example
- name: Build image
  run: docker build -t myimage:${{ github.sha }} .

- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myimage:${{ github.sha }}
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'
```

### 6.8 Sign Images

```bash
# Enable Docker Content Trust
export DOCKER_CONTENT_TRUST=1

# Sign and push
docker push myregistry/myimage:latest

# Verify signature
docker trust inspect myregistry/myimage:latest
```

### 6.9 Use Minimal Base Images

```dockerfile
# Distroless for minimal attack surface
FROM gcr.io/distroless/static-debian12

# Or scratch for statically compiled binaries
FROM scratch
```

### 6.10 Set Resource Limits

```yaml
# docker-compose.yml
services:
  app:
    image: myimage:latest
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
```

---

## 7. Base Image Selection

### 7.1 Base Image Comparison

| Base Image | Size | Use Case |
|------------|------|----------|
| scratch | 0MB | Statically compiled binaries |
| distroless | ~2-20MB | Production containers |
| alpine | ~7MB | Small, security-focused |
| slim variants | ~50-150MB | Reduced Debian/Ubuntu |
| full variants | ~200MB+ | Development, debugging |

### 7.2 Alpine Linux

```dockerfile
FROM alpine:3.18

# Install packages
RUN apk add --no-cache \
    python3 \
    py3-pip

# Note: Alpine uses musl libc instead of glibc
# Some applications may have compatibility issues
```

### 7.3 Debian Slim

```dockerfile
FROM debian:bookworm-slim

# Smaller than full Debian, larger than Alpine
# Better compatibility with glibc applications
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        python3 \
        python3-pip && \
    rm -rf /var/lib/apt/lists/*
```

### 7.4 Distroless Images

```dockerfile
# For Java
FROM gcr.io/distroless/java17-debian12

# For Python
FROM gcr.io/distroless/python3-debian12

# For Node.js
FROM gcr.io/distroless/nodejs20-debian12

# For static binaries
FROM gcr.io/distroless/static-debian12

# For C/C++ with glibc
FROM gcr.io/distroless/base-debian12
```

### 7.5 Language-Specific Recommendations

```dockerfile
# Node.js
FROM node:20-alpine              # Smallest
FROM node:20-slim                # Good compatibility
FROM node:20-bookworm            # Full featured

# Python
FROM python:3.11-alpine          # Smallest (potential musl issues)
FROM python:3.11-slim-bookworm   # Recommended
FROM python:3.11-bookworm        # Full featured

# Java
FROM eclipse-temurin:17-jre-alpine    # Smallest runtime
FROM eclipse-temurin:17-jre-jammy     # Standard runtime
FROM eclipse-temurin:17-jdk-jammy     # Development

# Go
FROM golang:1.21-alpine          # For building
FROM scratch                      # For production (static binary)

# Rust
FROM rust:1.73-alpine            # For building
FROM scratch                      # For production (static binary)
```

### 7.6 Choosing Based on Requirements

```dockerfile
# For maximum security and minimal size
FROM gcr.io/distroless/static-debian12

# For debugging capabilities
FROM gcr.io/distroless/static-debian12:debug

# For shell access during development
FROM alpine:3.18

# For compatibility with native extensions
FROM debian:bookworm-slim
```

---

## 8. Environment Variables and Secrets

### 8.1 Using ARG for Build-Time Variables

```dockerfile
# Define build argument
ARG NODE_VERSION=20
ARG APP_VERSION=1.0.0

FROM node:${NODE_VERSION}-alpine

# Use ARG in build
ARG APP_VERSION
RUN echo "Building version ${APP_VERSION}"

# Convert ARG to ENV for runtime
ENV APP_VERSION=${APP_VERSION}
```

### 8.2 Using ENV for Runtime Variables

```dockerfile
FROM node:20-alpine

# Set default values
ENV NODE_ENV=production \
    PORT=3000 \
    LOG_LEVEL=info

WORKDIR /app

COPY . .

EXPOSE ${PORT}

CMD ["node", "server.js"]
```

### 8.3 Never Hardcode Secrets

```dockerfile
# NEVER do this
ENV DATABASE_PASSWORD=mysecretpassword
COPY .env /app/.env
RUN echo "password123" > /app/config/secrets

# CORRECT: Use runtime injection
# docker run -e DATABASE_PASSWORD=mysecret myimage
```

### 8.4 Docker Secrets (Swarm Mode)

```yaml
# docker-compose.yml (Swarm mode)
version: '3.8'
services:
  app:
    image: myapp:latest
    secrets:
      - db_password
      - api_key
    environment:
      DB_PASSWORD_FILE: /run/secrets/db_password

secrets:
  db_password:
    external: true
  api_key:
    file: ./secrets/api_key.txt
```

```dockerfile
# Read secrets from files in application
FROM node:20-alpine

WORKDIR /app

COPY . .

# Application reads from /run/secrets/
CMD ["node", "server.js"]
```

### 8.5 BuildKit Secrets for Build Time

```dockerfile
# syntax=docker/dockerfile:1

FROM node:20-alpine

WORKDIR /app

# Mount secret during build (not stored in image)
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm install

COPY . .

CMD ["node", "server.js"]
```

```bash
# Build with secret
docker build --secret id=npmrc,src=$HOME/.npmrc -t myapp .
```

### 8.6 Environment Variable Best Practices

```dockerfile
# Group related variables
ENV APP_HOME=/app \
    APP_USER=appuser \
    APP_GROUP=appgroup

# Use default values with fallbacks
ENV LOG_LEVEL=${LOG_LEVEL:-info}

# Document required environment variables
# Required: DATABASE_URL, API_KEY
# Optional: LOG_LEVEL (default: info), PORT (default: 3000)
```

### 8.7 .env Files with Docker Compose

```yaml
# docker-compose.yml
services:
  app:
    image: myapp:latest
    env_file:
      - .env
      - .env.local
    environment:
      - NODE_ENV=production
```

```bash
# .env file (not committed to git)
DATABASE_URL=postgres://user:pass@db:5432/mydb
API_KEY=your-api-key
SECRET_KEY=your-secret-key
```

---

## 9. Health Checks

### 9.1 Basic Health Check

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY . .

RUN npm ci --only=production

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "server.js"]
```

### 9.2 Health Check Options

```dockerfile
HEALTHCHECK \
    --interval=30s \        # Time between checks
    --timeout=10s \         # Timeout for single check
    --start-period=60s \    # Grace period for container startup
    --retries=3 \           # Consecutive failures before unhealthy
    CMD curl -f http://localhost:8080/health || exit 1
```

### 9.3 Health Check Commands

```dockerfile
# Using curl
HEALTHCHECK CMD curl -f http://localhost:3000/health || exit 1

# Using wget (smaller footprint)
HEALTHCHECK CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

# Using custom script
COPY healthcheck.sh /healthcheck.sh
HEALTHCHECK CMD /healthcheck.sh

# For non-HTTP services
HEALTHCHECK CMD pg_isready -U postgres || exit 1
HEALTHCHECK CMD redis-cli ping || exit 1
```

### 9.4 Implementing Health Endpoints

```javascript
// Node.js Express example
app.get('/health', (req, res) => {
  res.status(200).json({
    status: 'healthy',
    timestamp: new Date().toISOString(),
    uptime: process.uptime()
  });
});

// Detailed health check
app.get('/health/ready', async (req, res) => {
  try {
    await db.ping();
    await cache.ping();
    res.status(200).json({ status: 'ready' });
  } catch (error) {
    res.status(503).json({ status: 'not ready', error: error.message });
  }
});
```

```python
# Python Flask example
@app.route('/health')
def health():
    return {'status': 'healthy'}, 200

@app.route('/health/ready')
def ready():
    try:
        db.ping()
        return {'status': 'ready'}, 200
    except Exception as e:
        return {'status': 'not ready', 'error': str(e)}, 503
```

### 9.5 Health Check in Docker Compose

```yaml
version: '3.8'
services:
  app:
    image: myapp:latest
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  db:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
```

### 9.6 Dependent Service Health Checks

```yaml
version: '3.8'
services:
  app:
    image: myapp:latest
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

  db:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5
```

---

## 10. Logging Best Practices

### 10.1 Log to STDOUT/STDERR

```dockerfile
FROM nginx:alpine

# Redirect logs to stdout/stderr
RUN ln -sf /dev/stdout /var/log/nginx/access.log && \
    ln -sf /dev/stderr /var/log/nginx/error.log
```

```python
# Python application logging
import logging
import sys

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.StreamHandler(sys.stdout)
    ]
)
```

### 10.2 Structured Logging (JSON)

```python
import json
import logging
import sys

class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_record = {
            'timestamp': self.formatTime(record),
            'level': record.levelname,
            'message': record.getMessage(),
            'module': record.module,
            'function': record.funcName,
        }
        if record.exc_info:
            log_record['exception'] = self.formatException(record.exc_info)
        return json.dumps(log_record)

handler = logging.StreamHandler(sys.stdout)
handler.setFormatter(JSONFormatter())
logging.root.addHandler(handler)
logging.root.setLevel(logging.INFO)
```

```javascript
// Node.js with Winston
const winston = require('winston');

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.Console()
  ]
});
```

### 10.3 Docker Logging Drivers

```yaml
# docker-compose.yml
services:
  app:
    image: myapp:latest
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
        compress: "true"
```

```bash
# Available logging drivers
# json-file (default) - JSON file on disk
# syslog - Syslog server
# journald - systemd journal
# fluentd - Fluentd server
# awslogs - Amazon CloudWatch
# gcplogs - Google Cloud Logging
# splunk - Splunk server
```

### 10.4 Centralized Logging with ELK

```yaml
version: '3.8'
services:
  app:
    image: myapp:latest
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: app.logs

  fluentd:
    image: fluent/fluentd:v1.16-1
    volumes:
      - ./fluentd/conf:/fluentd/etc
    ports:
      - "24224:24224"

  elasticsearch:
    image: elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    volumes:
      - esdata:/usr/share/elasticsearch/data

  kibana:
    image: kibana:8.11.0
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

volumes:
  esdata:
```

### 10.5 Log Rotation

```yaml
# docker-compose.yml
services:
  app:
    image: myapp:latest
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "5"
```

```bash
# Daemon-level configuration (/etc/docker/daemon.json)
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
```

### 10.6 Log Levels and Environment

```dockerfile
ENV LOG_LEVEL=info

# Application respects LOG_LEVEL environment variable
```

```bash
# Override at runtime
docker run -e LOG_LEVEL=debug myapp:latest
```

---

## 11. Volume Best Practices

### 11.1 Named Volumes vs Bind Mounts

```yaml
version: '3.8'
services:
  db:
    image: postgres:15
    volumes:
      # Named volume - managed by Docker, persistent
      - postgres_data:/var/lib/postgresql/data

  app:
    image: myapp:latest
    volumes:
      # Bind mount - direct host path mapping
      - ./src:/app/src:ro
      # Named volume for cache
      - node_modules:/app/node_modules

volumes:
  postgres_data:
  node_modules:
```

### 11.2 Volume Permissions

```dockerfile
FROM node:20-alpine

WORKDIR /app

# Create user before volume mount
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001 -G nodejs

# Create directories with proper ownership
RUN mkdir -p /app/data && chown -R nextjs:nodejs /app

USER nextjs

VOLUME ["/app/data"]

CMD ["node", "server.js"]
```

### 11.3 Anonymous Volumes for Ephemeral Data

```dockerfile
# Dockerfile
FROM node:20-alpine

WORKDIR /app

COPY . .

# Anonymous volume for node_modules
VOLUME ["/app/node_modules"]

# Anonymous volume for temp data
VOLUME ["/tmp"]

CMD ["node", "server.js"]
```

### 11.4 Volume Backup Strategies

```bash
# Backup a volume
docker run --rm \
    -v postgres_data:/source:ro \
    -v $(pwd)/backup:/backup \
    alpine tar czf /backup/postgres_backup.tar.gz -C /source .

# Restore a volume
docker run --rm \
    -v postgres_data:/target \
    -v $(pwd)/backup:/backup:ro \
    alpine tar xzf /backup/postgres_backup.tar.gz -C /target
```

### 11.5 Read-Only Bind Mounts

```yaml
version: '3.8'
services:
  app:
    image: myapp:latest
    volumes:
      - ./config:/app/config:ro    # Read-only config
      - ./secrets:/run/secrets:ro  # Read-only secrets
      - app_data:/app/data         # Writable data
```

### 11.6 tmpfs Mounts for Sensitive Data

```yaml
version: '3.8'
services:
  app:
    image: myapp:latest
    tmpfs:
      - /tmp
      - /run
    volumes:
      - type: tmpfs
        target: /app/sensitive
        tmpfs:
          size: 100M
          mode: 1700
```

### 11.7 Volume Driver Options

```yaml
version: '3.8'
services:
  app:
    image: myapp:latest
    volumes:
      - type: volume
        source: app_data
        target: /app/data
        volume:
          nocopy: true

volumes:
  app_data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.1.100,rw
      device: ":/path/to/dir"
```

---

## 12. Networking Best Practices

### 12.1 Use Custom Networks

```yaml
version: '3.8'
services:
  frontend:
    image: nginx:alpine
    networks:
      - frontend_net
      - backend_net

  api:
    image: myapi:latest
    networks:
      - backend_net
      - db_net

  db:
    image: postgres:15
    networks:
      - db_net

networks:
  frontend_net:
  backend_net:
  db_net:
    internal: true  # No external access
```

### 12.2 Network Isolation

```yaml
version: '3.8'
services:
  public_app:
    image: nginx:alpine
    ports:
      - "80:80"
    networks:
      - public

  internal_api:
    image: api:latest
    networks:
      - public
      - internal

  database:
    image: postgres:15
    networks:
      - internal  # Only accessible from internal network

networks:
  public:
  internal:
    internal: true
```

### 12.3 DNS and Service Discovery

```yaml
version: '3.8'
services:
  api:
    image: myapi:latest
    # Other services can reach this as 'api' or 'api.mynetwork'
    networks:
      mynetwork:
        aliases:
          - backend
          - api-service

  web:
    image: nginx:alpine
    networks:
      - mynetwork
    # Can connect to api using: http://api:3000 or http://backend:3000

networks:
  mynetwork:
```

### 12.4 Port Mapping Best Practices

```yaml
version: '3.8'
services:
  web:
    image: nginx:alpine
    ports:
      # Bind to specific interface
      - "127.0.0.1:8080:80"
      # Bind to all interfaces
      - "443:443"
      # Random host port
      - "80"

  db:
    image: postgres:15
    # Don't expose database ports publicly
    # Only accessible via internal network
    expose:
      - "5432"
```

### 12.5 Network Performance Optimization

```yaml
version: '3.8'
services:
  app:
    image: myapp:latest
    networks:
      - app_net

networks:
  app_net:
    driver: bridge
    driver_opts:
      com.docker.network.driver.mtu: 1500
    ipam:
      driver: default
      config:
        - subnet: 172.28.0.0/16
```

### 12.6 Host Network Mode (When Needed)

```yaml
version: '3.8'
services:
  monitoring:
    image: prometheus:latest
    network_mode: host
    # Use only when necessary - reduces isolation
```

### 12.7 Network Security

```yaml
version: '3.8'
services:
  app:
    image: myapp:latest
    networks:
      - secure_net
    cap_drop:
      - NET_RAW
      - NET_BIND_SERVICE

networks:
  secure_net:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.enable_icc: "false"
```

---

## 13. Docker Compose Best Practices

### 13.1 Use Multiple Compose Files

```bash
# Base configuration
# docker-compose.yml

# Override for development
# docker-compose.override.yml (automatically loaded)

# Override for production
# docker-compose.prod.yml

# Usage
docker compose up                                    # Uses base + override
docker compose -f docker-compose.yml -f docker-compose.prod.yml up
```

```yaml
# docker-compose.yml (base)
version: '3.8'
services:
  app:
    image: myapp:latest
    environment:
      - NODE_ENV=production

# docker-compose.override.yml (development)
version: '3.8'
services:
  app:
    build: .
    volumes:
      - ./src:/app/src
    environment:
      - NODE_ENV=development
      - DEBUG=true

# docker-compose.prod.yml (production)
version: '3.8'
services:
  app:
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
```

### 13.2 Environment Variable Management

```yaml
version: '3.8'
services:
  app:
    image: myapp:latest
    env_file:
      - .env
      - .env.${ENVIRONMENT:-development}
    environment:
      - NODE_ENV=${NODE_ENV:-production}
      - DATABASE_URL
```

```bash
# .env
DATABASE_HOST=localhost
DATABASE_PORT=5432

# .env.production
DATABASE_HOST=prod-db.example.com
```

### 13.3 Dependency Management

```yaml
version: '3.8'
services:
  app:
    image: myapp:latest
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started

  db:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7
```

### 13.4 Resource Limits

```yaml
version: '3.8'
services:
  app:
    image: myapp:latest
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 1G
        reservations:
          cpus: '0.25'
          memory: 256M
```

### 13.5 Restart Policies

```yaml
version: '3.8'
services:
  web:
    image: nginx:alpine
    restart: unless-stopped

  worker:
    image: myworker:latest
    restart: on-failure:5

  cron:
    image: mycron:latest
    restart: "no"  # For one-off tasks
```

### 13.6 Profiles for Optional Services

```yaml
version: '3.8'
services:
  app:
    image: myapp:latest

  db:
    image: postgres:15

  adminer:
    image: adminer
    profiles:
      - debug
    ports:
      - "8080:8080"

  prometheus:
    image: prom/prometheus
    profiles:
      - monitoring
```

```bash
# Start without optional services
docker compose up

# Start with debug tools
docker compose --profile debug up

# Start with monitoring
docker compose --profile monitoring up
```

### 13.7 Extension Fields

```yaml
version: '3.8'

x-common-env: &common-env
  LOG_LEVEL: info
  TZ: UTC

x-healthcheck: &default-healthcheck
  interval: 30s
  timeout: 10s
  retries: 3

services:
  api:
    image: myapi:latest
    environment:
      <<: *common-env
      SERVICE_NAME: api
    healthcheck:
      <<: *default-healthcheck
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]

  worker:
    image: myworker:latest
    environment:
      <<: *common-env
      SERVICE_NAME: worker
    healthcheck:
      <<: *default-healthcheck
      test: ["CMD", "pgrep", "worker"]
```

### 13.8 Named Volumes with Labels

```yaml
version: '3.8'
services:
  db:
    image: postgres:15
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
    labels:
      com.example.description: "PostgreSQL data"
      com.example.department: "IT"
      com.example.backup: "daily"
```

---

## 14. Container Orchestration Considerations

### 14.1 Kubernetes-Ready Containers

```dockerfile
FROM node:20-alpine

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci --only=production

# Copy application
COPY . .

# Non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001 -G nodejs
USER nextjs

# Health check endpoint for Kubernetes probes
EXPOSE 3000

# Graceful shutdown handling
CMD ["node", "server.js"]
```

```javascript
// Graceful shutdown for Kubernetes
process.on('SIGTERM', () => {
  console.log('SIGTERM received, shutting down gracefully');
  server.close(() => {
    console.log('Process terminated');
    process.exit(0);
  });
});
```

### 14.2 Resource Requests and Limits

```yaml
# Kubernetes deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: app
          image: myapp:latest
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
```

### 14.3 Readiness and Liveness Probes

```yaml
# Kubernetes deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: app
          image: myapp:latest
          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
          startupProbe:
            httpGet:
              path: /health/startup
              port: 3000
            failureThreshold: 30
            periodSeconds: 10
```

### 14.4 ConfigMaps and Secrets

```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"
  API_TIMEOUT: "30"

---
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  DATABASE_PASSWORD: cGFzc3dvcmQxMjM=  # base64 encoded

---
# Deployment using ConfigMap and Secret
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: app
          image: myapp:latest
          envFrom:
            - configMapRef:
                name: app-config
          env:
            - name: DATABASE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: DATABASE_PASSWORD
```

### 14.5 Stateless Container Design

```dockerfile
# Design for horizontal scaling
FROM node:20-alpine

WORKDIR /app

COPY . .

# No local state - use external services
# - Redis for sessions
# - S3 for file storage
# - PostgreSQL for data

ENV SESSION_STORE=redis
ENV FILE_STORAGE=s3

CMD ["node", "server.js"]
```

### 14.6 Docker Swarm Considerations

```yaml
version: '3.8'
services:
  web:
    image: myapp:latest
    deploy:
      mode: replicated
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      rollback_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      placement:
        constraints:
          - node.role == worker
```

---

## 15. CI/CD Integration

### 15.1 GitHub Actions Docker Build

```yaml
# .github/workflows/docker.yml
name: Docker Build and Push

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=sha

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### 15.2 GitLab CI Docker Build

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - push

variables:
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE

build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $DOCKER_IMAGE:$CI_COMMIT_SHA .
    - docker push $DOCKER_IMAGE:$CI_COMMIT_SHA

test:
  stage: test
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker run $DOCKER_IMAGE:$CI_COMMIT_SHA npm test

push:
  stage: push
  image: docker:24
  services:
    - docker:24-dind
  only:
    - main
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker pull $DOCKER_IMAGE:$CI_COMMIT_SHA
    - docker tag $DOCKER_IMAGE:$CI_COMMIT_SHA $DOCKER_IMAGE:latest
    - docker push $DOCKER_IMAGE:latest
```

### 15.3 Multi-Platform Builds

```yaml
# GitHub Actions multi-platform build
- name: Build and push multi-platform
  uses: docker/build-push-action@v5
  with:
    context: .
    platforms: linux/amd64,linux/arm64
    push: true
    tags: ${{ steps.meta.outputs.tags }}
```

```bash
# Manual multi-platform build
docker buildx create --use
docker buildx build \
    --platform linux/amd64,linux/arm64 \
    -t myregistry/myapp:latest \
    --push .
```

### 15.4 Security Scanning in CI

```yaml
# GitHub Actions with Trivy
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'

- name: Upload Trivy scan results
  uses: github/codeql-action/upload-sarif@v2
  with:
    sarif_file: 'trivy-results.sarif'
```

### 15.5 Build Caching Strategies

```yaml
# GitHub Actions with cache
- name: Build with cache
  uses: docker/build-push-action@v5
  with:
    context: .
    cache-from: |
      type=gha
      type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache
    cache-to: type=gha,mode=max
```

### 15.6 Automated Testing in CI

```yaml
# Test container before push
test:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4

    - name: Build test image
      run: docker build -t myapp:test --target test .

    - name: Run tests
      run: docker run --rm myapp:test npm test

    - name: Run integration tests
      run: |
        docker compose -f docker-compose.test.yml up -d
        docker compose -f docker-compose.test.yml run --rm test npm run test:integration
        docker compose -f docker-compose.test.yml down
```

---

## 16. Monitoring and Debugging

### 16.1 Container Metrics

```bash
# Real-time stats
docker stats

# Format output
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"

# One-time stats
docker stats --no-stream
```

### 16.2 Prometheus Metrics

```yaml
# docker-compose.yml with Prometheus
version: '3.8'
services:
  app:
    image: myapp:latest
    ports:
      - "3000:3000"
    labels:
      - "prometheus.scrape=true"
      - "prometheus.port=3000"
      - "prometheus.path=/metrics"

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    volumes:
      - grafana_data:/var/lib/grafana

volumes:
  grafana_data:
```

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'docker'
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
    relabel_configs:
      - source_labels: [__meta_docker_container_label_prometheus_scrape]
        regex: true
        action: keep
```

### 16.3 Container Logs

```bash
# View logs
docker logs container_name

# Follow logs
docker logs -f container_name

# Last N lines
docker logs --tail 100 container_name

# Since timestamp
docker logs --since 2024-01-01T00:00:00 container_name

# With timestamps
docker logs -t container_name
```

### 16.4 Debugging Running Containers

```bash
# Execute shell in running container
docker exec -it container_name /bin/sh

# Execute command
docker exec container_name ls -la /app

# Inspect container
docker inspect container_name

# View processes
docker top container_name

# View resource usage
docker stats container_name
```

### 16.5 Debug Image Issues

```bash
# Run container with shell override
docker run -it --rm myimage:latest /bin/sh

# Run with different entrypoint
docker run -it --rm --entrypoint /bin/sh myimage:latest

# Build with debug output
docker build --progress=plain -t myimage:latest .

# Export filesystem for inspection
docker export container_name > container.tar
```

### 16.6 Network Debugging

```bash
# Inspect network
docker network inspect bridge

# Check container networking
docker exec container_name cat /etc/hosts
docker exec container_name cat /etc/resolv.conf

# Test connectivity
docker exec container_name ping other_container
docker exec container_name curl http://other_container:3000

# Check ports
docker port container_name
```

### 16.7 Health Check Debugging

```bash
# Check health status
docker inspect --format='{{.State.Health.Status}}' container_name

# View health check logs
docker inspect --format='{{range .State.Health.Log}}{{.Output}}{{end}}' container_name

# Manual health check
docker exec container_name wget --spider http://localhost:3000/health
```

### 16.8 Performance Profiling

```bash
# CPU and memory profile
docker stats --no-stream --format \
    "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}\t{{.NetIO}}\t{{.BlockIO}}"

# View container events
docker events --filter container=container_name

# Analyze image layers
docker history myimage:latest
dive myimage:latest
```

---

## 17. Common Patterns and Anti-Patterns

### 17.1 Pattern: Init Containers

```yaml
# docker-compose.yml
version: '3.8'
services:
  init-db:
    image: myapp:latest
    command: ["npm", "run", "db:migrate"]
    depends_on:
      db:
        condition: service_healthy

  app:
    image: myapp:latest
    depends_on:
      init-db:
        condition: service_completed_successfully

  db:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
```

### 17.2 Pattern: Sidecar Container

```yaml
version: '3.8'
services:
  app:
    image: myapp:latest
    volumes:
      - logs:/var/log/app

  log-shipper:
    image: fluent/fluent-bit:latest
    volumes:
      - logs:/var/log/app:ro
    depends_on:
      - app

volumes:
  logs:
```

### 17.3 Pattern: Ambassador Container

```yaml
version: '3.8'
services:
  app:
    image: myapp:latest
    environment:
      - DATABASE_HOST=db-proxy
      - DATABASE_PORT=5432

  db-proxy:
    image: haproxy:latest
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
```

### 17.4 Anti-Pattern: Running Multiple Processes

```dockerfile
# ANTI-PATTERN: Multiple processes in one container
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y nginx php-fpm
CMD service nginx start && php-fpm

# BETTER: Separate containers
# nginx/Dockerfile
FROM nginx:alpine
COPY nginx.conf /etc/nginx/nginx.conf

# php/Dockerfile
FROM php:8.2-fpm
COPY . /var/www/html
```

### 17.5 Anti-Pattern: Using Latest Tag

```dockerfile
# ANTI-PATTERN: Unpredictable builds
FROM node:latest
FROM python:latest

# CORRECT: Pin versions
FROM node:20.10.0-alpine3.18
FROM python:3.11.7-slim-bookworm
```

### 17.6 Anti-Pattern: Installing Unnecessary Packages

```dockerfile
# ANTI-PATTERN: Installing everything
RUN apt-get update && apt-get install -y \
    vim \
    curl \
    wget \
    build-essential \
    python3 \
    ...

# CORRECT: Install only what's needed
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        ca-certificates \
        curl && \
    rm -rf /var/lib/apt/lists/*
```

### 17.7 Anti-Pattern: Storing Secrets in Images

```dockerfile
# ANTI-PATTERN: Secrets in image
ENV API_KEY=secret123
COPY .env /app/.env

# CORRECT: Runtime injection
# docker run -e API_KEY=$API_KEY myapp
# Or use Docker secrets / external secret management
```

### 17.8 Anti-Pattern: Not Handling Signals

```javascript
// ANTI-PATTERN: Ignoring signals
// Container won't shut down gracefully

// CORRECT: Handle SIGTERM
process.on('SIGTERM', async () => {
  console.log('Received SIGTERM, shutting down gracefully');
  await server.close();
  await db.disconnect();
  process.exit(0);
});

process.on('SIGINT', async () => {
  console.log('Received SIGINT, shutting down gracefully');
  await server.close();
  await db.disconnect();
  process.exit(0);
});
```

### 17.9 Anti-Pattern: Large Build Context

```dockerfile
# ANTI-PATTERN: Sending entire directory as build context
# Without .dockerignore, this includes node_modules, .git, etc.

# CORRECT: Use .dockerignore
# .dockerignore
node_modules/
.git/
*.log
dist/
coverage/
```

### 17.10 Anti-Pattern: Not Using Multi-Stage Builds

```dockerfile
# ANTI-PATTERN: Build tools in production image
FROM node:20
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build
CMD ["node", "dist/server.js"]

# CORRECT: Multi-stage build
FROM node:20 AS builder
WORKDIR /app
COPY . .
RUN npm ci && npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/server.js"]
```

### 17.11 Pattern: Healthcheck with Dependencies

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY . .

RUN npm ci --only=production

# Health check that verifies database connectivity
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD node -e "require('./healthcheck').check()" || exit 1

CMD ["node", "server.js"]
```

```javascript
// healthcheck.js
const db = require('./db');
const redis = require('./redis');

async function check() {
  await db.ping();
  await redis.ping();
  console.log('Health check passed');
  process.exit(0);
}

check().catch((err) => {
  console.error('Health check failed:', err);
  process.exit(1);
});

module.exports = { check };
```

### 17.12 Pattern: Graceful Shutdown

```javascript
// Graceful shutdown pattern
const server = require('./server');
const db = require('./db');
const queue = require('./queue');

let isShuttingDown = false;

async function shutdown(signal) {
  if (isShuttingDown) return;
  isShuttingDown = true;

  console.log(`Received ${signal}, starting graceful shutdown`);

  // Stop accepting new requests
  server.close();

  // Wait for in-flight requests (with timeout)
  await Promise.race([
    new Promise(resolve => setTimeout(resolve, 30000)),
    waitForInflightRequests()
  ]);

  // Close database connections
  await db.disconnect();

  // Close queue connections
  await queue.close();

  console.log('Graceful shutdown completed');
  process.exit(0);
}

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));
```

### 17.13 Pattern: Configuration Hierarchy

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY . .

# Default configuration (can be overridden)
ENV NODE_ENV=production \
    PORT=3000 \
    LOG_LEVEL=info \
    LOG_FORMAT=json

# Configuration hierarchy:
# 1. Dockerfile defaults (lowest priority)
# 2. docker-compose.yml environment
# 3. .env file
# 4. Runtime -e flags (highest priority)

CMD ["node", "server.js"]
```

### 17.14 Pattern: Build Arguments for Customization

```dockerfile
FROM node:20-alpine

# Build-time customization
ARG NODE_ENV=production
ARG BUILD_DATE
ARG VCS_REF

# Labels for image metadata
LABEL org.opencontainers.image.created="${BUILD_DATE}" \
      org.opencontainers.image.revision="${VCS_REF}" \
      org.opencontainers.image.source="https://github.com/example/repo"

ENV NODE_ENV=${NODE_ENV}

WORKDIR /app

COPY . .

RUN npm ci --only=production

CMD ["node", "server.js"]
```

```bash
# Build with arguments
docker build \
    --build-arg BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ") \
    --build-arg VCS_REF=$(git rev-parse --short HEAD) \
    -t myapp:latest .
```

---

## Quick Reference

### Essential Dockerfile Instructions

| Instruction | Purpose | Best Practice |
|-------------|---------|---------------|
| `FROM` | Base image | Pin to specific version |
| `WORKDIR` | Working directory | Always set explicitly |
| `COPY` | Copy files | Prefer over ADD |
| `RUN` | Execute commands | Combine related commands |
| `ENV` | Environment variables | Use for runtime config |
| `ARG` | Build arguments | Use for build-time config |
| `EXPOSE` | Document ports | Informational only |
| `USER` | Run as user | Use non-root |
| `HEALTHCHECK` | Health monitoring | Always include |
| `CMD` | Default command | Use JSON array format |
| `ENTRYPOINT` | Container entrypoint | Use for fixed commands |

### Common Commands

```bash
# Build
docker build -t myimage:tag .
docker build --no-cache -t myimage:tag .
docker build --target stage -t myimage:tag .

# Run
docker run -d --name mycontainer myimage:tag
docker run -it --rm myimage:tag /bin/sh
docker run -e VAR=value -p 8080:80 myimage:tag

# Manage
docker ps -a
docker logs -f container
docker exec -it container /bin/sh
docker stop container
docker rm container

# Images
docker images
docker rmi image:tag
docker system prune -a

# Compose
docker compose up -d
docker compose down
docker compose logs -f
docker compose exec service /bin/sh
```

### Security Checklist

- [ ] Use official base images
- [ ] Pin image versions
- [ ] Use non-root user
- [ ] Scan for vulnerabilities
- [ ] Don't store secrets in images
- [ ] Use read-only filesystem where possible
- [ ] Drop unnecessary capabilities
- [ ] Use minimal base images
- [ ] Keep images updated
- [ ] Sign images for production

---

## References

- [Docker Documentation](https://docs.docker.com/)
- [Dockerfile Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Docker Security](https://docs.docker.com/engine/security/)
- [Docker Compose](https://docs.docker.com/compose/)
- [BuildKit](https://docs.docker.com/build/buildkit/)
- [Multi-platform Builds](https://docs.docker.com/build/building/multi-platform/)
