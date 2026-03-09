# GitHub Best Practices (FAANG-Level Engineering)

**Related:** [Actions Fundamentals](github_actions_fundamentals.md) | [Actions Workflows](github_actions_workflows.md) | [Actions CI/CD](github_actions_cicd.md) | [Docker Best Practices](../docker/docker_best_practices.md) | [Ansible Best Practices](../ansible/ansible_best_practices.md)

---

## 1. Branch Protection Rules

Branch protection prevents direct pushes to critical branches and enforces quality gates.

```
Settings → Branches → Branch protection rules → Add rule
Branch name pattern: main
```

### Essential Protection Settings

| Setting | FAANG Default | Why |
|---|---|---|
| Require pull request reviews | 1-2 reviewers | No code ships without review |
| Dismiss stale reviews | Enabled | New pushes invalidate old approvals |
| Require review from CODEOWNERS | Enabled | Domain experts must approve |
| Require status checks to pass | Enabled | CI must pass before merge |
| Require conversation resolution | Enabled | All review comments must be addressed |
| Require linear history | Enabled | No merge commits. Rebase only |
| Require signed commits | Optional | GPG-verified authorship |
| Restrict who can push | Team leads only | Emergency escape hatch |
| Allow force pushes | NEVER | Prevents history rewriting |

**What's happening:** With these settings, code reaches `main` only after:
1. A PR is opened
2. CI passes (lint, test, build, scan)
3. Required reviewers approve
4. CODEOWNERS approve for their areas
5. All review comments are resolved
6. The PR is rebased on latest `main`

---

## 2. Pull Request Workflows

### PR Template

```markdown
<!-- .github/pull_request_template.md -->
## Summary
<!-- What does this PR do? Why? -->

## Changes
<!-- Bulleted list of what changed -->

## Testing
<!-- How was this tested? -->
- [ ] Unit tests added/updated
- [ ] Integration tests pass
- [ ] Manually tested locally

## Checklist
- [ ] Code follows project conventions
- [ ] Documentation updated (if applicable)
- [ ] No secrets or credentials committed
- [ ] Breaking changes documented
```

### Required Status Checks

```yaml
# .github/workflows/ci.yml — these become required status checks
name: CI
on: [pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run lint

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test

  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run build
```

Configure in Settings → Branches → Require status checks:
- `lint` (required)
- `test` (required)
- `build` (required)

**Critical:** If the workflow name changes or a job is renamed, the status check reference breaks. Pin the job names.

---

## 3. Git Flow vs Trunk-Based Development

### Trunk-Based Development (FAANG Preferred)

```
main ──────●────●────●────●────●────●────●──►
            \  /      \  /      \  /
             \/        \/        \/
         feature-1  feature-2  feature-3
         (short-lived branches, 1-3 days max)
```

```
Principles:
  • main is always deployable
  • Feature branches are short-lived (hours to days, NOT weeks)
  • Small PRs (< 400 lines of diff)
  • Feature flags for incomplete features
  • Deploy from main continuously
```

### Git Flow (Legacy / Complex Release Cycles)

```
main ─────────────────●─────────────────────●──►
                      ↑                      ↑
release/1.0 ────●────●        release/1.1 ──●──►
                ↑
develop ──●──●──●──●──●──●──●──●──●──●──●──●──►
           \/ \/ \/               \/ \/
         feat feat feat         feat feat
```

**When to use which:**
- **Trunk-based:** Fast-moving teams, continuous deployment, microservices
- **Git flow:** Packaged software with formal releases, regulatory requirements, multiple supported versions

---

## 4. Commit Conventions

### Conventional Commits

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

```bash
# Types
feat:     # new feature
fix:      # bug fix
docs:     # documentation only
style:    # formatting, semicolons, etc.
refactor: # code change that neither fixes a bug nor adds a feature
test:     # adding or correcting tests
chore:    # build process, dependencies, CI
perf:     # performance improvement
ci:       # CI/CD changes

# Examples
feat(auth): add OAuth2 login with Google
fix(api): handle null response from payment gateway
docs(readme): add deployment instructions
refactor(db): extract query builder into separate module
test(auth): add integration tests for token refresh
chore(deps): upgrade express to 4.19
```

**Why conventional commits matter:**
- Automated changelog generation
- Automated semantic versioning (`feat` = minor, `fix` = patch, `BREAKING CHANGE` = major)
- Easy to read git history
- Machine-parseable for release tooling

