# Variables

**Status:** Active
**Applies to:** Variable declarations and tfvars files in all Avlon Terraform roots and modules.

---

## Purpose

Variables are the entire input surface of a Terraform configuration — the module contract's front door and the only legitimate home of environment differences. Undisciplined variables produce the familiar failure modes: untyped inputs that accept garbage until apply time, undescribed inputs that force consumers to read the source, defaults that silently deploy dev-sized (or dev-secured) infrastructure to prod, and secrets smuggled through tfvars into state and git. This standard makes every variable typed, described, validated where constrained, and defaulted only where a default is safe.

## Guiding Principles

1. **Fail at plan, not at apply.** Types and validation blocks catch bad input in seconds, before anything touches Azure. Every constraint the code knows about should be enforced where it's cheapest.
2. **A variable's declaration is its documentation.** Type plus description must be enough to use it correctly without reading `main.tf`.
3. **Defaults are policy.** A default is what happens when someone isn't paying attention — so module defaults are secure and sensible, and environment-differentiating values have no defaults at all.
4. **Secrets don't travel through variables.** tfvars are committed to git and variable values land in state. Neither is a secret store.
5. **One home for environment differences.** If dev and prod differ in a value, that value lives in `terraform.tfvars` — not in conditionals, not in locals, not in defaults.

## Standard

### Declaration basics

- Names are **`snake_case`**, descriptive, unabbreviated (`min_replicas`, not `minreps`). Full naming rules in [Terraform Naming](../naming/terraform.md).
- **Every variable has an explicit `type`** — never rely on the implicit `any`. Prefer precise types (`number`, `bool`, `list(string)`, typed `object(...)`) over `string`-and-parse.
- **Every variable has a `description`** — written for the consumer, stating meaning, units, and constraints ("Maximum replica count (1–30)"), not restating the name.
- **No negative booleans.** `public_network_access_enabled`, never `disable_public_network` — negated flags produce double-negative call sites (`disable_x = false`) that reviewers misread under pressure.

### Validation blocks

Add `validation` wherever the acceptable values are knowable. The two canonical examples every workload uses, matching [Azure Naming](../naming/azure.md):

```hcl
variable "environment" {
  description = "Deployment environment code. See azure/environments.md."
  type        = string

  validation {
    condition     = contains(["dev", "stg", "prod"], var.environment)
    error_message = "environment must be one of: dev, stg, prod."
  }
}

variable "workload" {
  description = "Short workload name used in all resource names (lowercase, hyphens, max 12 chars)."
  type        = string

  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{1,11}$", var.workload))
    error_message = "workload must be lowercase alphanumeric/hyphens, start with a letter, max 12 chars."
  }
}
```

These two validations are the difference between `terraform plan` rejecting `environment = "staging"` instantly and a half-created stack of misnamed resources that automation, budgets, and RBAC can't see. Validate similarly for SKUs from a known set, CIDR shapes, port ranges — anywhere "valid" is expressible.

### Defaults policy

Two opposite rules, by module kind:

- **Reusable modules default to secure and sensible.** A caller supplying only required inputs gets the Avlon baseline: TLS enforced, public access off, diagnostics on, modest-but-safe capacity ([Modules](modules.md)). Optional knobs exist for the exceptions.
- **Root modules do NOT default environment-differentiating values.** SKU, replica counts, capacity, `environment` itself — anything that legitimately differs between dev and prod is declared without a default in the root, forcing the value to appear explicitly in that environment's `terraform.tfvars`. A default here is an invisible environment configuration: prod silently running dev's SKU because nobody wrote a value is exactly the failure this rule exists to prevent.

### tfvars discipline

**`environments/<env>/terraform.tfvars` is the ONLY place environment-specific values live** (see [Environments](environments.md)). Not in per-env locals, not in `var.environment == "prod" ? ... : ...` expressions, not in pipeline `-var` flags (pipeline-supplied values are limited to identity/subscription plumbing that CI owns). This makes "how does prod differ from dev?" answerable by diffing two small files — which is also precisely what a reviewer checks during promotion.

