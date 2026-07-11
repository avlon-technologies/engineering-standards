# Terraform Modules

**Status:** Active
**Applies to:** All Terraform modules at Avlon — local modules inside workload repositories and shared modules in `terraform-azurerm-*` repositories.

---

## Purpose

Modules are where Terraform either pays off or rots. Done well, a module encodes a decision once — how Avlon runs a Container App, how a SQL database gets its private endpoint — and every workload inherits it. Done badly, modules become grab-bags of loosely related resources with fifty variables, or thin wrappers that add a layer of indirection and nothing else. This standard defines what a module is for, what its contract looks like, where it lives, and when it graduates from local to shared.

## Guiding Principles

1. **Modules solve categories, not instances.** A module answers "how does Avlon deploy a Container App?" — never "how do I deploy the billing app?". If a module's name contains a workload name and it lives outside that workload's repo, it is an instance masquerading as an abstraction.
2. **Small, composable, single-responsibility.** A module should do one thing a caller can name in a sentence. Roots compose small modules; nobody debugs a 40-resource mega-module at 2 a.m.
3. **The contract is variables in, outputs out.** A module must not reach outside itself: no provider blocks, no backend config, no assumptions about where it runs. Everything it needs arrives through variables; everything it offers leaves through outputs.
4. **Secure by default.** A caller who supplies only the required variables must get a secure configuration — TLS enforced, public access off, diagnostics on. Insecurity must be an explicit, visible opt-out in the caller's code, never the silent default.
5. **A module without documentation is unfinished.** If consumers must read `main.tf` to use it, the module isn't done.

## Standard

### Module contract

- **No `provider` blocks inside modules.** Modules declare version constraints via `required_providers` only; provider configuration belongs exclusively to root modules (see [Providers](providers.md)). A module with an embedded provider block cannot be called twice, cannot use aliased providers, and hijacks configuration that belongs to the caller.
- Every variable is typed, described, and validated where the input is constrained; every output is described. See [Variables](variables.md) and [Outputs](outputs.md).
- Defaults follow the security principle above: omitting an optional variable must never produce a less secure resource than the Avlon baseline (private endpoints in stg/prod, `min_tls_version = "TLS1_2"`, public network access disabled — see [Networking](../azure/networking.md)).
- Resource names inside modules are **derived, never hardcoded**, from `workload`, `environment`, `location`, and `instance` inputs, per [Azure Naming](../naming/azure.md).

### Local modules vs. shared modules

**Local modules** live at `infra/modules/<purpose>` inside the workload repository (see [Repository Layout](repository-layout.md)). They exist to keep environment roots thin and to compose shared modules into workload-specific shapes. They are versioned implicitly with the repo and consumed by relative path:

```hcl
module "database" {
  source = "../../modules/billing-database"
  # ...
}
```

**Shared modules** live in dedicated repositories named `terraform-azurerm-<purpose>` (e.g. `terraform-azurerm-container-app`, `terraform-azurerm-sql-database`) in the `avlon` GitHub organization. They are released with **SemVer git tags** (`v1.2.0` — see [Releases](../github/releases.md)) and consumed only by pinned tag:

```hcl
module "container_app" {
  source = "git::https://github.com/avlon/terraform-azurerm-container-app.git?ref=v1.2.0"
  # ...
}
```

Never consume a shared module by branch (`?ref=main`) — a moving ref means two plans of the same code can produce different infrastructure. Version bumps are deliberate PRs in the consuming repo, where the plan shows exactly what the new version changes.

### When to promote a local module to shared

Promote when a **second workload needs it**. One consumer is a local module; two consumers copying it is a fork waiting to diverge. Promotion means: new `terraform-azurerm-<purpose>` repository, generalize any workload-specific assumptions into variables, tag `v1.0.0`, and switch both consumers to the pinned source. Do not pre-emptively build shared modules for hypothetical consumers — you don't know the right abstraction until the second real caller shows up.

