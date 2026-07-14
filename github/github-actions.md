# GitHub Actions

**Status:** Active
**Applies to:** All CI/CD workflows in the `avlon-technologies` organization.

---

## Purpose

GitHub Actions is Avlon's only CI/CD system: it verifies every pull request, builds
every artifact, and deploys every environment. Workflows are also the most privileged
code we run — they hold the identity that can modify production Azure resources, and
they execute third-party code by design. This standard makes workflows consistent
(every repo has the same two entry points), safe (no stored cloud credentials, no
mutable third-party code, no over-broad tokens), and boring — a deploy should be the
least interesting event of the day.

Without it: PATs and client secrets accumulate in repo settings and leak or expire at
the worst time; a hijacked action tag exfiltrates the org's secrets; two overlapping
deploys corrupt Terraform state; and every repository invents its own pipeline shape.

## Guiding Principles

1. **Identity, not secrets.** Workflows prove who they are with short-lived OIDC
   tokens, never with stored credentials. A secret that does not exist cannot leak,
   expire, or be exfiltrated.
2. **Least privilege everywhere.** Every workflow declares the minimum `permissions:`
   it needs. The default token grants far more than almost any job requires.
3. **Immutable dependencies.** Third-party code runs only at a reviewed, pinned commit
   SHA. Tags are marketing; SHAs are facts.
4. **Same pipeline, promoted artifact.** One build moves through
   `dev` → `stg` → `prod` — from `main` in GitHub Flow, or from the `release/*` branch it
   was cut on in GitFlow. Environments differ in configuration and approval gates, never
   in build; the artifact validated in `stg` is byte-for-byte the one that runs in
   `prod`.
5. **Shared logic is versioned code.** Pipeline logic used by many repos lives in
   reusable workflows, reviewed and pinned like any other dependency.

## Standard

### Workflow files

Every repository has exactly these entry points (names canonical in
[GitHub Naming](../naming/github.md)):

- **`ci.yml`** (`name: CI`) — build, test, lint on every pull request and every push to
  `main`. Its jobs are the required status checks that gate merge
  ([Repository Structure](repository-structure.md)).
- **`deploy.yml`** (`name: Deploy`) — deploys to the GitHub **environments** `dev`,
  `stg`, `prod` (the environment definitions live in
  [Azure Environments](../azure/environments.md)). Environment protection rules gate
  promotion: **required reviewers on `prod`** (the owning team), optionally on `stg`;
  `dev` deploys unattended. **What triggers each environment depends on the repository's
  branching track** (see [Branching](branching.md)):
  - **GitHub Flow repositories** deploy on push to `main`, promoting one artifact through
    `dev` → `stg` → `prod`. Environments are pipeline stages; the same build moves
    between them.
  - **GitFlow repositories** map environments to branches: a push to `develop` deploys
    `dev`, a push to a `release/*` branch deploys `stg`, and a merge to `main` (a tagged
    release) deploys `prod`. The artifact built from a `release/*` branch is the one
    promoted to `prod` on release — it is not rebuilt.
- **`reusable-<purpose>.yml`** — reusable workflows (`on: workflow_call`) for logic
  shared across repositories, e.g. `reusable-terraform-plan.yml`. These live in a
  platform-owned repo and are called with **pinned refs**, never `@main`.

### Authentication to Azure: OIDC only

Workflows authenticate to Azure exclusively via **Workload Identity Federation**:
`azure/login` exchanges the job's OIDC token for Azure credentials using a federated
credential on a user-assigned managed identity. There are **no client secrets, no
publish profiles, no stored cloud credentials anywhere** — not in repo secrets, not in
environment secrets, not in org secrets.

One deployment identity exists **per workload per environment** — for `billing` in
production, `mi-github-billing-prod-cc` — with a federated credential scoped to the
matching GitHub environment (`repo:avlon-technologies/billing:environment:prod`), so the `prod`
identity is only obtainable by a job that has passed the `prod` environment's approval
gate. Identity setup and RBAC scoping are covered in
[Identity](../azure/identity.md); the federated credential Terraform is in
[Terraform Security](../terraform/security.md).

OIDC requires the token permission in the workflow:

```yaml
permissions:
  id-token: write   # mint the OIDC token for azure/login
  contents: read
```

### Pin actions to commit SHAs

**Third-party actions are pinned to a full commit SHA**, with the version as a trailing
comment:

```yaml
- uses: hashicorp/setup-terraform@b9cd54a3c349d3f38e8881555d616ced269862dd # v3.1.2
```

Tags are mutable: whoever controls (or compromises) the action's repository can move
`v3` to point at malicious code, and every workflow using `@v3` runs it on the next
trigger — with the workflow's token and cloud identity. A SHA is immutable; what you
reviewed is what runs. **Official actions** (`actions/*`, `azure/*`, `github/*`) may use
major version tags — GitHub and Microsoft's supply-chain controls are an accepted risk;
random third parties' are not. Dependabot keeps pinned SHAs current.

### Least-privilege permissions

Every workflow declares an explicit top-level `permissions:` block. The default
(`read-all`, or worse, legacy `write-all`) hands every job broad API access it does not
need; a compromised step inherits whatever the token carries. Grant per-scope, and
elevate per-job only where needed (e.g. `pull-requests: write` only in the job that
comments the plan).

