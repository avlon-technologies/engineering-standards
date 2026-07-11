# GitHub Naming

**Status:** Active
**Applies to:** Everything named inside the `avlon-technologies` GitHub organization — repositories, branches, workflow files, GitHub environments, labels, and teams.

---

## Purpose

GitHub names have longer lives than the code they contain. A repository name is baked into every
clone URL, module source, CODEOWNERS entry, and bookmark; renaming one breaks Terraform module
pins and muscle memory simultaneously. Environment names are even more critical: GitHub
environment protection rules are the gate between a merged PR and production, and they match
Azure environments **by string comparison**. This standard exists so that names in GitHub are
predictable from the workload they serve, and so the strings that automation depends on never
drift.

## Guiding Principles

1. **The workload name is the repository name.** The same ≤ 12-character workload name defined
   in [General Naming Principles](general.md) flows unchanged from Azure resource to repository
   to state key. One name, everywhere.
2. **Names encode function, not history.** No codenames, no `-v2`, no `new-` — a repository
   describes what it is today, not the project that spawned it or the rewrite it replaced.
3. **Environment strings are a contract.** `dev`, `stg`, `prod` in GitHub must be byte-identical
   to the Azure environment codes, because deployment automation keys off them.
4. **Grouping is done with prefixes.** Labels, reusable workflows, and Terraform module repos
   all use a fixed prefix so related items sort and filter together.

## Standard

### Repositories

All repository names are lowercase kebab-case.

| Kind | Pattern | Examples |
|------|---------|----------|
| Product / workload | `<product>` or `<product>-<component>` | `billing`, `billing-api`, `billing-web`, `code-review` |
| Terraform module | `terraform-azurerm-<purpose>` | `terraform-azurerm-container-app`, `terraform-azurerm-key-vault` |
| Org-level | descriptive kebab-case | `engineering-standards` |

Single-repo workloads take the bare workload name (`billing`). Split a component out into its
own repository only when it has a genuinely independent release lifecycle — and then name it
`<product>-<component>` so the family sorts together. Terraform module repos follow the
`terraform-azurerm-<purpose>` convention required for registry compatibility and instantly
recognizable as reusable infrastructure — see [Terraform Naming](terraform.md) and
[Terraform Modules](../terraform/modules.md).

**Forbidden patterns:**

- **Codenames** (`falcon`, `project-titan`) — opaque to everyone hired after the kickoff.
- **Version suffixes** (`billing-v2`) — versions live in git tags and releases. If a rewrite
  must coexist with its predecessor, the rewrite is temporary or it is a different product with
  a real name.
- **`new-` prefixes** (`new-billing`) — nothing stays new; the name is wrong within a year.
- **Personal names** (`mikes-scripts`) — repositories outlive their creators; ownership belongs
  in CODEOWNERS and teams.

### Branches

Trunk-based development off `main` (see [Branching](../github/branching.md)). Short-lived
branches follow:

```text
<type>/<issue>-<slug>
```

where `<type>` is one of `feature`, `fix`, `chore`, `docs`, `hotfix`; `<issue>` is the issue or
ticket number; and `<slug>` is a short kebab-case description.

```text
feature/142-invoice-export
fix/198-null-invoice-total
chore/205-bump-azurerm-provider
docs/211-naming-standard-links
hotfix/220-payment-retry-loop
```

The type prefix makes branch listings scannable and lets automation apply labels; the issue
number links the branch to its context; the slug is for humans. Branches are deleted on merge —
never park long-lived work on a branch name.

### Workflow files

Workflow files live in `.github/workflows/` and are named for what they do, not how:

| File | Workflow `name:` | Purpose |
|------|------------------|---------|
| `ci.yml` | `CI` | Build, test, lint on every PR |
| `deploy.yml` | `Deploy` | Deployment to environments on merge to `main` |
| `reusable-<purpose>.yml` | descriptive | Called workflows, e.g. `reusable-terraform-plan.yml`, `reusable-container-build.yml` |

Every repository uses the same two entry-point names, so an engineer landing in any repo knows
exactly where the pipeline lives. Reusable workflows carry the `reusable-` prefix so callers and
callees are distinguishable at a glance in the workflows directory. See
[GitHub Actions](../github/github-actions.md).

### GitHub environments

GitHub environments are named `dev`, `stg`, `prod` — **exactly** the Azure environment codes,
no others and no variants. This is not stylistic. Deployment automation keys off these strings
at every layer:

