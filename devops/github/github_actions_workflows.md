# GitHub Actions Workflows — Deep Dive

**Related:** [Actions Fundamentals](github_actions_fundamentals.md) | [Actions CI/CD](github_actions_cicd.md) | [GitHub Best Practices](github_best_practices.md) | [Docker Best Practices](../docker/docker_best_practices.md)

---

## Job Configuration

### runs-on (Runner Selection)

```yaml
jobs:
  build:
    runs-on: ubuntu-latest          # GitHub-hosted Ubuntu
    # runs-on: ubuntu-22.04        # specific version
    # runs-on: macos-14            # macOS (Apple Silicon)
    # runs-on: windows-latest      # Windows Server
    # runs-on: [self-hosted, linux, x64]  # self-hosted with labels
```

### needs (Job Dependencies)

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps: ...

  test:
    runs-on: ubuntu-latest
    steps: ...

  build:
    needs: [lint, test]             # wait for both to complete
    runs-on: ubuntu-latest
    steps: ...

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    steps: ...

  deploy-production:
    needs: deploy-staging           # sequential: lint,test → build → staging → prod
    runs-on: ubuntu-latest
    steps: ...
```

```
lint ────┐
         ├──► build ──► deploy-staging ──► deploy-production
test ────┘
```

### Strategy Matrix

Run a job across multiple combinations of parameters:

```yaml
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest]
        node: [18, 20, 22]
        # Creates 6 jobs: ubuntu/18, ubuntu/20, ubuntu/22, macos/18, macos/20, macos/22

      fail-fast: false              # don't cancel others on first failure
      max-parallel: 4               # limit parallel jobs

    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
```

### Matrix: Include and Exclude

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: [18, 20]

    # Add specific combinations
    include:
      - os: ubuntu-latest
        node: 22
        experimental: true          # add extra variable for this combo

    # Remove specific combinations
    exclude:
      - os: windows-latest
        node: 18                    # don't test Node 18 on Windows
```

### Timeout

```yaml
jobs:
  test:
    timeout-minutes: 30             # kill job after 30 minutes (default: 360)

    steps:
      - name: Long running test
        timeout-minutes: 15         # per-step timeout
        run: npm test
```

---

## Steps Deep Dive

### uses (Run an Action)

```yaml
steps:
  - name: Checkout
    uses: actions/checkout@v4       # action from marketplace
    with:                           # inputs to the action
      fetch-depth: 0                # full history (for changelogs)
      ref: ${{ github.event.pull_request.head.sha }}

  - name: Setup Node
    uses: actions/setup-node@v4
    with:
      node-version: 20
      cache: npm                    # automatic npm cache
```

### run (Execute Shell Commands)

```yaml
steps:
  - name: Single command
    run: npm test

  - name: Multi-line script
    run: |
      echo "Building..."
      npm ci
      npm run build
      echo "Done!"

  - name: Different shell
    run: |
      Write-Host "PowerShell command"
    shell: pwsh

  - name: Python script
    run: |
      import json
      data = {"key": "value"}
      print(json.dumps(data))
    shell: python
```

### Conditional Steps (if)

```yaml
steps:
  - name: Run only on main branch
    if: github.ref == 'refs/heads/main'
    run: echo "On main branch"

  - name: Run only on PRs
    if: github.event_name == 'pull_request'
    run: echo "This is a PR"

  - name: Run only when previous step failed
    if: failure()
    run: echo "Previous step failed!"

  - name: Always run (even on failure/cancel)
    if: always()
    run: echo "Cleanup"

  - name: Run only on success
    if: success()
    run: echo "All good"

  - name: Run only on specific actor
    if: github.actor == 'dependabot[bot]'
    run: echo "Dependabot PR"

  - name: Run on tag push
    if: startsWith(github.ref, 'refs/tags/v')
    run: echo "Tag push: ${{ github.ref_name }}"

  - name: Complex condition
    if: |
      github.event_name == 'push' &&
      github.ref == 'refs/heads/main' &&
      !contains(github.event.head_commit.message, '[skip ci]')
    run: echo "Deploy!"
```

