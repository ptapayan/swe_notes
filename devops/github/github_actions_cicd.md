# GitHub Actions CI/CD — Real-World Pipelines

**Related:** [Actions Fundamentals](github_actions_fundamentals.md) | [Actions Workflows](github_actions_workflows.md) | [GitHub Best Practices](github_best_practices.md) | [Dockerfiles](../docker/dockerfiles.md) | [Docker Best Practices](../docker/docker_best_practices.md) | [Ansible Best Practices](../ansible/ansible_best_practices.md)

---

## CI/CD Pipeline Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│  Developer pushes code / opens PR                                  │
└─────────────────────────┬──────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│  CI: Continuous Integration                                      │
│                                                                  │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────────┐  ┌──────────────┐ │
│  │ Lint │  │ Test │  │Build │  │ Security │  │ Artifact     │ │
│  │      │  │      │  │      │  │  Scan    │  │  Upload      │ │
│  └──────┘  └──────┘  └──────┘  └──────────┘  └──────────────┘ │
│      ↓ All pass                                                  │
└─────────────────────────┬────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│  CD: Continuous Delivery / Deployment                            │
│                                                                  │
│  ┌───────────┐     ┌───────────┐     ┌──────────────────────┐  │
│  │  Deploy   │ ──► │  Deploy   │ ──► │  Deploy              │  │
│  │  Staging  │     │  Canary   │     │  Production (manual) │  │
│  └───────────┘     └───────────┘     └──────────────────────┘  │
│       │                  │                      │               │
│   Smoke tests        Verify           Approval gate             │
│                      metrics                                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## Build → Test → Deploy Pattern

### Complete Node.js Pipeline

```yaml
name: CI/CD Pipeline
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

permissions:
  contents: read
  packages: write
  pull-requests: write

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm run lint
      - run: npm run type-check

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports: ['5432:5432']
        options: --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm test -- --coverage
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/testdb
      - name: Upload coverage
        if: github.event_name == 'pull_request'
        uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage/

  build-and-push:
    needs: [lint, test]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      digest: ${{ steps.build.outputs.digest }}

    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=
            type=raw,value=latest

      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Scan image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          severity: HIGH,CRITICAL
          exit-code: 1

  deploy-staging:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to staging
        run: |
          echo "Deploying ${{ needs.build-and-push.outputs.image-tag }} to staging"
          # kubectl set image deployment/api api=$IMAGE --namespace staging
          # or: aws ecs update-service ...

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production         # requires approval
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to production
        run: |
          echo "Deploying to production"
```

---

## Docker Build and Push Workflows

### Multi-Architecture Build

```yaml
name: Docker Multi-Arch
on:
  push:
    tags: ['v*']

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-qemu-action@v3    # for cross-platform builds

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64    # build for both architectures
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:${{ github.ref_name }}
            ghcr.io/${{ github.repository }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## Deployment to Cloud Providers

### AWS ECS Deployment

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    permissions:
      id-token: write              # needed for OIDC
      contents: read

    steps:
      - uses: actions/checkout@v4

      # OIDC authentication (no static credentials!)
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions
          aws-region: us-east-1

      - uses: aws-actions/amazon-ecr-login@v2
        id: ecr

      - name: Build and push to ECR
        run: |
          docker build -t ${{ steps.ecr.outputs.registry }}/my-app:${{ github.sha }} .
          docker push ${{ steps.ecr.outputs.registry }}/my-app:${{ github.sha }}

      - name: Update ECS service
        run: |
          aws ecs update-service \
            --cluster production \
            --service my-api \
            --force-new-deployment \
            --task-definition $(aws ecs register-task-definition \
              --cli-input-json file://task-definition.json \
              --query 'taskDefinition.taskDefinitionArn' \
              --output text)
```

### Kubernetes Deployment

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - uses: azure/setup-kubectl@v3

      - uses: azure/k8s-set-context@v3
        with:
          kubeconfig: ${{ secrets.KUBE_CONFIG }}

      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/api \
            api=ghcr.io/${{ github.repository }}:${{ github.sha }} \
            --namespace production

          kubectl rollout status deployment/api \
            --namespace production \
            --timeout=300s
```

---

## Environment-Based Deployments (Staging → Production)

```yaml
name: Staged Deployment
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image: ${{ steps.build.outputs.image }}
    steps:
      - uses: actions/checkout@v4
      - id: build
        run: |
          IMAGE="ghcr.io/${{ github.repository }}:${{ github.sha }}"
          docker build -t $IMAGE .
          docker push $IMAGE
          echo "image=$IMAGE" >> $GITHUB_OUTPUT

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - run: echo "Deploying ${{ needs.build.outputs.image }} to staging"
      - name: Smoke test
        run: |
          sleep 30   # wait for deployment
          curl -f https://staging.example.com/health || exit 1

  deploy-production:
    needs: [build, deploy-staging]
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://app.example.com
    steps:
      - run: echo "Deploying ${{ needs.build.outputs.image }} to production"
      - name: Health check
        run: |
          sleep 30
          curl -f https://app.example.com/health || exit 1
```

**What's happening:** The `production` environment has required reviewers configured. When `deploy-staging` succeeds, GitHub pauses and waits for a team member to approve before `deploy-production` runs.

---

## Rollback Strategies

### Manual Rollback via workflow_dispatch

```yaml
name: Rollback
on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to rollback to (e.g., abc1234)'
        required: true
        type: string
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options: [staging, production]

jobs:
  rollback:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - name: Rollback deployment
        run: |
          echo "Rolling back ${{ inputs.environment }} to ${{ inputs.version }}"
          kubectl set image deployment/api \
            api=ghcr.io/${{ github.repository }}:${{ inputs.version }} \
            --namespace ${{ inputs.environment }}
          kubectl rollout status deployment/api \
            --namespace ${{ inputs.environment }} \
            --timeout=300s
