# Providers and Version Pinning

**Status:** Active
**Applies to:** Terraform core and provider configuration in every root module and reusable module.

---

## Purpose

Terraform behavior is the product of three moving parts: the Terraform CLI version, the provider versions, and the configuration itself. Only the last one is in your repo unless you pin the other two. Unpinned versions mean the same commit plans differently on two machines, a provider's breaking major release lands mid-sprint via a routine `init`, and "works in CI" stops meaning anything. This standard fixes where provider configuration may live, how versions are constrained in roots versus modules, and how upgrades happen — deliberately, in reviewed PRs, never as a surprise.

## Guiding Principles

1. **Reproducibility is non-negotiable.** The same commit must produce the same plan on every machine, today and in six months. Everything that influences a plan is pinned and committed.
2. **Roots decide versions; modules state compatibility.** Exactly one place chooses the provider version actually used — the root. Modules declare the range they work with and otherwise stay out of the way.
3. **Provider configuration is root-module territory.** Credentials, subscriptions, and features are deployment context. Modules that carry deployment context can't be reused.
4. **Upgrades are changes, not maintenance noise.** A provider bump can rewrite every resource's diff. It gets a PR, a plan, and a reviewer like any other change.
5. **Point at the right subscription explicitly.** An apply that lands in the wrong subscription is a wrong-environment incident. Ambient CLI context is not a targeting mechanism.

## Standard

### Root module configuration

Every environment root pins Terraform and providers with **pessimistic constraints** and configures the provider explicitly:

```hcl
terraform {
  required_version = "~> 1.9"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }
}

provider "azurerm" {
  features {}

  subscription_id     = var.subscription_id   # explicit — never inherited from az CLI context
  storage_use_azuread = true                  # data-plane via Entra ID, matching our key-less storage posture
}
```

`~> 4.0` allows 4.x minor/patch releases but never 5.0; `~> 1.9` holds Terraform within the 1.x series at 1.9 or later. The `features {}` block is mandatory azurerm syntax even when empty.