### Environment Variables

```yaml
# Workflow-level
env:
  CI: true
  REGISTRY: ghcr.io

jobs:
  build:
    # Job-level
    env:
      NODE_ENV: production

    steps:
      - name: Step with env
        # Step-level (highest precedence)
        env:
          DATABASE_URL: postgresql://localhost:5432/test
        run: echo "$DATABASE_URL"

      - name: Set env for subsequent steps
        run: echo "VERSION=1.2.3" >> $GITHUB_ENV

      - name: Use env from previous step
        run: echo "Version is $VERSION"       # 1.2.3
```

---

## Secrets and Variables

### Secrets (Encrypted)

```yaml
steps:
  - name: Deploy
    run: ./deploy.sh
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
```

**What's happening:** Secrets are encrypted at rest, decrypted only during workflow execution, and masked in logs. They are set in repository Settings → Secrets and variables → Actions.

**Secret scopes:**
- **Repository secrets:** Available to all workflows in the repo
- **Environment secrets:** Available only to jobs targeting that environment
- **Organization secrets:** Shared across repos (with policy control)

### Variables (Unencrypted)

```yaml
steps:
  - name: Use variable
    run: echo "Deploying to ${{ vars.DEPLOY_TARGET }}"
    # vars.* are unencrypted configuration values
```

### GITHUB_TOKEN (Automatic)

```yaml
permissions:
  contents: read
  pull-requests: write
  packages: write

steps:
  - name: Push to container registry
    run: |
      echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
      docker push ghcr.io/${{ github.repository }}:latest
```

**GITHUB_TOKEN permissions:** Always set minimal permissions at the workflow or job level. The default permissions are too broad.

---

## Artifacts

Artifacts let you share data between jobs and persist build outputs.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build

      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/
          retention-days: 7         # default: 90 days

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: build-output
          path: dist/

      - name: Deploy
        run: ./deploy.sh dist/
```

### Multiple Artifacts

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: test-results
    path: |
      coverage/
      test-results.xml
    if-no-files-found: error       # fail if no files match
```

---

## Caching

Caching dependencies between workflow runs dramatically speeds up builds.

```yaml
steps:
  # Automatic caching (built into setup actions)
  - uses: actions/setup-node@v4
    with:
      node-version: 20
      cache: npm                    # automatically caches ~/.npm

  # Manual caching
  - name: Cache node_modules
    uses: actions/cache@v4
    id: cache-deps
    with:
      path: node_modules
      key: node-modules-${{ hashFiles('package-lock.json') }}
      restore-keys: |
        node-modules-

  - name: Install dependencies
    if: steps.cache-deps.outputs.cache-hit != 'true'
    run: npm ci
```

### Cache Keys Strategy

```yaml
# Exact match: restore only if lock file hasn't changed
key: deps-${{ hashFiles('package-lock.json') }}

# Fallback: try exact match, then most recent partial match
key: deps-${{ hashFiles('package-lock.json') }}
restore-keys: |
  deps-

# Include OS and Node version for platform-specific caches
key: deps-${{ runner.os }}-node-${{ matrix.node }}-${{ hashFiles('package-lock.json') }}
restore-keys: |
  deps-${{ runner.os }}-node-${{ matrix.node }}-
  deps-${{ runner.os }}-
```

**What's happening:** Caches are stored in GitHub's infrastructure. On cache hit, the cached directory is extracted (much faster than installing from network). Cache eviction is LRU, with a total limit of 10GB per repository.

---

## Environment Protection Rules

Environments add approval gates and deployment tracking.

```yaml
jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment: staging            # uses "staging" environment settings
    steps:
      - run: ./deploy.sh staging

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.example.com    # shown in GitHub UI
    steps:
      - run: ./deploy.sh production
```

**Environment settings (configured in repo Settings → Environments):**
- **Required reviewers:** Approval before deployment
- **Wait timer:** Delay before deployment starts
- **Deployment branches:** Restrict which branches can deploy
- **Environment secrets:** Secrets scoped to this environment

