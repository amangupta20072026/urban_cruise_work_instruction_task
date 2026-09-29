# GitHub Actions CI/CD — Production Guide (Node.js)

A complete, opinionated guide to setting up CI/CD for a Node.js project on GitHub. Every rule here exists because someone got burned by not following it.

---

## Table of contents

1. [Concepts you need first](#1-concepts-you-need-first)
2. [The rules, categorized](#2-the-rules-categorized)
3. [Repo layout](#3-repo-layout)
4. [Workflow 1 — CI (`ci.yml`)](#4-workflow-1--ci-ciyml)
5. [Workflow 2 — Staging deploy (`deploy-staging.yml`)](#5-workflow-2--staging-deploy-deploy-stagingyml)
6. [Workflow 3 — Production deploy (`deploy-production.yml`)](#6-workflow-3--production-deploy-deploy-productionyml)
7. [Supporting configuration](#7-supporting-configuration)
8. [AWS OIDC setup (one-time)](#8-aws-oidc-setup-one-time)
9. [Branch protection and environments (one-time)](#9-branch-protection-and-environments-one-time)
10. [Rollback strategy](#10-rollback-strategy)
11. [Common failure modes and fixes](#11-common-failure-modes-and-fixes)
12. [Checklist](#12-checklist)

---

## 1. Concepts you need first

**GitHub Actions** is a robot inside GitHub. Every time something happens in your repo (a push, a PR, a tag), it can run a script for you. You tell it what to do with YAML files in `.github/workflows/`.

**Workflow** — one YAML file. Contains **jobs**. Each job runs on a fresh virtual machine (a **runner**). Each job contains **steps**, which run in order.

**Runner** — a temporary Ubuntu (or Windows/macOS) VM. Destroyed when the job ends. Anything you want to keep must be uploaded as an artifact or pushed to storage.

**Action** — a reusable step someone else wrote, referenced by `owner/name@ref`. For example, `actions/checkout@<SHA>` clones your repo onto the runner.

**Secret** — an encrypted value stored in GitHub, accessible as `${{ secrets.NAME }}`. Never appears in logs.

**Environment** — a named grouping (e.g. `staging`, `production`) with its own scoped secrets, protection rules, and deploy history.

**OIDC** — a way for GitHub to authenticate to your cloud (AWS/GCP/Azure) without any long-lived secret. GitHub mints a short-lived signed token; the cloud verifies it against a trust policy.

**Docker image** — a self-contained package with your app, dependencies, and a minimal OS. Built once, run anywhere identically. Stored in a **registry** (AWS ECR, GitHub Container Registry, Docker Hub).

**Staging vs Production**
- **Staging** — a live copy of the app for QA/team testing. No real users.
- **Production** — the real thing. Real users. Only tested, approved code goes here.

---

## 2. The rules, categorized

### Repository hygiene
- Protect `main`: require PRs, required status checks, at least one review, no force-push, no deletion.
- Use `CODEOWNERS` to auto-request reviewers for sensitive paths, especially `.github/workflows/` itself.

### Workflow structure
- **Pin every action to a full commit SHA.** Tags are mutable; SHAs are cryptographic. The March 2025 `tj-actions/changed-files` incident leaked secrets from thousands of repos because tags were rewritten.
- Set `timeout-minutes` on every job (roughly 2× normal runtime).
- Set `concurrency` to cancel superseded runs on PR workflows and to queue (not cancel) deploy workflows.

### Permissions
- Set `permissions: contents: read` at the workflow level. Jobs escalate as needed.
- Never use `pull_request_target` unless you fully understand it — it runs untrusted PR code with write access.

### Secrets
- Store in GitHub Secrets or environment-scoped secrets.
- Never `echo` a secret or interpolate it into a shell as a CLI arg where it could hit logs.
- For cloud deploys, use OIDC instead of long-lived access keys.

### Untrusted input
- Never interpolate `${{ github.event.* }}` (PR titles, branch names, tag names, issue bodies) directly into `run:` blocks. Pass through `env:` and reference as `"$VAR"`. Otherwise: shell injection.

### Speed and cost
- Cache dependencies (`setup-node`, `setup-python`, etc. have built-in caching).
- Use matrix builds for parallel test/OS coverage with `fail-fast: false`.
- Split slow jobs so failures surface fast.
- Use `paths:` filters to skip workflows when unrelated files change.

### Deploys
- Gate production through a GitHub Environment with required reviewers.
- Deploy production on tag or `workflow_dispatch`, never on branch push.
- Make deploys idempotent — reruns should be safe.

### Observability
- Upload test reports and build artifacts with `if: always()`.
- Post structured summaries via `$GITHUB_STEP_SUMMARY`.
- Alert on failure only (silence = success).

---

## 3. Repo layout

```
my-api/
├── .github/
│   ├── workflows/
│   │   ├── ci.yml
│   │   ├── deploy-staging.yml
│   │   └── deploy-production.yml
│   ├── dependabot.yml
│   └── CODEOWNERS
├── scripts/
│   ├── deploy.sh
│   └── smoke-test.sh
├── src/
├── .nvmrc                    # e.g. "20.11.0"
├── Dockerfile
├── package.json
├── package-lock.json
└── tsconfig.json
```

---

## 4. Workflow 1 — CI (`ci.yml`)

Runs on every PR and every push to `main`. Fast, parallel, no cloud access.

```yaml
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-24.04
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af # v4.1.0
        with:
          node-version-file: '.nvmrc'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck

  test:
    name: Test (Node ${{ matrix.node }})
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    strategy:
      fail-fast: false
      matrix:
        node: ['20', '22']
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af # v4.1.0
        with:
          node-version: ${{ matrix.node }}
          cache: 'npm'
      - run: npm ci
      - run: npm run test:unit -- --reporter=junit --outputFile=junit.xml
      - name: Upload test report
        if: always()
        uses: actions/upload-artifact@b4b15b8c7c6ac21ea08fcf65892d2ee8f75cf882 # v4.4.3
        with:
          name: junit-node-${{ matrix.node }}
          path: junit.xml

  build:
    name: Build
    runs-on: ubuntu-24.04
    timeout-minutes: 10
    needs: [lint, test]
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af # v4.1.0
        with:
          node-version-file: '.nvmrc'
          cache: 'npm'
      - run: npm ci
      - run: npm run build
      - run: npm audit --audit-level=high
```

### Key points
- `concurrency` with `cancel-in-progress: true` kills stale runs when a new commit lands on the same branch.
- `permissions: contents: read` at the top locks down `GITHUB_TOKEN` to read-only.
- Every action pinned to a 40-character commit SHA. `# v4.2.2` comment is human-readable.
- `node-version-file: '.nvmrc'` keeps CI and local Node versions in lockstep.
- `cache: 'npm'` uses built-in dependency caching. No separate `actions/cache` needed.
- Matrix runs Node 20 and 22 in parallel. `fail-fast: false` = get both results even if one fails.
- `if: always()` on artifact upload = get the test report even when tests fail (when you need it most).
- `needs: [lint, test]` = `build` only runs after both pass.
- `npm audit --audit-level=high` = fail on high/critical CVEs only.

---

## 5. Workflow 2 — Staging deploy (`deploy-staging.yml`)

Automatic on merge to `main`. Builds and pushes a Docker image, deploys to ECS, verifies with smoke test.

```yaml
name: Deploy to Staging

on:
  push:
    branches: [main]

concurrency:
  group: deploy-staging
  cancel-in-progress: false

permissions:
  contents: read
  id-token: write
  packages: write

jobs:
  build-and-push:
    runs-on: ubuntu-24.04
    timeout-minutes: 20
    environment:
      name: staging
      url: https://staging.example.com
    outputs:
      image: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502 # v4.0.2
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-staging
          aws-region: us-east-1

      - name: Login to Amazon ECR
        id: ecr
        uses: aws-actions/amazon-ecr-login@062b18b96a7aff071d4dc91bc00c4c1a7945b076 # v2.0.1

      - name: Extract image metadata
        id: meta
        uses: docker/metadata-action@369eb591f429131d6889c46b94e711f089e6ca96 # v5.6.1
        with:
          images: ${{ steps.ecr.outputs.registry }}/my-api
          tags: |
            type=sha,prefix=,format=long
            type=raw,value=staging-latest

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@c47758b77c9736f4b2ef4073d4d51994fabfe349 # v3.7.1

      - name: Build and push
        uses: docker/build-push-action@4f58ea79222b3b9dc2c8bbdd6debcef730109a75 # v6.9.0
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build-and-push
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    environment:
      name: staging
      url: https://staging.example.com
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502 # v4.0.2
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-staging
          aws-region: us-east-1

      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster staging-cluster \
            --service my-api \
            --force-new-deployment
          aws ecs wait services-stable \
            --cluster staging-cluster \
            --services my-api

      - name: Smoke test
        run: |
          for i in {1..10}; do
            if curl -fsS https://staging.example.com/health; then
              echo "Healthy"
              exit 0
            fi
            sleep 10
          done
          exit 1
```

### Key points
- Triggered only by push to `main`, not by PRs. Merges = deploys.
- `cancel-in-progress: false` — deploys queue, never get killed mid-flight.
- `id-token: write` enables OIDC. No AWS access keys stored anywhere.
- `environment: staging` scopes secrets and records deploy history in the UI.
- Image tagged with commit SHA (traceable) and `staging-latest` (convenient pointer).
- `cache-from: type=gha` reuses Docker layers across builds — sub-minute rebuilds are typical.
- `aws ecs wait services-stable` turns "started deploy" into "deploy completed."
- Smoke test hits `/health` from outside — verifies the app is actually reachable.

---

## 6. Workflow 3 — Production deploy (`deploy-production.yml`)

Triggered by version tag or manual dispatch. Gated by required human approval.

```yaml
name: Deploy to Production

on:
  push:
    tags: ['v*.*.*']
  workflow_dispatch:
    inputs:
      image_tag:
        description: 'Image SHA to deploy'
        required: true

concurrency:
  group: deploy-production
  cancel-in-progress: false

permissions:
  contents: read
  id-token: write

jobs:
  deploy:
    runs-on: ubuntu-24.04
    timeout-minutes: 20
    environment:
      name: production
      url: https://api.example.com
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502 # v4.0.2
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-production
          aws-region: us-east-1

      - name: Resolve image tag
        id: tag
        env:
          INPUT_TAG: ${{ inputs.image_tag }}
          REF_NAME: ${{ github.ref_name }}
        run: |
          if [ -n "$INPUT_TAG" ]; then
            echo "tag=$INPUT_TAG" >> "$GITHUB_OUTPUT"
          else
            echo "tag=$REF_NAME" >> "$GITHUB_OUTPUT"
          fi

      - name: Deploy to ECS
        env:
          IMAGE_TAG: ${{ steps.tag.outputs.tag }}
        run: |
          ./scripts/deploy.sh "$IMAGE_TAG"

      - name: Post-deploy verification
        run: ./scripts/smoke-test.sh https://api.example.com

      - name: Notify Slack on failure
        if: failure()
        uses: slackapi/slack-github-action@37ebaef184d7626c5f204ab8d3baff4262dd30f0 # v1.27.0
        with:
          payload: |
            {"text": "🚨 Production deploy failed: ${{ github.sha }}"}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### Key points
- Triggered by version tags (`v1.4.7`) or manual dispatch — never by branch push.
- `workflow_dispatch` with `image_tag` input = rollback and manual redeploy path.
- `environment: production` is configured in repo settings to require named reviewers. GitHub pauses the workflow until humans approve.
- `role-to-assume` is a *separate* IAM role from staging. Its trust policy only accepts tokens for tag pushes.
- User-controlled inputs (`inputs.image_tag`, `github.ref_name`) passed through `env:` — never interpolated into shell. Prevents shell injection.
- Deploy and smoke-test logic in `scripts/` — testable, reviewable, reusable.
- Slack notification only fires on failure. Silence = success. Success alerts train people to ignore the channel.

---

## 7. Supporting configuration

### `.github/CODEOWNERS`

```
# Everything defaults to the platform team
* @ExampleCorp/platform

# Workflow files require security review
/.github/workflows/ @ExampleCorp/platform @ExampleCorp/security

# Deploy scripts require senior engineer review
/scripts/deploy.sh @ExampleCorp/platform-leads
```

### `.github/dependabot.yml`

Keeps action SHAs and npm packages up to date.

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    groups:
      actions:
        patterns: ["*"]

  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    groups:
      production-dependencies:
        dependency-type: "production"
      development-dependencies:
        dependency-type: "development"
```

### `Dockerfile` (multi-stage, production-ready)

```dockerfile
# Build stage
FROM node:20.11.0-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Runtime stage
FROM node:20.11.0-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production

# Non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

COPY --chown=nodejs:nodejs package*.json ./
RUN npm ci --omit=dev && npm cache clean --force

COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist

USER nodejs
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "dist/index.js"]
```

### `scripts/deploy.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

IMAGE_TAG="${1:?Usage: deploy.sh <image-tag>}"
CLUSTER="production-cluster"
SERVICE="my-api"
TASK_FAMILY="my-api"

echo "Deploying image tag: $IMAGE_TAG"

# Get current task definition, update the image, register new revision
CURRENT_TASK_DEF=$(aws ecs describe-task-definition --task-definition "$TASK_FAMILY")
NEW_TASK_DEF=$(echo "$CURRENT_TASK_DEF" | jq --arg IMAGE "my-api:$IMAGE_TAG" '
  .taskDefinition
  | .containerDefinitions[0].image = $IMAGE
  | {family, taskRoleArn, executionRoleArn, networkMode, containerDefinitions,
     volumes, placementConstraints, requiresCompatibilities, cpu, memory}
')

NEW_REVISION=$(aws ecs register-task-definition \
  --cli-input-json "$NEW_TASK_DEF" \
  --query 'taskDefinition.taskDefinitionArn' \
  --output text)

echo "Registered new task definition: $NEW_REVISION"

aws ecs update-service \
  --cluster "$CLUSTER" \
  --service "$SERVICE" \
  --task-definition "$NEW_REVISION"

echo "Waiting for service to stabilize..."
aws ecs wait services-stable --cluster "$CLUSTER" --services "$SERVICE"

echo "Deploy complete."
```

### `scripts/smoke-test.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

BASE_URL="${1:?Usage: smoke-test.sh <base-url>}"

check() {
  local path="$1"
  local expected_status="${2:-200}"
  echo -n "  $path ... "
  local status
  status=$(curl -s -o /dev/null -w "%{http_code}" "$BASE_URL$path")
  if [ "$status" = "$expected_status" ]; then
    echo "OK ($status)"
  else
    echo "FAIL (got $status, expected $expected_status)"
    exit 1
  fi
}

echo "Smoke testing $BASE_URL"

# Wait for the service to be reachable
for i in {1..20}; do
  if curl -fsS "$BASE_URL/health" > /dev/null; then break; fi
  echo "  waiting for /health... ($i/20)"
  sleep 5
done

check /health 200
check /api/v1/status 200
check /api/v1/does-not-exist 404

echo "All smoke tests passed."
```

---

## 8. AWS OIDC setup (one-time)

Do this once per AWS account. Skip if already set up.

### Step 1 — Create the OIDC provider in AWS

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

### Step 2 — Create the staging IAM role

Trust policy (`staging-trust-policy.json`):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:ExampleCorp/my-api:ref:refs/heads/main"
      }
    }
  }]
}
```

```bash
aws iam create-role \
  --role-name github-actions-staging \
  --assume-role-policy-document file://staging-trust-policy.json
```

### Step 3 — Create the production IAM role

Trust policy (`production-trust-policy.json`) — note the ref pattern requires a version tag:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:ExampleCorp/my-api:ref:refs/tags/v*"
      }
    }
  }]
}
```

```bash
aws iam create-role \
  --role-name github-actions-production \
  --assume-role-policy-document file://production-trust-policy.json
```

### Step 4 — Attach minimum-privilege permissions

Attach only the ECR push and ECS deploy permissions each role needs. Do not grant broad admin access.

---

## 9. Branch protection and environments (one-time)

### Branch protection on `main`

In repo Settings → Branches → Add rule for `main`:
- Require pull request before merging (1 approval minimum).
- Require status checks to pass: `Lint`, `Test (Node 20)`, `Test (Node 22)`, `Build`.
- Require branches to be up to date before merging.
- Require conversation resolution before merging.
- Require signed commits (optional but recommended).
- Do not allow force pushes.
- Do not allow deletions.

### Environment: `staging`

Settings → Environments → New environment `staging`:
- No protection rules needed.
- Add environment secrets for staging (e.g. `STAGING_DATABASE_URL`).

### Environment: `production`

Settings → Environments → New environment `production`:
- **Required reviewers**: add 2 named people or a team.
- **Wait timer**: 5 minutes (gives time to cancel accidental triggers).
- **Deployment branches**: restrict to tags matching `v*.*.*`.
- Add environment secrets for production (e.g. `PROD_DATABASE_URL`, `SLACK_WEBHOOK_URL`).

---

## 10. Rollback strategy

Fastest path: manual dispatch with the previous known-good image SHA.

1. Open Actions → "Deploy to Production" → "Run workflow".
2. Enter the previous commit SHA in the `image_tag` field.
3. Approve the deploy.
4. ECS rolls back to the previous image within minutes.

Because every image is tagged with its commit SHA and stored in ECR, rollback is just re-deploying an older tag. You never rebuild.

For faster rollback, keep a `scripts/rollback.sh` that reads the current ECS task definition, fetches the *previous* one, and re-registers it. Wire it to a `rollback.yml` workflow if it becomes routine.

---

## 11. Common failure modes and fixes

**"Error: Could not assume role with OIDC"**
The role's trust policy doesn't match your workflow's ref. Check the `sub` condition — it must match `repo:OWNER/REPO:ref:refs/heads/BRANCH` or `refs/tags/TAG`.

**"npm ci fails with peer dependency conflict"**
Your `package-lock.json` is out of sync with `package.json`. Run `npm install` locally, commit the updated lockfile.

**"Docker build cache never hits"**
Your `COPY . .` is above `RUN npm ci`. Order matters — copy `package*.json` and install first, then copy source. Cache invalidates only when dependencies change.

**"Deploy job hangs at `services-stable`"**
New task is failing its health check. Check ECS event logs, then CloudWatch logs for the container. Usually a missing env var or a bad startup command.

**"Workflow ran but nothing deployed"**
The trigger didn't fire. Check that the branch or tag matches your `on:` filter. Tags need `git push --tags` (or `git push origin v1.2.3`) — a plain `git push` doesn't push tags.

**"Secrets are empty in the job"**
Environment secrets require the job to declare `environment: NAME`. Repo secrets work everywhere but environment secrets are scoped.

**"Action version was updated and now workflow breaks"**
Dependabot PR raised the SHA to a version with a breaking change. Read the action's release notes, adjust inputs, or pin back to the previous SHA.

---

## 12. Checklist

Setting up a new repo? Work through this in order.

- [ ] `.nvmrc` committed with pinned Node version.
- [ ] `Dockerfile` uses a specific tag (not `latest`), runs as non-root, has a `HEALTHCHECK`.
- [ ] `.github/workflows/ci.yml` created. All actions pinned to SHAs.
- [ ] Branch protection on `main` requires the CI checks.
- [ ] `.github/CODEOWNERS` covers `.github/workflows/` and `scripts/`.
- [ ] `.github/dependabot.yml` created for `npm` and `github-actions`.
- [ ] AWS OIDC provider created (once per account).
- [ ] Staging IAM role with minimum permissions, trust policy scoped to `refs/heads/main`.
- [ ] Production IAM role with minimum permissions, trust policy scoped to `refs/tags/v*`.
- [ ] `staging` environment in repo settings with staging secrets attached.
- [ ] `production` environment with required reviewers, wait timer, tag restriction, prod secrets.
- [ ] `deploy-staging.yml` deploys on push to `main`.
- [ ] `deploy-production.yml` deploys on tag push, gated by environment approval.
- [ ] `scripts/deploy.sh` and `scripts/smoke-test.sh` committed and executable.
- [ ] Slack (or PagerDuty) webhook for failure notifications, stored as secret.
- [ ] Rollback path documented and tested at least once.
- [ ] First end-to-end deploy done. Environments UI shows staging and production entries.

---

## Appendix — The mental model

Every rule here exists because someone got burned:

- **Pinned SHAs** — because tags got hijacked.
- **Least-privilege permissions** — because default tokens got abused.
- **OIDC** — because leaked cloud keys cost companies millions.
- **Required reviewers on prod** — because someone deployed to prod on a Friday at 5pm.
- **Timeouts** — because a runaway workflow burned a month's budget in a night.
- **Env-var pattern for shell injection** — because tag names became attack vectors.
- **Cancel-in-progress on CI, queue on deploys** — because half-deployed prod is worse than not deployed.
- **Silence-on-success alerting** — because success spam trained people to ignore failure alerts.

You don't need all of it on day one. Order to add things to a new repo:

1. CI with pinned SHAs and least-privilege permissions.
2. Branch protection requiring CI.
3. Concurrency and timeouts.
4. Staging deploy with OIDC.
5. Production environment with required reviewers.
6. Failure notifications.
7. Dependabot for npm + github-actions.

That's the whole thing.