```

### Automatic Rollback on Health Check Failure

```yaml
- name: Deploy
  id: deploy
  run: kubectl set image deployment/api api=$IMAGE --namespace production

- name: Wait for rollout
  id: rollout
  run: kubectl rollout status deployment/api --timeout=300s --namespace production
  continue-on-error: true

- name: Health check
  id: health
  if: steps.rollout.outcome == 'success'
  run: |
    for i in $(seq 1 10); do
      if curl -sf https://app.example.com/health; then
        echo "Health check passed"
        exit 0
      fi
      sleep 10
    done
    exit 1
  continue-on-error: true

- name: Rollback on failure
  if: steps.rollout.outcome == 'failure' || steps.health.outcome == 'failure'
  run: |
    echo "Deployment failed, rolling back..."
    kubectl rollout undo deployment/api --namespace production
    kubectl rollout status deployment/api --namespace production
```

---

## Release Workflows

### Semantic Versioning with Automated Changelog

```yaml
name: Release
on:
  push:
    tags: ['v*']

permissions:
  contents: write
  packages: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0              # full history for changelog

      - name: Generate changelog
        id: changelog
        run: |
          PREVIOUS_TAG=$(git tag --sort=-version:refname | head -2 | tail -1)
          echo "## Changes since $PREVIOUS_TAG" > CHANGELOG.md
          echo "" >> CHANGELOG.md
          git log --pretty=format:"- %s (%h)" $PREVIOUS_TAG..HEAD >> CHANGELOG.md
          cat CHANGELOG.md

      - name: Build and push Docker image
        run: |
          echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
          docker build -t ghcr.io/${{ github.repository }}:${{ github.ref_name }} .
          docker push ghcr.io/${{ github.repository }}:${{ github.ref_name }}

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          body_path: CHANGELOG.md
          generate_release_notes: true
          draft: false
          prerelease: ${{ contains(github.ref_name, '-rc') }}
```

---

## Monorepo Strategies

### Path Filters (Only Build What Changed)

```yaml
name: Monorepo CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      api: ${{ steps.filter.outputs.api }}
      web: ${{ steps.filter.outputs.web }}
      shared: ${{ steps.filter.outputs.shared }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            api:
              - 'packages/api/**'
              - 'packages/shared/**'
            web:
              - 'packages/web/**'
              - 'packages/shared/**'
            shared:
              - 'packages/shared/**'

  test-api:
    needs: detect-changes
    if: needs.detect-changes.outputs.api == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cd packages/api && npm ci && npm test

  test-web:
    needs: detect-changes
    if: needs.detect-changes.outputs.web == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cd packages/web && npm ci && npm test

  deploy-api:
    needs: [detect-changes, test-api]
    if: |
      needs.detect-changes.outputs.api == 'true' &&
      github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - run: echo "Deploy API"

  deploy-web:
    needs: [detect-changes, test-web]
    if: |
      needs.detect-changes.outputs.web == 'true' &&
      github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - run: echo "Deploy Web"
```

**What's happening:** The `paths-filter` action detects which directories have changes. Only the affected packages are tested and deployed. In a monorepo with 10 services, a change to one service only triggers CI for that service.

---

## GitOps Pattern

```yaml
# Application repo: build and update manifests repo
name: GitOps Deploy
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build and push image
        run: |
          docker build -t ghcr.io/org/api:${{ github.sha }} .
          docker push ghcr.io/org/api:${{ github.sha }}

      - name: Update Kubernetes manifests
        run: |
          git clone https://x-access-token:${{ secrets.MANIFESTS_TOKEN }}@github.com/org/k8s-manifests.git
          cd k8s-manifests
          sed -i "s|ghcr.io/org/api:.*|ghcr.io/org/api:${{ github.sha }}|" \
            production/api/deployment.yaml
          git config user.name "github-actions"
          git config user.email "github-actions@github.com"
          git add .
          git commit -m "Deploy api:${{ github.sha }}"
          git push
          # ArgoCD or Flux detects the change and syncs to cluster
```

---

## Practical Examples / Real-World Patterns

### PR Validation with Status Checks

```yaml
name: PR Validation
on:
  pull_request:
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Check for breaking changes
        run: npm run api:check-compat

      - name: Run full test suite
        run: npm test

      - name: Check bundle size
        run: |
          npm run build
          BUNDLE_SIZE=$(du -sk dist/ | cut -f1)
          if [ $BUNDLE_SIZE -gt 5000 ]; then
            echo "::warning::Bundle size is ${BUNDLE_SIZE}KB (limit: 5000KB)"
          fi

      - name: Comment PR with results
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '## CI Results\nAll checks passed.'
            })
```

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| Pipeline flow | lint → test → build → scan → deploy staging → deploy prod |
| Docker builds | `docker/build-push-action@v5` with `cache-from: type=gha` |
| OIDC auth | No static credentials for cloud. Use `id-token: write` + role assumption |
| Environments | Staging → Production with approval gates. Scoped secrets |
| Rollback | Manual via workflow_dispatch or automatic on health check failure |
| Releases | Tag-triggered. Auto-changelog. GitHub Releases API |
| Monorepo | Path filters to only build changed packages. `dorny/paths-filter` |
| GitOps | Build image → update manifests repo → ArgoCD/Flux syncs |
| Concurrency | `cancel-in-progress: true` for PRs, `false` for deploy queuing |
| Security | Scan images with trivy. OIDC for cloud auth. Pin actions to SHA |