### Sensitive inputs — and why secrets still don't go in tfvars

Mark secrets-adjacent inputs `sensitive = true` so their values are redacted from plan output and console:

```hcl
variable "alerting_webhook_url" {
  description = "Webhook URL for alert routing (contains an embedded token)."
  type        = string
  sensitive   = true
}
```

But understand what `sensitive` does not do: the value still lands in **state in plaintext**, and if it came from tfvars it's in **git**. Therefore actual secrets — passwords, keys, connection strings — never arrive via variables at all. They flow from Key Vault at deploy or run time: a `data "azurerm_key_vault_secret"` lookup, or better, an app-side Key Vault reference so Terraform never touches the value (full flow in [Security](security.md)). `sensitive = true` is hygiene for the borderline cases, not permission to pass secrets.

### Group values that travel together

When several settings always move as a unit, make them a typed object instead of a flat namespace of prefixed variables:

```hcl
variable "sql_database" {
  description = "SQL database sizing and retention configuration."
  type = object({
    sku_name                    = string
    max_size_gb                 = number
    backup_retention_days       = optional(number, 7)
    zone_redundant              = optional(bool, true)
  })
}
```

One coherent value in tfvars, one argument at the call site, and `optional()` carries the secure defaults. Don't overdo it — grouping unrelated settings into a kitchen-sink object just moves the mess.

## Examples

The billing prod root's `terraform.tfvars` — every environment-differentiating value, explicit:

```hcl
environment  = "prod"
workload     = "billing"
location     = "canadacentral"

sql_database = {
  sku_name    = "GP_Gen5_4"
  max_size_gb = 250
}

min_replicas = 2
max_replicas = 10

tags = {
  workload    = "billing"
  environment = "prod"
  owner       = "billing-team"
  cost-center = "cc-1204"
  managed-by  = "terraform"
  repository  = "https://github.com/avlon-technologies/billing"
}
```

## Anti-patterns

| Anti-pattern | Why it's wrong |
|--------------|----------------|
| Untyped variables (implicit `any`) or missing descriptions | Garbage in until apply time; consumers forced to read the source. Type and describe everything. |
| `default = "dev"` on `environment` in a root module | The single most dangerous default in Terraform: forget one tfvars line and prod deploys as dev — wrong names, wrong RBAC assumptions, wrong scale. |
| Constrainable inputs without `validation` | `environment = "staging"` sails through plan and produces resources the entire naming/automation estate can't see. |
| `sql_admin_password` in `terraform.tfvars` | A secret in git *and* in state. Secrets come from Key Vault; see [Security](security.md). |
| `disable_public_access = false` style negative booleans | Double-negative call sites that humans reliably misread. Name flags positively. |
| `-var 'sku=...'` in pipeline definitions for env config | Environment configuration hidden in YAML instead of tfvars; the promotion diff no longer tells the truth. |
| Six `sql_*` variables that only ever change together | A contract wider than its meaning. Group them into one typed object. |

## Tradeoffs

- **No-defaults roots are verbose.** Every environment spells out every differentiating value, and adding a variable means editing three tfvars files. That verbosity *is* the audit trail — we accept the typing to make every environment's configuration explicit and diffable.
- **Validation blocks duplicate provider knowledge.** A SKU list in a validation can lag Azure's actual offerings and needs occasional upkeep. Failing at plan instead of twenty minutes into apply pays for the maintenance.
- **Typed objects churn call sites.** Adding a field to an object type touches consumers unless `optional()` covers it. Prefer `optional()` with a secure default for additive evolution; take the breaking change when the default would be unsafe.

## Related Standards

- [Modules](modules.md) — secure defaults as the module contract
- [Environments](environments.md) — tfvars as the sole home of environment differences
- [Security](security.md) — the Key Vault secret flow that replaces secret variables
- [Terraform Naming](../naming/terraform.md) — variable naming conventions
- [Azure Naming](../naming/azure.md) — the workload/environment inputs these validations protect
