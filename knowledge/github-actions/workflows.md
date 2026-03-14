# GitHub Actions Workflows Reference

Comprehensive documentation for GitHub Actions workflows.

**Official Documentation:** https://docs.github.com/en/actions/using-workflows

---

## Table of Contents

1. [Workflow Basics](#workflow-basics)
2. [Triggers](#triggers)
3. [Jobs](#jobs)
4. [Steps](#steps)
5. [Environment Variables](#environment-variables)
6. [Secrets](#secrets)
7. [Matrix Builds](#matrix-builds)
8. [Caching](#caching)
9. [Artifacts](#artifacts)
10. [Reusable Workflows](#reusable-workflows)
11. [Common Patterns](#common-patterns)

---

## Workflow Basics

### File Location

```
.github/
└── workflows/
    ├── ci.yml
    ├── deploy.yml
    └── release.yml
```

### Basic Structure

```yaml
name: CI Pipeline

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test
```

---

## Triggers

### Push and Pull Request

```yaml
on:
  push:
    branches:
      - main
      - 'release/**'
    paths:
      - 'src/**'
      - 'package.json'
    paths-ignore:
      - '**.md'
      - 'docs/**'

  pull_request:
    branches:
      - main
    types:
      - opened
      - synchronize
      - reopened
```

### Schedule (Cron)

```yaml
on:
  schedule:
    # Every day at 2am UTC
    - cron: '0 2 * * *'
    # Every Monday at 9am UTC
    - cron: '0 9 * * 1'
```

### Manual Dispatch

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deployment environment'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
      debug:
        description: 'Enable debug mode'
        required: false
        type: boolean
        default: false
```

### Repository Events

```yaml
on:
  release:
    types: [published]

  issues:
    types: [opened, labeled]

  issue_comment:
    types: [created]

  workflow_run:
    workflows: ["Build"]
    types: [completed]

  workflow_call:
    inputs:
      config:
        required: true
        type: string
```

---

## Jobs

### Basic Job

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run build
```

### Job Dependencies

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build

  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  deploy:
    needs: [build, test]
    runs-on: ubuntu-latest
    steps:
      - run: npm run deploy
```

### Conditional Jobs

```yaml
jobs:
  deploy:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - run: npm run deploy

  notify:
    if: failure()
    needs: [build, test]
    runs-on: ubuntu-latest
    steps:
      - run: echo "Build failed!"
```

### Job Outputs

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.value }}
    steps:
      - id: version
        run: echo "value=$(cat package.json | jq -r .version)" >> $GITHUB_OUTPUT

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying version ${{ needs.build.outputs.version }}"
```

### Runner Types

```yaml
jobs:
  linux:
    runs-on: ubuntu-latest    # or ubuntu-22.04, ubuntu-20.04

  macos:
    runs-on: macos-latest     # or macos-14, macos-13

  windows:
    runs-on: windows-latest   # or windows-2022, windows-2019

  self-hosted:
    runs-on: [self-hosted, linux, x64]
```

---

## Steps

### Using Actions

```yaml
steps:
  # Use action from marketplace
  - uses: actions/checkout@v4

  # With inputs
  - uses: actions/setup-node@v4
    with:
      node-version: '20'
      cache: 'npm'

  # Use action from another repo
  - uses: owner/repo@v1

  # Use action from local path
  - uses: ./.github/actions/my-action
```

### Running Commands

```yaml
steps:
  # Single command
  - run: npm test

  # Multiple commands
  - run: |
      npm ci
      npm run build
      npm test

  # With name
  - name: Install and build
    run: |
      npm ci
      npm run build

  # With working directory
  - name: Build frontend
    working-directory: ./frontend
    run: npm run build

  # With shell
  - name: Run script
    shell: bash
    run: ./scripts/build.sh

  # PowerShell on Windows
  - name: Windows command
    shell: pwsh
    run: Get-Process
```

### Conditional Steps

```yaml
steps:
  # Always run
  - run: echo "Always runs"

  # Only on main branch
  - if: github.ref == 'refs/heads/main'
    run: npm run deploy

  # Only on pull request
  - if: github.event_name == 'pull_request'
    run: npm run lint

  # On success
  - if: success()
    run: echo "Previous steps succeeded"

  # On failure
  - if: failure()
    run: echo "A previous step failed"

  # Always (even on failure)
  - if: always()
    run: npm run cleanup

  # Cancelled
  - if: cancelled()
    run: echo "Workflow was cancelled"
```

---

## Environment Variables

### Defining Variables

```yaml
env:
  # Workflow level
  NODE_ENV: production
  CI: true

jobs:
  build:
    env:
      # Job level
      BUILD_MODE: release
    runs-on: ubuntu-latest
    steps:
      - env:
          # Step level
          STEP_VAR: value
        run: echo $STEP_VAR
```

### Built-in Variables

```yaml
steps:
  - run: |
      echo "Repository: ${{ github.repository }}"
      echo "Branch: ${{ github.ref_name }}"
      echo "SHA: ${{ github.sha }}"
      echo "Actor: ${{ github.actor }}"
      echo "Event: ${{ github.event_name }}"
      echo "Run ID: ${{ github.run_id }}"
      echo "Run Number: ${{ github.run_number }}"
      echo "Workspace: ${{ github.workspace }}"
```

### Setting Output Variables

```yaml
steps:
  - id: set-output
    run: |
      echo "version=1.0.0" >> $GITHUB_OUTPUT
      echo "date=$(date +'%Y-%m-%d')" >> $GITHUB_OUTPUT

  - run: |
      echo "Version: ${{ steps.set-output.outputs.version }}"
      echo "Date: ${{ steps.set-output.outputs.date }}"
```

### Environment Files

```yaml
steps:
  # Set environment variable for subsequent steps
  - run: echo "MY_VAR=value" >> $GITHUB_ENV

  - run: echo $MY_VAR  # Outputs: value

  # Multiline value
  - run: |
      echo "JSON_DATA<<EOF" >> $GITHUB_ENV
      cat data.json >> $GITHUB_ENV
      echo "EOF" >> $GITHUB_ENV

  # Add to PATH
  - run: echo "/custom/bin" >> $GITHUB_PATH
```

---

## Secrets

### Using Secrets

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        env:
          API_KEY: ${{ secrets.API_KEY }}
          AWS_ACCESS_KEY: ${{ secrets.AWS_ACCESS_KEY_ID }}
        run: ./deploy.sh

      - name: Docker login
        run: |
          echo "${{ secrets.DOCKER_PASSWORD }}" | \
          docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin
```

### Environment Secrets

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production  # Uses production secrets
    steps:
      - run: echo ${{ secrets.PROD_API_KEY }}
```

### GITHUB_TOKEN

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      packages: write
    steps:
      - uses: actions/checkout@v4

      - name: Create Release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: gh release create v1.0.0

      - name: Push to registry
        run: |
          echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
          docker push ghcr.io/${{ github.repository }}:latest
```

---

## Matrix Builds

### Basic Matrix

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18, 20, 22]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm test
```

### Multi-Dimensional Matrix

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node-version: [18, 20]
        # Runs 6 jobs: 3 OS x 2 Node versions
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
```

### Include/Exclude

```yaml
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
        node-version: [18, 20]
        exclude:
          # Don't run Node 18 on Windows
          - os: windows-latest
            node-version: 18
        include:
          # Add specific configuration
          - os: ubuntu-latest
            node-version: 20
            experimental: true
```

### Fail-Fast

```yaml
jobs:
  test:
    strategy:
      fail-fast: false  # Continue other jobs if one fails
      matrix:
        node-version: [18, 20, 22]
```

### Max Parallel

```yaml
jobs:
  test:
    strategy:
      max-parallel: 2  # Run max 2 jobs at a time
      matrix:
        version: [1, 2, 3, 4, 5]
```

---

## Caching

### npm Cache

```yaml
steps:
  - uses: actions/checkout@v4

  - uses: actions/setup-node@v4
    with:
      node-version: '20'
      cache: 'npm'

  - run: npm ci
```

### Custom Cache

```yaml
steps:
  - uses: actions/cache@v4
    with:
      path: |
        ~/.npm
        node_modules
      key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
      restore-keys: |
        ${{ runner.os }}-node-

  - run: npm ci
```

### Cache with Fallback

```yaml
steps:
  - uses: actions/cache@v4
    id: cache
    with:
      path: node_modules
      key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}

  - if: steps.cache.outputs.cache-hit != 'true'
    run: npm ci
```

---

## Artifacts

### Upload Artifact

```yaml
steps:
  - run: npm run build

  - uses: actions/upload-artifact@v4
    with:
      name: build-output
      path: dist/
      retention-days: 5
```

### Download Artifact

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/

      - run: ls -la dist/
```

### Multiple Artifacts

```yaml
steps:
  - uses: actions/upload-artifact@v4
    with:
      name: test-results
      path: |
        coverage/
        test-results.xml
        !coverage/tmp/
```

---

## Reusable Workflows

### Defining Reusable Workflow

```yaml
# .github/workflows/reusable-build.yml
name: Reusable Build

on:
  workflow_call:
    inputs:
      node-version:
        required: false
        type: string
        default: '20'
      environment:
        required: true
        type: string
    secrets:
      npm-token:
        required: true
    outputs:
      version:
        description: "Build version"
        value: ${{ jobs.build.outputs.version }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.value }}
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}

      - run: npm ci
        env:
          NPM_TOKEN: ${{ secrets.npm-token }}

      - id: version
        run: echo "value=$(npm pkg get version | tr -d '\"')" >> $GITHUB_OUTPUT
```

### Calling Reusable Workflow

```yaml
name: CI

on: [push]

jobs:
  build:
    uses: ./.github/workflows/reusable-build.yml
    with:
      node-version: '20'
      environment: production
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying version ${{ needs.build.outputs.version }}"
```

---

## Common Patterns

### Node.js CI

```yaml
name: Node.js CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18, 20]

    steps:
      - uses: actions/checkout@v4

      - name: Use Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - run: npm ci
      - run: npm run build
      - run: npm test

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
```

### Docker Build and Push

```yaml
name: Docker

on:
  push:
    branches: [main]
    tags: ['v*']

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

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### Deploy to Cloud

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Deploy to S3
        run: aws s3 sync ./dist s3://${{ secrets.S3_BUCKET }}

      - name: Invalidate CloudFront
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CF_DISTRIBUTION_ID }} \
            --paths "/*"
```

### Release Workflow

```yaml
name: Release

on:
  push:
    tags: ['v*']

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - uses: actions/checkout@v4

      - name: Build
        run: npm run build

      - name: Create Release
        uses: softprops/action-gh-release@v1
        with:
          files: |
            dist/*.js
            dist/*.css
          generate_release_notes: true
```

---

## Quick Reference

### Triggers

| Event | Description |
|-------|-------------|
| `push` | On push to repo |
| `pull_request` | On PR events |
| `schedule` | Cron schedule |
| `workflow_dispatch` | Manual trigger |
| `release` | On release events |
| `workflow_call` | Reusable workflow |

### Contexts

| Context | Description |
|---------|-------------|
| `github.*` | Workflow/repo info |
| `env.*` | Environment variables |
| `secrets.*` | Repository secrets |
| `steps.*` | Step outputs |
| `needs.*` | Job outputs |
| `matrix.*` | Matrix values |

### Expressions

| Expression | Description |
|------------|-------------|
| `${{ ... }}` | Expression syntax |
| `success()` | Previous steps passed |
| `failure()` | A step failed |
| `always()` | Always run |
| `cancelled()` | Workflow cancelled |
| `contains(s, 'x')` | String contains |
| `startsWith(s, 'x')` | String starts with |
| `hashFiles('**/*.lock')` | Hash of files |