### Enforcing with CI

```yaml
name: Lint Commits
on: [pull_request]

jobs:
  commitlint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: wagoid/commitlint-github-action@v5
```

---

## 5. CODEOWNERS

The `CODEOWNERS` file defines who must review changes to specific paths:

```bash
# .github/CODEOWNERS

# Default: require review from the team
*                           @org/engineering

# Frontend team owns frontend code
/packages/web/              @org/frontend-team
/packages/ui-components/    @org/frontend-team

# Backend team owns API code
/packages/api/              @org/backend-team
/packages/api/auth/         @org/security-team  # security team for auth

# Infrastructure team owns deployment configs
/.github/workflows/         @org/platform-team
/terraform/                 @org/platform-team
/ansible/                   @org/platform-team
/docker/                    @org/platform-team

# Database migrations need DBA review
/migrations/                @org/dba-team

# Docs can be reviewed by anyone
/docs/                      @org/engineering
```

**What's happening:** When a PR changes files in `/packages/api/auth/`, the `@org/security-team` is automatically requested for review. The PR cannot be merged until they approve (if "Require review from CODEOWNERS" is enabled).

---

## 6. Security

### Dependabot

```yaml
# .github/dependabot.yml
version: 2
updates:
  # npm dependencies
  - package-ecosystem: npm
    directory: "/"
    schedule:
      interval: weekly
      day: monday
    open-pull-requests-limit: 10
    labels:
      - dependencies
    reviewers:
      - org/platform-team
    groups:
      minor-and-patch:
        update-types:
          - minor
          - patch

  # GitHub Actions
  - package-ecosystem: github-actions
    directory: "/"
    schedule:
      interval: weekly
    labels:
      - ci

  # Docker base images
  - package-ecosystem: docker
    directory: "/"
    schedule:
      interval: weekly
```

### Secret Scanning

GitHub automatically scans for leaked secrets (API keys, tokens, passwords) in pushes and alerts you. Enable **push protection** to block pushes containing detected secrets.

```
Settings → Code security and analysis → Secret scanning → Enable
Settings → Code security and analysis → Push protection → Enable
```

### Code Scanning with CodeQL

```yaml
# .github/workflows/codeql.yml
name: CodeQL Analysis
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'    # weekly scan

jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
    strategy:
      matrix:
        language: [javascript, python]   # add your languages

    steps:
      - uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: ${{ matrix.language }}
          queries: +security-extended     # more thorough analysis

      - name: Autobuild
        uses: github/codeql-action/autobuild@v3

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
```

**What CodeQL finds:** SQL injection, XSS, path traversal, insecure deserialization, hardcoded credentials, and hundreds of other vulnerability patterns.

### Actions Security

```yaml
# SECURITY BEST PRACTICE: Pin actions to SHA, not version tag
# Version tags can be moved to point to different code (supply chain attack)

# BAD: tag can be moved
- uses: actions/checkout@v4

# GOOD: pinned to exact commit SHA
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11  # v4.1.1

# GOOD: use Dependabot to keep SHAs updated (see dependabot.yml above)
```

### OIDC for Cloud Authentication

```yaml
# Instead of storing cloud credentials as secrets, use OIDC
# GitHub Issues a JWT → Cloud provider validates it → No static credentials

permissions:
  id-token: write     # needed for OIDC token request

steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789:role/github-actions
      aws-region: us-east-1
      # No AWS_ACCESS_KEY_ID or AWS_SECRET_ACCESS_KEY needed!
```

**What's happening:** GitHub generates a short-lived JWT token containing the repo, branch, and workflow info. AWS validates this token against a pre-configured trust policy and issues temporary credentials. No long-lived secrets to rotate.

### Least-Privilege GITHUB_TOKEN

```yaml
# ALWAYS set minimal permissions
# Default permissions are too broad

permissions:
  contents: read          # read repo code
  pull-requests: write    # comment on PRs
  packages: write         # push to container registry
  # Everything else defaults to "none"
```

---

## 7. Repository Templates

Create template repositories for consistent project setup:

