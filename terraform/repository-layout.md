# Terraform Repository Layout

**Status:** Active
**Applies to:** Every repository at Avlon that contains Terraform configuration.

---

## Purpose

Terraform tolerates almost any directory structure, which is exactly the problem: without a standard, every repository invents its own, and engineers pay a re-orientation tax on every repo they touch. Reviewers can't tell at a glance what a change affects, pipelines need per-repo configuration, and "where do I put the staging tfvars?" becomes a Slack question instead of muscle memory. A single, predictable layout means anyone can open any Avlon repository and know immediately where the modules are, where each environment's root lives, and what a change to a given file will affect.

## Guiding Principles

1. **One layout, everywhere.** The structure below is not a suggestion or a starting point — it is the layout. Predictability across dozens of repositories is worth more than any local optimization in one of them.
2. **Infra lives next to the code it serves.** A workload's infrastructure changes in lockstep with its application; keeping them in one repository keeps them in one pull request, one review, and one deployment history.
3. **Environment roots are thin.** Roots compose modules and supply values. If a root contains interesting logic, that logic belongs in a module — see [Modules](modules.md).
4. **The directory tree is the blast-radius map.** `infra/environments/prod/` affects production and nothing else. A reviewer should be able to judge the scope of a change from the file paths alone.
5. **Format and lint are gates, not opinions.** `terraform fmt`, `tflint`, and `terraform validate` run in CI and block merges. Style debates ended when the check went red.

## Standard

### Workload infrastructure lives in the workload repository

Application workloads (`code-review`, `billing`) keep their Terraform in an `infra/` directory at the root of the application repository. The application and its infrastructure share a lifecycle — a new queue and the code that consumes it should land in the same pull request.

Two kinds of Terraform get a **dedicated repository** instead:

- **Platform infrastructure** — capabilities that serve every workload and have no single application to live with: `platform-network`, `platform-tfstate`, `platform-acr`, `platform-monitor`. Each is its own repository named for the capability.
- **Shared reusable modules** — modules consumed by two or more workloads live in dedicated repositories named `terraform-azurerm-<purpose>` (e.g. `terraform-azurerm-container-app`), released with SemVer tags. See [Modules](modules.md) for promotion criteria and versioning.

### The standard `infra/` layout

```text
billing/                                # the workload repository
├── src/                                # application code (not covered here)
├── .github/workflows/                  # CI + deploy pipelines
└── infra/
    ├── modules/                        # local modules, specific to this workload
    │   └── billing-database/           #   e.g. SQL server + db + private endpoint composition
    │       ├── main.tf
    │       ├── variables.tf
    │       ├── outputs.tf
    │       └── README.md
    └── environments/
        ├── dev/
        │   ├── main.tf                 # module composition for dev
        │   ├── backend.tf              # azurerm backend → tfstate-dev / billing.tfstate
        │   ├── providers.tf            # provider config, subscription_id for dev
        │   ├── variables.tf            # variable declarations (identical across envs)
        │   ├── terraform.tfvars        # dev values — the ONLY per-env differences
        │   └── outputs.tf              # the workload's published interface
        ├── stg/
        │   └── ...                     # same five files, stg backend, stg values
        └── prod/
            └── ...                     # same five files, prod backend, prod values
```

### What every root module directory must contain

Each `environments/<env>/` directory is an independent Terraform **root module** and must contain, at minimum:

| File | Contents |
|------|----------|
| `main.tf` | Module calls and any root-level resources. Thin — composition only. |
| `backend.tf` | The `azurerm` backend block for this environment's state. See [Remote State](remote-state.md). |
| `providers.tf` | `terraform` block with `required_version` and `required_providers`, plus the provider block with an explicit `subscription_id`. See [Providers](providers.md). |
| `variables.tf` | Variable declarations. Structurally identical across all three environments. |
| `terraform.tfvars` | This environment's values. The only file that should meaningfully differ between environments. |
| `outputs.tf` | Root outputs — the workload's public interface. See [Outputs](outputs.md). |

The `providers.tf` content may live in `main.tf` for very small stacks, but the backend always gets its own `backend.tf` so the state location is findable without reading anything else.

The three environment directories must stay **structurally identical** — same files, same modules, same variables — differing only in `backend.tf` pointers and `terraform.tfvars` values. Why this matters, and why we use directories rather than workspaces at all, is covered in [Environments](environments.md).

### CI gates

Every repository containing Terraform runs these checks on every pull request, and they are required status checks (see [GitHub Actions](../github/github-actions.md)):

```bash
terraform fmt -check -recursive     # canonical formatting, no exceptions
tflint --recursive                  # lint: deprecated syntax, unused declarations, azurerm rules
terraform validate                  # per root, after init -backend=false
```

`terraform plan` also runs per environment on pull requests and posts its output to the PR — the plan is the real review artifact. A PR that fails `fmt -check` is not reviewed until it passes; formatting noise in diffs is a tax on every reviewer.

## Examples

A minimal `environments/dev/main.tf` for the `code-review` workload, showing the thin-root pattern:

```hcl
module "container_app" {
  source = "git::https://github.com/avlon-technologies/terraform-azurerm-container-app.git?ref=v1.2.0"

  workload     = "code-review"
  environment  = "dev"
  location     = "canadacentral"
  min_replicas = var.min_replicas
  max_replicas = var.max_replicas
  tags         = var.tags
}
```

The root contributes no logic: names, wiring, and secure defaults come from the module; scale comes from `terraform.tfvars`. Promoting this to stg means the same file in `environments/stg/` with different tfvars — nothing else.

## Anti-patterns

| Anti-pattern | Why it's wrong |
|--------------|----------------|
| A single `infra/main.tf` with `count`/conditionals switching on environment | One state file for all environments, one blast radius, unreviewable plans. See [Environments](environments.md). |
| Terraform scattered at repo root, or in `deploy/`, `iac/`, `tf/` | Breaks the predictability that is the entire point. `infra/` is the name. |
| A separate `billing-infra` repository for one workload's infra | Splits one lifecycle across two repos and two PRs; app and infra changes can no longer land atomically. Dedicated repos are for platform capabilities and shared modules only. |
| Copying a shared module's source into `infra/modules/` | Forks the module; fixes upstream never arrive. Consume shared modules by pinned git source. |
| Environment directories that have drifted structurally (extra resources in prod only) | Prod-only resources are untested by definition. Differences belong in tfvars values. |
| Committing `.terraform/` or `*.tfstate` files | Provider binaries bloat the repo; state contains secrets. Both are `.gitignore`d, always. |

## Tradeoffs

- **Three copies of near-identical root files.** Directory-per-environment means `variables.tf` and `main.tf` exist three times. This is deliberate duplication — it buys independent state, independent RBAC, and reviewable per-environment diffs — but it does require discipline (and review attention) to keep the copies aligned.
- **`infra/` in the app repo couples pipelines.** Application CI and infrastructure CI share a repository, so path filters in workflows are required to keep app-only changes from triggering plans. We accept the filter complexity for atomic app+infra changes.
- **Strict gates slow the first commit.** `fmt`/`tflint`/`validate` as required checks add minutes and occasional friction. They repay it by making every subsequent diff pure signal.

## Related Standards

- [Modules](modules.md) — what goes in `modules/` vs. a `terraform-azurerm-*` repo
- [Environments](environments.md) — why directory-per-environment, and keeping roots symmetric
- [Remote State](remote-state.md) — what goes in each `backend.tf`
- [Repository Structure](../github/repository-structure.md) — the surrounding repository standard
- [Terraform Naming](../naming/terraform.md) — naming for modules, files, and state keys