### Secrets

Prefer **no secrets at all**: OIDC for cloud auth, and application secrets fetched at
deploy time from Key Vault (`kv-billing-prod-cc`) by the workload's managed identity.
When a secret is genuinely unavoidable (a third-party SaaS API key), store it in the
**GitHub environment** it belongs to — never as a repo-wide secret — so it is only
exposed to jobs that passed that environment's protection rules.

### Concurrency

Every deploy job runs under a **concurrency group** so two merges cannot deploy the
same environment simultaneously — overlapping Terraform applies against one state file
are how state gets corrupted:

```yaml
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: false   # never cancel a running apply
```

CI workflows use `cancel-in-progress: true` keyed on the PR, so stale runs stop when a
new push arrives.

### Terraform in Actions

- **`plan` on PR**: CI runs `terraform plan` for affected environments and posts the
  plan as a **PR comment**, so reviewers approve the actual infrastructure delta, not
  just the HCL diff.
- **`apply` on merge**: `deploy.yml` applies per environment, in promotion order,
  gated by the environment protection rules. The directory-per-environment layout it
  operates on is defined in [Terraform Environments](../terraform/environments.md).

### Hygiene: caching and timeouts

- **Cache dependencies** (`actions/setup-node` / `setup-python` built-in caching, or
  `actions/cache`) — CI speed is review latency.
- **`timeout-minutes` on every job.** The default is 360 minutes; a hung job should
  fail in minutes, not consume runners for six hours.

## Examples

`ci.yml` for `avlon-technologies/billing`:

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build-test:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run lint
      - run: npm test
```

The prod stage of `deploy.yml`, authenticating with OIDC as
`mi-github-billing-prod-cc` and pulling the image promoted through
`acrplatformsharedcc`:

```yaml
  deploy-prod:
    needs: deploy-stg
    runs-on: ubuntu-latest
    timeout-minutes: 30
    environment: prod            # protection rule: required reviewers
    concurrency:
      group: deploy-billing-prod
      cancel-in-progress: false
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v2
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}        # mi-github-billing-prod-cc
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      - uses: hashicorp/setup-terraform@b9cd54a3c349d3f38e8881555d616ced269862dd # v3.1.2
      - name: Terraform apply (prod)
        working-directory: infra/environments/prod
        run: |
          terraform init
          terraform apply -auto-approve -var "image_tag=${{ github.sha }}"
```

Note that `AZURE_CLIENT_ID` etc. are environment **variables**, not secrets — client
and tenant IDs are identifiers, not credentials; the credential is the OIDC exchange.

Calling a reusable workflow at a pinned ref:

```yaml
jobs:
  plan:
    uses: avlon-technologies/platform-workflows/.github/workflows/reusable-terraform-plan.yml@v2.3.0
    with:
      working-directory: infra/environments/stg
```

## Anti-patterns

| Anti-pattern | Why it fails |
|---------------|--------------|
| `AZURE_CLIENT_SECRET` in repo secrets | Long-lived credential that leaks, expires mid-deploy, and defeats WIF entirely |
| `uses: someone/action@v3` (third party) | Tag can be repointed at malicious code that runs with your token and identity |
| No `permissions:` block | Token defaults are far broader than any job needs; a compromised step inherits it all |
| Deploying from permanent `staging`/`production` branches | Long-lived environment branches drift from `main`; use pipeline promotion (GitHub Flow) or GitFlow's short-lived, back-merged `release/*` branches — see [Branching](branching.md) |
| Rebuilding the image per environment | stg validated a different artifact than prod runs; promote one image by tag through `acrplatformsharedcc` (see [Container Registry](../azure/container-registry.md)) |
| No concurrency group on deploys | Two merges apply Terraform against the same state concurrently |
| Repo-wide secrets used by deploy jobs | Any workflow in the repo can read them, bypassing environment protection |
| Calling reusable workflows `@main` | Every consumer takes unreviewed pipeline changes instantly |
| Missing `timeout-minutes` | Hung jobs burn runners for 6 hours and mask failures |

## Tradeoffs

- **SHA pinning is unreadable and update-heavy.** A hash tells a human nothing, and
  staying current requires Dependabot PRs. Accepted: supply-chain compromise of a
  popular action is a realistic, high-impact attack, and the comment carries the
  version for humans.
- **OIDC setup is more upfront work than pasting a secret** — identities, federated
  credentials, RBAC per workload per environment. Accepted once: the setup is Terraform
  ([Terraform Security](../terraform/security.md)), and the steady state has zero
  credentials to rotate or leak.
- **Required reviewers on `prod` add deploy latency.** Accepted for production;
  `dev` stays unattended so iteration speed is preserved.

## Related Standards

- [Terraform Security](../terraform/security.md) — federated credential and RBAC setup for WIF
- [Identity](../azure/identity.md) — managed identities and RBAC model
- [Terraform Environments](../terraform/environments.md) — the environment directories deploys apply
- [Container Registry](../azure/container-registry.md) — image promotion through the shared registry
- [GitHub Naming](../naming/github.md) — workflow file and environment naming