```
template-repo/
├── .github/
│   ├── workflows/
│   │   ├── ci.yml
│   │   ├── deploy.yml
│   │   └── codeql.yml
│   ├── pull_request_template.md
│   ├── CODEOWNERS
│   └── dependabot.yml
├── .gitignore
├── .editorconfig
├── Dockerfile
├── docker-compose.yml
├── Makefile
└── src/
```

Settings → General → Template repository (checkbox)

Teams can then: "Use this template" → creates a new repo with all the CI/CD, protection rules, and project structure pre-configured.

---

## 8. GitHub CLI (gh)

```bash
# Authentication
gh auth login
gh auth status

# Pull Requests
gh pr create --title "feat: add auth" --body "Description"
gh pr list
gh pr view 123
gh pr checkout 123         # fetch and checkout PR branch
gh pr review 123 --approve
gh pr merge 123 --squash

# Issues
gh issue create --title "Bug: login fails" --label bug
gh issue list --label "high-priority"
gh issue close 123

# Workflows
gh run list                # list recent workflow runs
gh run view 12345          # view a specific run
gh run watch 12345         # live stream run output
gh run rerun 12345         # re-run a failed workflow

# Releases
gh release create v1.0.0 --generate-notes
gh release list

# Repository
gh repo create my-app --public --clone
gh repo clone org/repo
gh repo fork org/repo

# API access (for anything not covered by built-in commands)
gh api repos/org/repo/pulls/123/reviews
gh api -X POST repos/org/repo/dispatches -f event_type=deploy
```

---

## 9. Issue and Project Management

### Issue Templates

```yaml
# .github/ISSUE_TEMPLATE/bug_report.yml
name: Bug Report
description: Report a bug
labels: [bug, triage]
body:
  - type: input
    id: version
    attributes:
      label: Version
      placeholder: e.g., 2.1.0
    validations:
      required: true

  - type: textarea
    id: description
    attributes:
      label: Describe the bug
      description: Clear and concise description
    validations:
      required: true

  - type: textarea
    id: reproduction
    attributes:
      label: Steps to reproduce
      value: |
        1. Go to '...'
        2. Click on '...'
        3. See error

  - type: dropdown
    id: severity
    attributes:
      label: Severity
      options:
        - Critical (service down)
        - High (major feature broken)
        - Medium (workaround exists)
        - Low (cosmetic)
```

---

## 10. Production Checklist for GitHub Repositories

```
Branch Protection:
  [ ] main branch protected
  [ ] Require PR reviews (1-2 reviewers)
  [ ] Require status checks to pass
  [ ] Require CODEOWNERS review
  [ ] Dismiss stale reviews
  [ ] No force pushes allowed
  [ ] Require linear history (rebase only)

CI/CD:
  [ ] CI runs on every PR (lint, test, build)
  [ ] CD deploys on merge to main
  [ ] Environment protection rules for production
  [ ] Concurrency control (cancel-in-progress for PRs)
  [ ] Timeout limits on all jobs
  [ ] Minimal GITHUB_TOKEN permissions

Security:
  [ ] Dependabot enabled for all ecosystems
  [ ] Secret scanning with push protection
  [ ] CodeQL analysis on PRs and weekly
  [ ] Actions pinned to SHA (not version tags)
  [ ] OIDC for cloud authentication (no static secrets)
  [ ] Least-privilege GITHUB_TOKEN permissions

Developer Experience:
  [ ] PR template defined
  [ ] Issue templates defined
  [ ] CODEOWNERS file configured
  [ ] Conventional commit linting
  [ ] README with setup instructions
  [ ] Contributing guide
```

---

## Key Takeaways

| Practice | What to Remember |
|---|---|
| Branch protection | Require reviews, status checks, CODEOWNERS approval, linear history |
| Trunk-based development | Short-lived branches, small PRs, feature flags, deploy from main |
| Conventional commits | `feat:`, `fix:`, `docs:`. Enables automated changelog + versioning |
| CODEOWNERS | Per-path review requirements. Security team reviews auth code |
| Secret scanning | Enable push protection. No secrets in code. Use OIDC for cloud |
| CodeQL | Automated vulnerability detection. Run on PRs and weekly schedule |
| Actions security | Pin to SHA. Minimal permissions. OIDC over static credentials |
| Dependabot | Auto-update deps, actions, Docker images. Group minor/patch updates |
| GitHub CLI | `gh pr`, `gh run`, `gh release`. Scriptable for automation |
| Templates | Template repos for consistent project setup across the org |