**`subscription_id` is explicit in every root**, supplied per environment (tfvars or the pipeline's environment configuration). The azurerm 4.x provider no longer silently adopts the Azure CLI's current subscription for exactly this reason: an engineer with `sub-avlon-dev` selected in their shell must not be able to apply the prod root into it. The dev root targets `sub-avlon-dev`, prod targets `sub-avlon-prod` — structurally, not by whatever `az account show` happens to say.

### Reusable module constraints

Reusable modules declare **minimum-bounded ranges**, not pessimistic pins:

```hcl
terraform {
  required_version = ">= 1.9, < 2.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = ">= 4.0, < 5.0"
    }
  }
}
```

The difference is deliberate. Terraform resolves a single provider version satisfying *every* constraint in the graph. If a module pinned `~> 4.6` while the root pinned `~> 4.0` and its lock file held 4.3, init fails — or worse, a module upgrade force-marches the root's provider forward. Modules must not fight the root's choice: they state the honest range they're compatible with (wide), and the root — which owns the lock file and the blast radius — picks the exact version (narrow). Modules never contain `provider` blocks at all (see [Modules](modules.md)).

### The lock file

**`.terraform.lock.hcl` is committed in every root module, always.** The lock file records the exact provider versions and checksums that `init` selected; committing it means CI, teammates, and future-you install byte-identical providers instead of "whatever satisfied the constraint that day." Constraint ranges say what *could* run; the lock file says what *does*. A plan reviewed against azurerm 4.3.0 and applied against 4.8.0 was never actually reviewed.

Generate entries for all platforms CI and engineers use, so the file doesn't churn between Windows laptops and Linux runners:

```bash
terraform providers lock -platform=linux_amd64 -platform=windows_amd64 -platform=darwin_arm64
```

Reusable module repos have no lock file of their own semantics to publish — the consuming root's lock governs — but their `examples/` fixtures commit locks like any root.

### Upgrades

Provider and Terraform upgrades are **deliberate PRs**:

1. In the workload repo, run `terraform init -upgrade` in each environment root and commit the updated `.terraform.lock.hcl` (and constraint, for a major bump).
2. CI posts plans for all environments; the reviewer reads them for provider-driven diffs (renamed attributes, new defaults, forced replacements).
3. Merge and promote dev → stg → prod like any change ([Environments](environments.md)).

Track provider releases and schedule minor bumps regularly (quarterly at minimum) so upgrades stay small; a stack that sleeps through 4.1–4.9 pays for all nine at once. **Major** version upgrades (4.x → 5.x) follow the provider's upgrade guide and get their own dedicated PR — never bundled with feature work.

### Aliased providers

When one root must touch two subscriptions or configure multi-region resources, use **provider aliases** in the root and pass them explicitly:

```hcl
provider "azurerm" {
  features {}
  subscription_id = var.subscription_id            # sub-avlon-prod — default
}

provider "azurerm" {
  alias           = "platform"
  features {}
  subscription_id = var.platform_subscription_id   # sub-avlon-platform — e.g. DNS zone records
}

module "private_dns_records" {
  source    = "git::https://github.com/avlon-technologies/terraform-azurerm-private-dns-records.git?ref=v2.0.1"
  providers = { azurerm = azurerm.platform }
  # ...
}
```

The billing prod root uses the same mechanism for its Canada East DR resources where per-resource `location` isn't sufficient. Aliases keep multi-subscription intent explicit and reviewable; the alternative — a second root with tangled cross-state coupling — is worse for what is logically one stack.

## Examples

See the root block above; the complete per-file layout of a root (`providers.tf`, `backend.tf`, …) is defined in [Repository Layout](repository-layout.md), and the backend half of the configuration in [Remote State](remote-state.md).

## Anti-patterns

| Anti-pattern | Why it's wrong |
|--------------|----------------|
| No version constraints, or `version = ">= 4.0"` alone in a root | The next major release breaks you at a time of its choosing. Roots pin pessimistically. |
| `~> 4.6` (or exact pins) inside a reusable module | Fights the root's version choice; one module bump force-upgrades every consumer's provider. Modules declare ranges. |
| `provider` blocks inside reusable modules | Breaks aliasing and multi-instantiation; embeds deployment context in reusable code. See [Modules](modules.md). |
| `.terraform.lock.hcl` in `.gitignore` | Every machine resolves providers independently; plans stop being comparable across CI and laptops. Commit it. |
| Relying on `az account set` to choose the target subscription | Ambient shell state deciding where prod applies. `subscription_id` is explicit in every root. |
| `terraform init -upgrade` in the CI pipeline | Silent continuous upgrades — the plan you review runs different provider code than yesterday's, with no PR marking the change. Upgrades are explicit commits. |
| Bundling a provider major upgrade with feature work | Two risky diffs in one plan; neither can be reverted alone. |

## Tradeoffs

- **Pinning trades freshness for predictability.** Locked versions mean bug fixes and new resources arrive only when someone runs the upgrade PR, and stale stacks accumulate upgrade debt. A standing cadence keeps the debt shallow; we accept the chore over unscheduled breakage.
- **Explicit `subscription_id` is per-environment plumbing.** One more value to supply per root, and local plans require it set. That single line is the structural guarantee against wrong-subscription applies — cheap insurance.
- **Multi-platform lock files add a step.** `terraform providers lock` with three platforms is easy to forget until CI diffs the lock file. We accept the occasional red check as the mechanism that keeps hashes honest.

## Related Standards

- [Modules](modules.md) — no provider blocks in modules; constraint style rationale
- [Repository Layout](repository-layout.md) — where provider configuration lives in a root
- [Remote State](remote-state.md) — the backend half of root configuration
- [Environments](environments.md) — per-environment roots and promotion of upgrades
- [Security](security.md) — how pipelines authenticate the provider (OIDC, no secrets)
