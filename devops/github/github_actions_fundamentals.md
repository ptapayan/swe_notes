# GitHub Actions Fundamentals — How CI/CD Actually Works

**Related:** [Actions Workflows](github_actions_workflows.md) | [Actions CI/CD](github_actions_cicd.md) | [GitHub Best Practices](github_best_practices.md) | [Dockerfiles](../docker/dockerfiles.md) | [Docker Best Practices](../docker/docker_best_practices.md)

---

## What Is GitHub Actions?

GitHub Actions is a **CI/CD platform** built directly into GitHub. It automates building, testing, and deploying code in response to events in your repository.

**How to think about it:** GitHub Actions is an event-driven execution engine. Something happens in your repo (push, PR, schedule) → a workflow runs → jobs execute on runners (virtual machines) → steps perform work.

```
┌─────────────────────────────────────────────────────────────┐
│                      GitHub Event                            │
│  (push, pull_request, schedule, workflow_dispatch, etc.)    │
└──────────────────────────┬──────────────────────────────────┘
                           │ triggers
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                      Workflow                                │
│  .github/workflows/ci.yml                                   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Job: build                                          │    │
│  │  runs-on: ubuntu-latest                              │    │
│  │                                                     │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐         │    │
│  │  │ Step 1   │→ │ Step 2   │→ │ Step 3   │         │    │
│  │  │ checkout │  │ npm ci   │  │ npm test │         │    │
│  │  └──────────┘  └──────────┘  └──────────┘         │    │
│  └─────────────────────────────────────────────────────┘    │
│                           │                                  │
│                           │ needs: build                     │
│                           ▼                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Job: deploy                                         │    │
│  │  runs-on: ubuntu-latest                              │    │
│  │  environment: production                             │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

---

## Core Concepts

### Events

Events are things that happen in your GitHub repository that trigger workflows.

```yaml
on:
  # Git events
  push:
    branches: [main, develop]
    paths: ['src/**', '*.json']
    tags: ['v*']

  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]

  # Scheduled (cron syntax)
  schedule:
    - cron: '0 6 * * 1-5'     # weekdays at 6 AM UTC

  # Manual trigger (via UI or API)
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options: [staging, production]
      version:
        description: 'Version to deploy'
        required: false
        type: string

  # API trigger
  repository_dispatch:
    types: [deploy-production]

  # Other events
  release:
    types: [published]
  issues:
    types: [opened, labeled]
  workflow_run:
    workflows: ["Build"]
    types: [completed]
```

### Workflows

A workflow is a YAML file in `.github/workflows/` that defines an automated process. A repository can have multiple workflows.

```
.github/
└── workflows/
    ├── ci.yml              # runs on every push/PR
    ├── deploy.yml          # runs on push to main
    ├── release.yml         # runs on tag creation
    ├── nightly.yml         # runs on schedule
    └── cleanup.yml         # runs on PR close
```

### Jobs

Jobs are independent units of work within a workflow. By default, jobs run **in parallel**. Use `needs` to create dependencies.

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps: ...

  test:
    runs-on: ubuntu-latest
    steps: ...

  # lint and test run in parallel (no dependencies)

  deploy:
    needs: [lint, test]       # runs AFTER both lint and test succeed
    runs-on: ubuntu-latest
    steps: ...
```

```
lint ──────┐
           ├──► deploy
test ──────┘
```

### Steps

Steps are individual tasks within a job. They run sequentially and share the same runner filesystem.

```yaml
steps:
  # Step using a published action
  - name: Checkout code
    uses: actions/checkout@v4       # action reference

  # Step running a shell command
  - name: Install dependencies
    run: npm ci                     # shell command

  # Step with inputs and environment
  - name: Run tests
    run: npm test
    env:
      CI: true
      DATABASE_URL: ${{ secrets.TEST_DB_URL }}
```

### Actions

Actions are reusable units of code that can be used as steps. Three types:

```yaml
# 1. JavaScript action (runs in Node.js)
- uses: actions/checkout@v4

# 2. Docker action (runs in a container)
- uses: docker://alpine:3.19
  with:
    args: echo "Hello from Docker"

# 3. Composite action (YAML file with multiple steps)
- uses: ./.github/actions/my-action   # local composite action
```

**Action references:**
```yaml
uses: actions/checkout@v4              # official action, version tag
uses: actions/checkout@abc1234         # pinned to exact SHA (security best practice)
uses: owner/repo@main                  # branch reference (not recommended)
uses: owner/repo/path@v1              # subdirectory of a repo
uses: ./.github/actions/local-action  # local action in same repo
uses: docker://alpine:3.19            # Docker Hub image
```

### Runners

Runners are the machines where jobs execute. There are two types:

**GitHub-Hosted Runners:**