---

## Concurrency Control

```yaml
# Cancel superseded runs (same branch, newer push cancels older run)
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

# Prevent concurrent deployments
jobs:
  deploy:
    concurrency:
      group: deploy-production
      cancel-in-progress: false      # queue instead of cancel
```

**What's happening:** When a new run starts with the same concurrency group, the previous pending run is either cancelled (`cancel-in-progress: true`) or queued (`false`). This prevents wasted CI minutes and race conditions in deployments.

---

## Reusable Workflows (workflow_call)

Create reusable workflows that can be called from other workflows (like functions):

```yaml
# .github/workflows/reusable-deploy.yml
name: Reusable Deploy
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      version:
        required: true
        type: string
    secrets:
      deploy-token:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ inputs.version }}
      - run: ./deploy.sh ${{ inputs.environment }}
        env:
          DEPLOY_TOKEN: ${{ secrets.deploy-token }}
```

```yaml
# .github/workflows/release.yml — calls the reusable workflow
name: Release
on:
  push:
    tags: ['v*']

jobs:
  deploy-staging:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: staging
      version: ${{ github.ref_name }}
    secrets:
      deploy-token: ${{ secrets.DEPLOY_TOKEN }}

  deploy-production:
    needs: deploy-staging
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: production
      version: ${{ github.ref_name }}
    secrets:
      deploy-token: ${{ secrets.DEPLOY_TOKEN }}
```

---

## Composite Actions

Bundle multiple steps into a single reusable action:

```yaml
# .github/actions/setup-and-test/action.yml
name: Setup and Test
description: Install deps, build, and test
inputs:
  node-version:
    description: Node.js version
    required: false
    default: '20'
outputs:
  coverage:
    description: Coverage percentage
    value: ${{ steps.test.outputs.coverage }}

runs:
  using: composite
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: npm

    - name: Install dependencies
      shell: bash
      run: npm ci

    - name: Run tests
      id: test
      shell: bash
      run: |
        npm test -- --coverage
        echo "coverage=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')" >> $GITHUB_OUTPUT
```

```yaml
# Usage in a workflow
steps:
  - uses: actions/checkout@v4
  - uses: ./.github/actions/setup-and-test
    id: test
    with:
      node-version: 20
  - run: echo "Coverage was ${{ steps.test.outputs.coverage }}%"
```

---

## Practical Examples / Real-World Patterns

### Output Variables Between Steps and Jobs

```yaml
jobs:
  prepare:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.version }}
      should-deploy: ${{ steps.check.outputs.deploy }}
    steps:
      - id: version
        run: echo "version=$(cat package.json | jq -r '.version')" >> $GITHUB_OUTPUT

      - id: check
        run: |
          if [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
            echo "deploy=true" >> $GITHUB_OUTPUT
          else
            echo "deploy=false" >> $GITHUB_OUTPUT
          fi

  deploy:
    needs: prepare
    if: needs.prepare.outputs.should-deploy == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying version ${{ needs.prepare.outputs.version }}"
```

### Services (Sidecar Containers for Testing)

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
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
      - run: npm ci
      - run: npm test
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379
```

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| Matrix strategy | Test across OS/version combos. `fail-fast: false` to see all results |
| Conditional steps | `if: github.ref == 'refs/heads/main'`, `if: failure()`, `if: always()` |
| Secrets | Encrypted, masked in logs. Repo/env/org scopes. Never echo secrets |
| Artifacts | Share data between jobs. `upload-artifact` + `download-artifact` |
| Caching | `actions/cache@v4`. Key by lockfile hash. 10GB limit per repo |
| Environments | Approval gates, deployment tracking, scoped secrets |
| Concurrency | `cancel-in-progress: true` for CI. `false` for deploy queuing |
| Reusable workflows | `workflow_call` trigger. Like functions for workflows |
| Composite actions | Bundle steps into reusable local actions |
| Outputs | `$GITHUB_OUTPUT` for step outputs. `outputs:` at job level for cross-job |
