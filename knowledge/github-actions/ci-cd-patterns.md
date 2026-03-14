# GitHub Actions CI/CD Patterns & Security

Advanced patterns for production-grade CI/CD pipelines with security best practices.

**Official Documentation:** https://docs.github.com/en/actions/security-guides

---

## Table of Contents

1. [Security Best Practices](#security-best-practices)
2. [Advanced CI Patterns](#advanced-ci-patterns)
3. [Deployment Strategies](#deployment-strategies)
4. [Monorepo Patterns](#monorepo-patterns)
5. [Testing Strategies](#testing-strategies)
6. [Release Automation](#release-automation)
7. [Composite Actions](#composite-actions)
8. [Self-Hosted Runners](#self-hosted-runners)
9. [Cost Optimization](#cost-optimization)
10. [Debugging & Troubleshooting](#debugging--troubleshooting)

---

## Security Best Practices

### Principle of Least Privilege

```yaml
# Always specify minimal permissions
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read      # Read repo only
      packages: write     # Push to GHCR
      id-token: write     # OIDC for cloud auth

    steps:
      - uses: actions/checkout@v4
```

### Permission Reference

```yaml
permissions:
  actions: read|write|none
  checks: read|write|none
  contents: read|write|none
  deployments: read|write|none
  id-token: write|none           # OIDC tokens
  issues: read|write|none
  packages: read|write|none
  pages: read|write|none
  pull-requests: read|write|none
  repository-projects: read|write|none
  security-events: read|write|none
  statuses: read|write|none

# Disable all permissions at workflow level
permissions: {}
```

### Secret Management

```yaml
# ❌ Bad: Secrets in logs
- run: echo ${{ secrets.API_KEY }}

# ✅ Good: Mask secrets properly
- run: |
    echo "::add-mask::$SECRET_VALUE"
    ./deploy.sh
  env:
    SECRET_VALUE: ${{ secrets.API_KEY }}

# ✅ Better: Use environment secrets with approvals
jobs:
  deploy:
    environment:
      name: production
      url: https://myapp.com
    steps:
      - run: ./deploy.sh
        env:
          API_KEY: ${{ secrets.PROD_API_KEY }}
```

### OIDC Authentication (Keyless)

```yaml
# AWS without long-lived credentials
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions
          aws-region: us-east-1
          # No secrets needed - uses OIDC

      - run: aws s3 ls

# Google Cloud with OIDC
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: 'projects/123/locations/global/workloadIdentityPools/pool/providers/github'
          service_account: 'sa@project.iam.gserviceaccount.com'

      - uses: google-github-actions/setup-gcloud@v2
      - run: gcloud compute instances list

# Azure with OIDC
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

### Preventing Script Injection

```yaml
# ❌ Vulnerable to injection
- run: echo "Title: ${{ github.event.issue.title }}"

# ✅ Safe: Use environment variable
- run: echo "Title: $TITLE"
  env:
    TITLE: ${{ github.event.issue.title }}

# ❌ Vulnerable in PR context
- run: |
    comment="${{ github.event.comment.body }}"
    if [[ "$comment" == "/deploy" ]]; then
      ./deploy.sh
    fi

# ✅ Safe: Validate input
- run: |
    if [[ "$COMMENT" =~ ^/deploy$ ]]; then
      ./deploy.sh
    fi
  env:
    COMMENT: ${{ github.event.comment.body }}
```

### Fork Security

```yaml
# Prevent secrets exposure in fork PRs
on:
  pull_request_target:
    types: [labeled]

jobs:
  build:
    # Only run if labeled by maintainer
    if: contains(github.event.pull_request.labels.*.name, 'safe-to-test')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          # Checkout PR head safely
          ref: ${{ github.event.pull_request.head.sha }}

      - run: npm test
        env:
          API_KEY: ${{ secrets.API_KEY }}
```

### Dependency Security

```yaml
name: Security Scan

on:
  push:
    branches: [main]
  pull_request:
  schedule:
    - cron: '0 0 * * 1'  # Weekly

jobs:
  dependency-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Dependency Review
        uses: actions/dependency-review-action@v4
        with:
          fail-on-severity: moderate
          deny-licenses: GPL-3.0, AGPL-3.0

  codeql:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: javascript, typescript
          queries: security-extended

      - name: Autobuild
        uses: github/codeql-action/autobuild@v3

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3

  trivy-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
```

---

## Advanced CI Patterns

### Path-Based Filtering

```yaml
name: Smart CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      frontend: ${{ steps.filter.outputs.frontend }}
      backend: ${{ steps.filter.outputs.backend }}
      docs: ${{ steps.filter.outputs.docs }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            frontend:
              - 'frontend/**'
              - 'package.json'
            backend:
              - 'backend/**'
              - 'requirements.txt'
            docs:
              - 'docs/**'
              - '*.md'

  frontend-ci:
    needs: changes
    if: needs.changes.outputs.frontend == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm test
        working-directory: frontend

  backend-ci:
    needs: changes
    if: needs.changes.outputs.backend == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install -r requirements.txt && pytest
        working-directory: backend
```

### Concurrency Control

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

# Cancel previous runs for same PR/branch
concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test
```

### Required Status Checks

```yaml
# Workflow that always reports status
name: CI Required

on:
  push:
    branches: [main]
  pull_request:

jobs:
  # Determine what needs to run
  changes:
    runs-on: ubuntu-latest
    outputs:
      should-test: ${{ steps.filter.outputs.src }}
    steps:
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            src:
              - 'src/**'
              - 'package.json'

  # Actual test job
  test:
    needs: changes
    if: needs.changes.outputs.should-test == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test

  # Required check that always passes
  ci-status:
    runs-on: ubuntu-latest
    needs: [changes, test]
    if: always()
    steps:
      - name: Check CI status
        run: |
          if [[ "${{ needs.test.result }}" == "failure" ]]; then
            exit 1
          fi
          echo "CI passed or was skipped"
```

### Service Containers

```yaml
jobs:
  integration-test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Run integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379
```

### Dynamic Matrix

```yaml
jobs:
  prepare:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
      - uses: actions/checkout@v4

      - id: set-matrix
        run: |
          # Generate matrix from package.json workspaces
          PACKAGES=$(cat package.json | jq -c '.workspaces')
          echo "matrix={\"package\":$PACKAGES}" >> $GITHUB_OUTPUT

  build:
    needs: prepare
    runs-on: ubuntu-latest
    strategy:
      matrix: ${{ fromJson(needs.prepare.outputs.matrix) }}
    steps:
      - uses: actions/checkout@v4
      - run: npm run build -w ${{ matrix.package }}
```

---

## Deployment Strategies

### Blue-Green Deployment

```yaml
name: Blue-Green Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to inactive slot
        id: deploy
        run: |
          # Get current active slot
          ACTIVE=$(aws ssm get-parameter --name /app/active-slot --query Parameter.Value --output text)
          if [ "$ACTIVE" == "blue" ]; then
            INACTIVE="green"
          else
            INACTIVE="blue"
          fi
          echo "inactive=$INACTIVE" >> $GITHUB_OUTPUT

          # Deploy to inactive slot
          aws ecs update-service \
            --cluster production \
            --service app-$INACTIVE \
            --task-definition app:latest \
            --force-new-deployment

          # Wait for deployment
          aws ecs wait services-stable \
            --cluster production \
            --services app-$INACTIVE

      - name: Run smoke tests
        run: |
          SLOT="${{ steps.deploy.outputs.inactive }}"
          ./scripts/smoke-test.sh "https://$SLOT.internal.app.com"

      - name: Switch traffic
        run: |
          SLOT="${{ steps.deploy.outputs.inactive }}"

          # Update load balancer target group
          aws elbv2 modify-listener \
            --listener-arn ${{ secrets.ALB_LISTENER_ARN }} \
            --default-actions Type=forward,TargetGroupArn=${{ secrets[format('{0}_TG_ARN', steps.deploy.outputs.inactive)] }}

          # Update active slot parameter
          aws ssm put-parameter \
            --name /app/active-slot \
            --value "$SLOT" \
            --overwrite

      - name: Notify
        if: always()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Deployment ${{ job.status }}: ${{ github.repository }}@${{ github.sha }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### Canary Deployment

```yaml
name: Canary Deploy

on:
  push:
    branches: [main]

jobs:
  canary:
    runs-on: ubuntu-latest
    environment: production

    steps:
      - uses: actions/checkout@v4

      - name: Deploy canary (10%)
        run: |
          kubectl set image deployment/app \
            app=ghcr.io/${{ github.repository }}:${{ github.sha }} \
            --record

          kubectl patch deployment app -p '
            {"spec": {"strategy": {"rollingUpdate": {"maxSurge": "10%", "maxUnavailable": 0}}}}'

          kubectl rollout status deployment/app --timeout=5m

      - name: Monitor canary
        run: |
          # Monitor for 10 minutes
          for i in {1..20}; do
            ERROR_RATE=$(curl -s "http://prometheus:9090/api/v1/query?query=rate(http_requests_total{status=~'5..'}[1m])/rate(http_requests_total[1m])" | jq '.data.result[0].value[1]' | tr -d '"')

            if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
              echo "Error rate too high: $ERROR_RATE"
              kubectl rollout undo deployment/app
              exit 1
            fi

            sleep 30
          done

      - name: Complete rollout
        run: |
          kubectl scale deployment/app --replicas=10
          kubectl rollout status deployment/app --timeout=10m
```

### Rolling Deployment with Approval

```yaml
name: Production Deploy

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to deploy'
        required: true

jobs:
  staging:
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - run: ./deploy.sh staging ${{ inputs.version }}

  integration-tests:
    needs: staging
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run test:e2e -- --env staging

  # Manual approval gate via environment
  production-approval:
    needs: integration-tests
    runs-on: ubuntu-latest
    environment:
      name: production-approval
      # Requires manual approval in GitHub
    steps:
      - run: echo "Approved for production"

  production:
    needs: production-approval
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://app.com
    steps:
      - uses: actions/checkout@v4
      - run: ./deploy.sh production ${{ inputs.version }}
```

### GitOps with ArgoCD

```yaml
name: GitOps Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.build.outputs.tag }}
    steps:
      - uses: actions/checkout@v4

      - name: Build and push
        id: build
        run: |
          TAG="${{ github.sha }}"
          docker build -t ghcr.io/${{ github.repository }}:$TAG .
          docker push ghcr.io/${{ github.repository }}:$TAG
          echo "tag=$TAG" >> $GITHUB_OUTPUT

  update-manifests:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          repository: org/kubernetes-manifests
          token: ${{ secrets.MANIFEST_REPO_TOKEN }}

      - name: Update image tag
        run: |
          cd apps/myapp/overlays/production
          kustomize edit set image app=ghcr.io/${{ github.repository }}:${{ needs.build.outputs.image-tag }}

      - name: Commit and push
        run: |
          git config user.name github-actions
          git config user.email github-actions@github.com
          git add .
          git commit -m "Update myapp to ${{ needs.build.outputs.image-tag }}"
          git push
```

---

## Monorepo Patterns

### Nx Affected

```yaml
name: Nx CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  affected:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Get affected projects
        id: affected
        run: |
          if [ "${{ github.event_name }}" == "pull_request" ]; then
            BASE="origin/${{ github.base_ref }}"
          else
            BASE="HEAD~1"
          fi

          AFFECTED=$(npx nx show projects --affected --base=$BASE | tr '\n' ',')
          echo "projects=$AFFECTED" >> $GITHUB_OUTPUT

      - name: Test affected
        run: npx nx affected --target=test --base=$BASE

      - name: Build affected
        run: npx nx affected --target=build --base=$BASE

      - name: Lint affected
        run: npx nx affected --target=lint --base=$BASE
```

### Turborepo

```yaml
name: Turborepo CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Cache turbo
        uses: actions/cache@v4
        with:
          path: .turbo
          key: ${{ runner.os }}-turbo-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-turbo-

      - run: npm ci

      - name: Build
        run: npx turbo run build --filter='...[HEAD^1]'

      - name: Test
        run: npx turbo run test --filter='...[HEAD^1]'
```

### Workspace-Based Pipeline

```yaml
name: Monorepo CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      packages: ${{ steps.filter.outputs.changes }}
    steps:
      - uses: actions/checkout@v4

      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            web:
              - 'packages/web/**'
            api:
              - 'packages/api/**'
            shared:
              - 'packages/shared/**'

  build:
    needs: changes
    if: needs.changes.outputs.packages != '[]'
    runs-on: ubuntu-latest
    strategy:
      matrix:
        package: ${{ fromJson(needs.changes.outputs.packages) }}
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Build package
        run: npm run build -w packages/${{ matrix.package }}

      - name: Test package
        run: npm test -w packages/${{ matrix.package }}
```

---

## Testing Strategies

### Test Sharding

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Run tests (shard ${{ matrix.shard }}/4)
        run: npx playwright test --shard=${{ matrix.shard }}/4

      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results-${{ matrix.shard }}
          path: test-results/
          retention-days: 7

  merge-results:
    needs: test
    if: always()
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          pattern: test-results-*
          merge-multiple: true
          path: all-results/

      - name: Generate report
        run: npx playwright merge-reports all-results/
```

### Flaky Test Retry

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - run: npm ci

      - name: Run tests with retry
        uses: nick-fields/retry@v3
        with:
          timeout_minutes: 10
          max_attempts: 3
          command: npm test
          retry_on: error
```

### Visual Regression Testing

```yaml
jobs:
  visual-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - run: npm ci

      - name: Build Storybook
        run: npm run build-storybook

      - name: Run Chromatic
        uses: chromaui/action@latest
        with:
          projectToken: ${{ secrets.CHROMATIC_PROJECT_TOKEN }}
          exitZeroOnChanges: true
          autoAcceptChanges: main
```

### Coverage Enforcement

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - run: npm ci

      - name: Run tests with coverage
        run: npm run test:coverage

      - name: Check coverage threshold
        run: |
          COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "Coverage $COVERAGE% is below 80% threshold"
            exit 1
          fi

      - name: Upload to Codecov
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          fail_ci_if_error: true
          flags: unittests
```

---

## Release Automation

### Semantic Release

```yaml
name: Release

on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      packages: write
      issues: write
      pull-requests: write

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          persist-credentials: false

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Semantic Release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
        run: npx semantic-release
```

### Changesets

```yaml
name: Release

on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - run: npm ci

      - name: Create Release PR or Publish
        uses: changesets/action@v1
        with:
          publish: npm run release
          version: npm run version
          commit: 'chore: version packages'
          title: 'chore: version packages'
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### Release Please

```yaml
name: Release

on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    outputs:
      release_created: ${{ steps.release.outputs.release_created }}
      tag_name: ${{ steps.release.outputs.tag_name }}
    steps:
      - uses: google-github-actions/release-please-action@v4
        id: release
        with:
          release-type: node
          package-name: my-package

  publish:
    needs: release
    if: needs.release.outputs.release_created
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          registry-url: 'https://registry.npmjs.org'

      - run: npm ci
      - run: npm publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

---

## Composite Actions

### Creating Composite Action

```yaml
# .github/actions/setup-project/action.yml
name: 'Setup Project'
description: 'Setup Node.js and install dependencies'

inputs:
  node-version:
    description: 'Node.js version'
    required: false
    default: '20'
  install-command:
    description: 'Install command'
    required: false
    default: 'npm ci'

outputs:
  cache-hit:
    description: 'Whether cache was hit'
    value: ${{ steps.cache.outputs.cache-hit }}

runs:
  using: 'composite'
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}

    - name: Cache dependencies
      id: cache
      uses: actions/cache@v4
      with:
        path: node_modules
        key: ${{ runner.os }}-node-${{ inputs.node-version }}-${{ hashFiles('**/package-lock.json') }}

    - name: Install dependencies
      if: steps.cache.outputs.cache-hit != 'true'
      shell: bash
      run: ${{ inputs.install-command }}
```

### Using Composite Action

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: ./.github/actions/setup-project
        with:
          node-version: '20'

      - run: npm run build
```

### Docker-Based Action

```yaml
# .github/actions/custom-lint/action.yml
name: 'Custom Linter'
description: 'Run custom linting rules'

inputs:
  config:
    description: 'Config file path'
    required: false
    default: '.lintrc'

runs:
  using: 'docker'
  image: 'Dockerfile'
  args:
    - ${{ inputs.config }}
```

---

## Self-Hosted Runners

### Runner Setup

```yaml
# Kubernetes runner deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: github-runner
spec:
  replicas: 3
  selector:
    matchLabels:
      app: github-runner
  template:
    metadata:
      labels:
        app: github-runner
    spec:
      containers:
        - name: runner
          image: myrunner:latest
          env:
            - name: RUNNER_TOKEN
              valueFrom:
                secretKeyRef:
                  name: github-runner
                  key: token
            - name: RUNNER_REPOSITORY_URL
              value: https://github.com/org/repo
            - name: RUNNER_LABELS
              value: self-hosted,linux,x64,k8s
          resources:
            requests:
              cpu: 2
              memory: 4Gi
            limits:
              cpu: 4
              memory: 8Gi
```

### Runner Scale Set (ARC)

```yaml
# Actions Runner Controller
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: runner-deployment
spec:
  replicas: 1
  template:
    spec:
      repository: org/repo
      labels:
        - self-hosted
        - linux
        - x64

---
apiVersion: actions.summerwind.dev/v1alpha1
kind: HorizontalRunnerAutoscaler
metadata:
  name: runner-autoscaler
spec:
  scaleTargetRef:
    name: runner-deployment
  minReplicas: 1
  maxReplicas: 10
  scaleDownDelaySecondsAfterScaleOut: 300
  metrics:
    - type: PercentageRunnersBusy
      scaleUpThreshold: '0.75'
      scaleDownThreshold: '0.25'
      scaleUpFactor: '2'
      scaleDownFactor: '0.5'
```

### Using Self-Hosted Runners

```yaml
jobs:
  build:
    runs-on: [self-hosted, linux, x64]
    steps:
      - uses: actions/checkout@v4
      - run: ./build.sh

  # With specific labels
  gpu-job:
    runs-on: [self-hosted, linux, gpu]
    steps:
      - run: nvidia-smi
      - run: python train.py
```

---

## Cost Optimization

### Efficient Caching

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Use setup action with built-in cache
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      # Only install if cache miss
      - run: npm ci

      # Cache build outputs
      - uses: actions/cache@v4
        with:
          path: |
            .next/cache
            dist
          key: ${{ runner.os }}-build-${{ hashFiles('src/**') }}
```

### Skip Unnecessary Runs

```yaml
on:
  push:
    branches: [main]
    paths-ignore:
      - '**.md'
      - 'docs/**'
      - '.github/ISSUE_TEMPLATE/**'

jobs:
  build:
    # Skip if commit message contains [skip ci]
    if: "!contains(github.event.head_commit.message, '[skip ci]')"
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
```

### Smaller Runners

```yaml
jobs:
  lint:
    # Use smaller runner for light tasks
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run lint

  build:
    # Use larger runner only when needed
    runs-on: ubuntu-latest-16-cores  # If available
    steps:
      - uses: actions/checkout@v4
      - run: npm run build
```

---

## Debugging & Troubleshooting

### Debug Logging

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Enable debug output
      - name: Debug info
        run: |
          echo "Event: ${{ github.event_name }}"
          echo "Ref: ${{ github.ref }}"
          echo "SHA: ${{ github.sha }}"
          echo "Actor: ${{ github.actor }}"
          echo "Workflow: ${{ github.workflow }}"
          cat $GITHUB_EVENT_PATH | jq '.'

      # Dump all contexts
      - name: Dump contexts
        env:
          GITHUB_CONTEXT: ${{ toJson(github) }}
          JOB_CONTEXT: ${{ toJson(job) }}
          STEPS_CONTEXT: ${{ toJson(steps) }}
        run: |
          echo "GitHub context:"
          echo "$GITHUB_CONTEXT"
```

### SSH Debug Session

```yaml
jobs:
  debug:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup tmate session
        if: failure()
        uses: mxschmitt/action-tmate@v3
        with:
          limit-access-to-actor: true
          timeout-minutes: 15
```

### Artifact Debugging

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - run: npm run build

      # Upload logs on failure
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: debug-logs
          path: |
            npm-debug.log
            **/*.log
            **/coverage
          retention-days: 5
```

### Common Issues

```yaml
# Issue: Checkout not getting latest
- uses: actions/checkout@v4
  with:
    fetch-depth: 0  # Get full history
    ref: ${{ github.head_ref }}  # PR head

# Issue: Permission denied
- run: chmod +x ./scripts/*.sh

# Issue: Node modules not found
- run: npm ci  # Use ci, not install

# Issue: Docker layer caching
- uses: docker/build-push-action@v5
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max

# Issue: Git operations failing
- uses: actions/checkout@v4
  with:
    persist-credentials: true
    token: ${{ secrets.PAT }}
```

---

## Quick Reference

### Status Functions

| Function | Description |
|----------|-------------|
| `success()` | All previous steps passed |
| `failure()` | Any previous step failed |
| `always()` | Always runs |
| `cancelled()` | Workflow was cancelled |

### Useful Expressions

```yaml
# Check branch
if: github.ref == 'refs/heads/main'

# Check event
if: github.event_name == 'pull_request'

# Check actor
if: github.actor == 'dependabot[bot]'

# Boolean from string
if: ${{ inputs.debug == 'true' }}

# Contains check
if: contains(github.event.pull_request.labels.*.name, 'deploy')

# Check output
if: steps.changes.outputs.frontend == 'true'

# Multiple conditions
if: github.ref == 'refs/heads/main' && github.event_name == 'push'
```

### Environment URLs

```yaml
environment:
  name: production
  url: ${{ steps.deploy.outputs.url }}
```

### Job Timeout

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - run: npm run build
```