| Runner | Label | OS | CPU/RAM |
|---|---|---|---|
| Ubuntu | `ubuntu-latest`, `ubuntu-22.04` | Ubuntu 22.04/24.04 | 4 vCPU, 16GB RAM |
| macOS | `macos-latest`, `macos-14` | macOS 13/14 | 3-4 vCPU, 14GB RAM |
| Windows | `windows-latest`, `windows-2022` | Windows Server 2022 | 4 vCPU, 16GB RAM |

```yaml
jobs:
  build:
    runs-on: ubuntu-latest          # GitHub-hosted
```

**What's happening with GitHub-hosted runners:** Each job gets a fresh virtual machine. The VM is provisioned with common tools (git, Docker, Node.js, Python, Go, Java, etc.), runs your job, and is destroyed. Nothing persists between jobs or workflow runs.

**Self-Hosted Runners:**

```yaml
jobs:
  build:
    runs-on: [self-hosted, linux, x64, gpu]   # custom labels
```

Self-hosted runners are machines you manage. Use them for:
- Custom hardware (GPUs, ARM, specialized chips)
- Private network access (internal APIs, databases)
- Cost savings at scale
- Persistent caches and tools

---

## Workflow YAML Structure

```yaml
# .github/workflows/ci.yml
name: CI Pipeline                    # display name in GitHub UI

on:                                  # triggers
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:                         # GITHUB_TOKEN permissions
  contents: read
  pull-requests: write

env:                                 # workflow-level environment variables
  NODE_VERSION: 20
  REGISTRY: ghcr.io

jobs:
  test:
    name: Run Tests                  # display name in GitHub UI
    runs-on: ubuntu-latest
    timeout-minutes: 15              # kill job after 15 minutes

    services:                        # sidecar containers (databases, etc.)
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/postgres
```

---

## Marketplace Actions

The GitHub Marketplace has thousands of pre-built actions. The most commonly used:

```yaml
# Checkout repository code
- uses: actions/checkout@v4

# Set up language runtimes
- uses: actions/setup-node@v4
- uses: actions/setup-python@v5
- uses: actions/setup-go@v5
- uses: actions/setup-java@v4

# Caching
- uses: actions/cache@v4

# Artifacts
- uses: actions/upload-artifact@v4
- uses: actions/download-artifact@v4

# Docker
- uses: docker/setup-buildx-action@v3
- uses: docker/login-action@v3
- uses: docker/build-push-action@v5

# Cloud deployments
- uses: aws-actions/configure-aws-credentials@v4
- uses: google-github-actions/auth@v2
- uses: azure/login@v2

# Security
- uses: github/codeql-action/analyze@v3
- uses: aquasecurity/trivy-action@master
```

---

## Practical Examples / Real-World Patterns

### Minimal CI Workflow

```yaml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm test
      - run: npm run lint
```

### Matrix Testing (Multiple Versions/OS)

```yaml
name: Matrix CI
on: [push]

jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest]
        node-version: [18, 20, 22]
      fail-fast: false              # don't cancel others if one fails

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm ci
      - run: npm test
```

### Manual Deploy with Inputs

```yaml
name: Deploy
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deploy target'
        required: true
        type: choice
        options: [staging, production]
      version:
        description: 'Version tag'
        required: true
        type: string

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ inputs.version }}
      - run: echo "Deploying ${{ inputs.version }} to ${{ inputs.environment }}"
```

---

## How Billing Works

| Runner Type | Per-Minute Cost | Storage | Included (Free tier) |
|---|---|---|---|
| Ubuntu | $0.008/min | $0.25/GB/month | 2,000 min/month |
| macOS | $0.08/min | — | 200 min/month |
| Windows | $0.016/min | — | 2,000 min/month |
| Self-hosted | Free | — | Unlimited |

**Cost optimization tips:**
- Cancel redundant runs (push + PR for same commit)
- Use caching aggressively (dependency installs)
- Use `timeout-minutes` to prevent runaway jobs
- Use `concurrency` to cancel superseded runs
- Use self-hosted runners for heavy workloads

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| Workflow | YAML file in `.github/workflows/`. Triggered by events |
| Events | push, pull_request, schedule, workflow_dispatch, release, etc. |
| Jobs | Independent units. Parallel by default. `needs` for dependencies |
| Steps | Sequential within a job. `uses` for actions, `run` for commands |
| Actions | Reusable code. Pin to SHA for security. Marketplace has thousands |
| Runners | GitHub-hosted (fresh VM each time) or self-hosted (you manage) |
| Services | Sidecar containers (postgres, redis) for integration tests |
| Permissions | Always set minimal `permissions` for GITHUB_TOKEN |
| Billing | Ubuntu cheapest. macOS 10x. Self-hosted is free. Cache to save minutes |