### Required documentation

Every module — local or shared — has a `README.md` containing:

- One paragraph: what the module manages and the opinionated decisions it bakes in.
- A copy-pasteable **usage example** with realistic values.
- **Inputs and outputs tables**, generated with [`terraform-docs`](https://terraform-docs.io/) so they cannot drift from the code. Shared module repos regenerate them in CI and fail the build on a dirty diff.

### Testing

- Minimum bar for all modules: `terraform fmt -check`, `terraform validate`, and a successful `terraform plan` against a **fixture** — a small root in `examples/` or `tests/` that exercises the module with realistic inputs. The fixture doubles as the README usage example.
- Where behavior is worth asserting (conditional resource creation, derived naming, validation rules), use native [`terraform test`](https://developer.hashicorp.com/terraform/language/tests) with `command = plan` runs. Shared modules with non-trivial logic should have tests; a tagged release with a broken plan breaks every consumer at once.

## Examples

A shared module's shape (`terraform-azurerm-container-app`):

```text
terraform-azurerm-container-app/
├── main.tf            # the resources
├── variables.tf       # typed, described, validated inputs
├── outputs.tf         # the public API
├── versions.tf        # required_version + required_providers (no provider block)
├── README.md          # usage + terraform-docs inputs/outputs
├── examples/basic/    # plan fixture = usage example
└── tests/             # terraform test files where valuable
```

Its `versions.tf` — note the constraint style, explained in [Providers](providers.md):

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

And a secure-by-default variable:

```hcl
variable "public_network_access_enabled" {
  description = "Allow public ingress to the Container App. Leave false; enable only in dev with a documented reason."
  type        = bool
  default     = false
}
```

## Anti-patterns

| Anti-pattern | Why it's wrong |
|--------------|----------------|
| `provider "azurerm" {}` inside a module | Breaks multi-instantiation and aliasing; steals root-module responsibility. Modules declare `required_providers` only. |
| Wrapper modules that add nothing (`module "rg"` wrapping `azurerm_resource_group` 1:1) | Pure indirection: one more repo to version, one more README to read, zero encoded decisions. If a module adds no opinion, call the resource directly. |
| A module named `terraform-azurerm-billing` | That's an instance, not a category. Workload composition belongs in the workload's local modules. |
| Consuming shared modules via `?ref=main` or no ref | Unpinned source; plans stop being reproducible. Pin exact tags, upgrade via PR. |
| Insecure defaults "for convenience" (`public_network_access_enabled = true` by default) | Every caller who forgets a variable ships an insecure resource. Secure is the default; insecure is opt-in. |
| One mega-module deploying app + database + networking + monitoring | Unreviewable plans, all-or-nothing changes, and no reuse — the opposite of composability. |
| Copying a shared module's code into a workload repo to "tweak one thing" | A silent fork. Add a variable upstream instead, or compose around it locally. |

## Tradeoffs

- **Pinned versions mean stale consumers.** Pinning to exact tags makes upgrades a manual PR per consumer, and consumers will lag. We accept lag over the alternative — unpinned modules changing infrastructure underneath every workload simultaneously. Dependabot-style bump PRs keep the lag honest.
- **The two-consumer rule delays reuse.** The first workload builds a local module the second workload could have used. Extracting after the second consumer appears costs a small migration — cheaper than maintaining speculative shared modules that abstracted the wrong thing.
- **Documentation and fixtures are real overhead.** `terraform-docs` and plan fixtures add work per module. It is the price of modules being consumable without reading their source, which is the entire value of a shared module.

## Related Standards

- [Repository Layout](repository-layout.md) — where local modules live
- [Providers](providers.md) — version constraints in modules vs. roots
- [Variables](variables.md) / [Outputs](outputs.md) — the two halves of the module contract
- [Releases](../github/releases.md) — SemVer tagging for module repositories
- [Terraform Naming](../naming/terraform.md) — module, resource, and repository naming