- The workflow job's `environment:` selects the protection rules (required reviewers on `prod`).
- The same string is passed to Terraform as `var.environment` and validated against
  `["dev", "stg", "prod"]`.
- The same string appears in the Azure resource names being deployed and selects the state
  container (`tfstate-prod`).
- The OIDC federated credential for `mi-github-billing-prod-cc` is scoped to the repository
  *and environment* — a job in environment `production` would simply fail to authenticate.

A GitHub environment named `staging` next to a Terraform variable expecting `stg` is a broken
deployment discovered at the worst time. One string, end to end. The full environment
definitions live in [Azure Environments](../azure/environments.md).

### Labels

Labels use prefixed groups, one color per prefix group, so they sort and filter as families (see
[Labels](../github/labels.md) for the full palette and process):

```text
type/bug        type/feature      type/chore       type/docs
priority/p1     priority/p2       priority/p3
status/blocked  status/needs-review
area/<component>                  # e.g. area/billing-api, area/infra
```

`area/` labels take component names from the same vocabulary as everything else — `area/billing-api`,
not `area/backend-stuff`.

### Teams

Teams are named `<team>-team` and referenced as `@avlon-technologies/<team>-team`:

```text
@avlon-technologies/platform-team
@avlon-technologies/billing-team
```

The `-team` suffix keeps team mentions unambiguous next to repository and workload names in
CODEOWNERS files and PR reviews. CODEOWNERS entries always reference teams, never individuals —
see [Code Reviews](../github/code-reviews.md).

## Examples

The `billing` workload, end to end:

```text
Repository:          avlon-technologies/billing            (infra + app, single lifecycle)
Module consumed:     avlon-technologies/terraform-azurerm-container-app?ref=v1.2.0
Branch:              feature/142-invoice-export
Workflows:           .github/workflows/ci.yml         (name: CI)
                     .github/workflows/deploy.yml     (name: Deploy)
GitHub environments: dev → stg → prod   (prod: required reviewers @avlon-technologies/billing-team)
Labels:              type/feature, priority/p2, area/billing-api
CODEOWNERS:          * @avlon-technologies/billing-team
                     /infra/ @avlon-technologies/platform-team
```

And the string-equality chain that makes `deploy.yml` work:

```yaml
jobs:
  deploy-prod:
    environment: prod                    # GitHub environment — gates on reviewers
    steps:
      - run: terraform apply -var environment=prod   # matches Terraform validation
        working-directory: infra/environments/prod   # matches directory layout
        # → deploys app-billing-api-prod-cc, state key tfstate-prod/billing.tfstate
```

Four systems, one string.

## Anti-patterns

| Anti-pattern | Why it fails |
|--------------|--------------|
| `BillingAPI`, `billing_api` | Repository names are lowercase kebab-case, always. |
| `project-falcon`, `new-billing`, `billing-v2` | Codenames, `new-`, and version suffixes rot immediately. |
| `mikes-terraform-stuff` | Personal names; unownable the day Mike moves on. |
| GitHub environment `staging` or `production` | Breaks string equality with Azure/Terraform `stg`/`prod`; OIDC scoping and validation fail. |
| `build-and-deploy-final2.yml` | Unpredictable workflow names; use `ci.yml` / `deploy.yml`. |
| Branch `mike/wip` | No type, no issue link, invites long-lived personal branches. |
| Labels `bug`, `Bug`, `bugs` coexisting | Unprefixed labels drift into duplicates; use `type/bug`. |

## Tradeoffs

- **Strict repo naming can feel heavyweight for experiments.** A quick spike still needs a real
  kebab-case name. Acceptable: renaming a repo later breaks clone URLs and module pins, so we
  pay the naming cost when it is cheapest — at creation.
- **`<type>/<issue>-<slug>` requires an issue to exist first.** That is partly the point —
  work is tracked — but it adds friction for trivial changes. `chore/` with a housekeeping
  issue number keeps the format uniform.
- **Exactly-matching environment strings couple GitHub to Azure.** Renaming an environment tier
  would touch every repo. We accept this: the coupling is real whether or not the names admit
  it, and matching strings make it visible instead of latent.

## Related Standards

- [General Naming Principles](general.md) — workload names and environment codes
- [Repository Structure](../github/repository-structure.md) — what goes inside the repo
- [Branching](../github/branching.md) — trunk-based flow the branch names serve
- [GitHub Actions](../github/github-actions.md) — workflows, OIDC, environment protection
- [Labels](../github/labels.md) — full label set and colors
