# Terraform Environments

**Status:** Active
**Applies to:** All Terraform configurations that deploy to more than one environment — which is all of them.

---

## Purpose

Every workload deploys to `dev`, `stg`, and `prod` (defined canonically in [Azure Environments](../azure/environments.md)). Terraform offers two ways to model this — workspaces or separate root directories — and the choice is one of the most consequential structural decisions in a Terraform estate. This standard mandates **directory-per-environment** and explains why. Get this wrong and you get shared blast radii, invisible environment context, and RBAC that can't distinguish a dev deploy from a prod one.

## Guiding Principles

1. **Environments are isolation boundaries, not variations.** dev, stg, and prod differ in who can touch them, what breaks when they break, and which subscription they live in. That demands separate state, separate credentials, and separate pipelines — not a CLI flag.
2. **Identical in shape, different in values.** Environments run the same modules at the same versions with different tfvars. Anything that exists only in prod is untested by definition.
3. **The current environment must be impossible to mistake.** The directory you are in *is* the environment. No hidden mode, no `terraform workspace show` to remember.
4. **Promotion is repetition, not modification.** A change reaches prod by applying the exact configuration already proven in dev and stg, with prod's values.
5. **Symmetry is maintained, not assumed.** Environment roots drift unless reviewers actively hold them identical. Drift between roots is a bug, not a style issue.

## Standard

### Directory-per-environment

Each environment is an independent root module at `infra/environments/<env>/` with its own `backend.tf`, `terraform.tfvars`, and provider configuration (full layout in [Repository Layout](repository-layout.md)):

```text
infra/environments/
├── dev/     # own state: tfstate-dev / billing.tfstate — sub-avlon-dev
├── stg/     # own state: tfstate-stg / billing.tfstate — prod subscription, isolated by RG/RBAC
└── prod/    # own state: tfstate-prod / billing.tfstate — sub-avlon-prod
```

Every root declares the same variables and calls the same modules **at the same pinned versions**. The differences live in exactly two places: `backend.tf` (which state) and `terraform.tfvars` (which values — SKUs, replica counts, capacity, feature toggles). A diff of `dev/main.tf` against `prod/main.tf` should be empty or nearly so.

### Why not workspaces

Terraform workspaces are explicitly **not used** for environment separation at Avlon:

- **Shared backend = shared blast radius.** All workspaces of a configuration share one backend block, so every environment's state sits in one container behind one set of credentials. A backend misconfiguration, a compromised credential, or an errant `terraform state` command has all environments in scope at once.
- **Invisible environment context.** The active workspace is ambient CLI state. `terraform apply` looks identical whether you're in `dev` or `prod`; the only guard is remembering to check. With directories, the environment is in your prompt, your muscle memory, and the file path of every reviewed change.
- **RBAC cannot be scoped per environment.** We grant the dev CI identity access to `tfstate-dev` and nothing else (see [Remote State](remote-state.md)). Workspaces store all environments' state under one container, making that separation impossible — anyone who can plan dev can read prod state, and prod state contains secrets.
- **Workspaces breed conditional environment logic.** Once `terraform.workspace` is available, `count = terraform.workspace == "prod" ? 3 : 1` follows, and the configuration becomes a branching program where no environment runs code any other environment has tested. Directory roots keep the code identical and push differences into data (tfvars).
- **No per-environment version skew control.** With one configuration, you cannot hold prod at a known-good module version while dev trials the next one. Directory roots make version rollout an explicit, reviewable, per-environment step.

Workspaces remain acceptable for genuinely ephemeral copies of a *single* environment (e.g. short-lived PR preview stacks in dev) — never as the boundary between dev, stg, and prod.

### Promotion

A change flows `dev → stg → prod` as the **same code and module versions** applied with each environment's tfvars:

1. PR updates the module version or configuration in `environments/dev/` (or all three at once when review capacity allows — apply order still holds).
2. CI plans; reviewer approves; merge applies to dev. Validate.
3. Repeat for `stg`, then `prod`, gated by GitHub environment protection rules ([GitHub Actions](../github/github-actions.md)).

Prod never receives a module version that stg hasn't run. If stg exposed a problem, fix it in dev first — never patch prod directly and back-port.

### Keeping roots symmetric

Because the three roots are separate files, symmetry is enforced by review, not by the tool:

- Any PR touching one environment root must either touch all three identically or state in the description why the change is intentionally staged (usually: promotion in flight).
- Reviewers diff roots against each other, not just against `main`. A simple symmetry check in CI — `diff` of `main.tf`/`variables.tf` across environment directories, ignoring `backend.tf` and `terraform.tfvars` — turns drift from a code-review hope into a failing status check, and is recommended for repos that have been burned.
- Structural drift found later is treated as a bug: raise an issue, converge the roots.

## Examples

The *only* interesting difference between environments — `terraform.tfvars`:

```hcl
# environments/dev/terraform.tfvars
environment  = "dev"
sql_sku_name = "GP_S_Gen5_1"   # serverless, cheap, auto-pause
min_replicas = 0
max_replicas = 2

# environments/prod/terraform.tfvars
environment  = "prod"
sql_sku_name = "GP_Gen5_4"     # provisioned, prod-parity with stg
min_replicas = 2
max_replicas = 10
```

Same variables, same modules, same shape — different scale. The `environment` value also drives resource naming (`rg-billing-prod-cc-01` vs `rg-billing-dev-cc-01`) via the derivation pattern in [Azure Naming](../naming/azure.md).

## Anti-patterns

| Anti-pattern | Why it's wrong |
|--------------|----------------|
| `terraform workspace select prod && terraform apply` | Everything in "Why not workspaces": shared backend, invisible context, unscopeable RBAC. |
| `count = var.environment == "prod" ? 1 : 0` sprinkled through code | Environments stop running the same code; prod paths are never exercised before prod. Push differences into tfvars values, not code branches. |
| A resource added only to `environments/prod/main.tf` | Untested by construction, and the roots have now drifted. Add it to all three; disable via tfvars where an env genuinely doesn't need it. |
| Hand-editing prod to hotfix, promoting later ("fix-forward-backwards") | Prod now runs code that exists nowhere in git history's promotion path; the next apply may revert the fix. Fix in dev, promote fast. |
| Skipping stg because "it's just a tag change" | Every prod incident starts as a change someone judged too small to stage. The pipeline is the judgment. |
| An extra `environments/mike-test/` root | Improvised environments have no RBAC story, no naming code, no teardown owner. Ephemeral experimentation belongs in dev. |

## Tradeoffs

- **Duplication is real.** Three near-identical roots per workload, and adding a variable means touching three files. We accept ~200 duplicated lines per workload to get hard state isolation, per-environment RBAC, and explicit promotion — the things workspaces cannot give us.
- **Promotion is slower than a single apply.** Three sequenced applies with gates instead of one. That latency *is* the safety mechanism; urgent fixes ride the same rails, just faster.
- **Symmetry needs policing.** Nothing in Terraform itself keeps the roots identical; review discipline (or a CI symmetry check) carries that weight. It is a maintenance cost we pay knowingly.

## Related Standards

- [Azure Environments](../azure/environments.md) — the canonical dev/stg/prod/shared definitions
- [Repository Layout](repository-layout.md) — the directory structure these roots live in
- [Remote State](remote-state.md) — per-environment state containers and RBAC
- [GitHub Actions](../github/github-actions.md) — environment protection rules gating stg/prod
- [Variables](variables.md) — tfvars as the only home of environment-specific values
