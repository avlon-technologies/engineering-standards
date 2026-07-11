# Terraform Naming

**Status:** Active
**Applies to:** All Terraform code at Avlon — module repositories, local modules, resource and data blocks, variables, outputs, locals, state keys, and tfvars files.

---

## Purpose

Terraform code has two naming surfaces: the **Azure names** it generates (governed by
[Azure Resource Naming](azure.md)) and the **identifiers inside the code itself** — resource
block labels, variables, outputs, state keys. The second surface is this document. It matters
because Terraform identifiers are an API: `azurerm_key_vault.main` appears in plan output, state
addresses, `terraform import` commands, and cross-module references. Renaming an identifier
later means a state move (`terraform state mv`) — mechanical but risky, and always at the worst
time. Consistent identifiers also make plan diffs reviewable: a reviewer who sees
`azurerm_subnet.app` knows what changed without opening the file.

## Guiding Principles

1. **snake_case because HCL demands it.** This is the sanctioned exception to Avlon's
   kebab-case house style ([General Naming Principles](general.md)): identifiers are
   snake_case; the *values* they hold (`"billing-api"`, `"prod"`) stay kebab-case.
2. **Name by role, not by type.** The resource type is already in the address —
   `azurerm_subnet.subnet` says nothing twice; `azurerm_subnet.app` says something once.
3. **Identifiers read as English in plan output.** `azurerm_resource_group.main will be
   created` should be a sentence a reviewer can approve.
4. **State keys are derivable from the workload name.** Anyone who knows the workload and
   environment can locate its state without asking.

## Standard

### Module repositories and local modules

Shared reusable modules live in dedicated repositories named `terraform-azurerm-<purpose>` —
e.g. `terraform-azurerm-container-app`, `terraform-azurerm-key-vault`. The prefix is the
community and registry convention for provider-specific modules, and it makes reusable
infrastructure instantly recognizable in the org's repository list. They are versioned with
SemVer tags and consumed with pinned refs (see [Modules](../terraform/modules.md)):

```hcl
module "container_app" {
  source = "git::https://github.com/avlon/terraform-azurerm-container-app.git?ref=v1.2.0"
  # ...
}
```

Local, workload-specific modules live under `modules/<purpose>` inside the workload repo's
`infra/` tree (see [Repository Layout](../terraform/repository-layout.md)):

```text
infra/
├── modules/
│   ├── database/          # kebab-case directory, named for what it provides
│   └── frontend/
└── environments/
    ├── dev/
    ├── stg/
    └── prod/
```

Module directory names describe the capability (`database`), not the implementation
(`sql-server-and-private-endpoint`). The module *call* label is snake_case and matches the
directory: `module "database" { source = "../../modules/database" }`.

### Resource and data blocks

Resource and data block labels are snake_case and named by **role within the configuration**:

- **`main` for the singleton.** When a configuration has exactly one resource of a type filling
  the obvious central role, call it `main`: `azurerm_resource_group.main`,
  `azurerm_key_vault.main`. `main` is a deliberate choice meaning "the one and only" — unlike
  `this`, which is often pasted from module boilerplate without thought. If you cannot say why
  a resource is the main one, it needs a role name instead.
- **Role names when there is more than one** (or when the role is more specific than "the
  one"): `azurerm_subnet.app`, `azurerm_subnet.data`, `azurerm_linux_web_app.api`,
  `azurerm_linux_web_app.web`.
- **Never restate the type.** `azurerm_subnet.app_subnet` and `azurerm_storage_account.storage`
  duplicate what the address already says. The label answers "which one?", not "what is it?".

```hcl
resource "azurerm_resource_group" "main" { ... }
resource "azurerm_virtual_network" "main" { ... }
resource "azurerm_subnet" "app" { ... }
resource "azurerm_subnet" "data" { ... }
resource "azurerm_mssql_server" "main" { ... }
resource "azurerm_private_endpoint" "sql" { ... }

data "azurerm_client_config" "current" { ... }
data "azurerm_container_registry" "platform" {
  name                = "acrplatformsharedcc01"
  resource_group_name = "rg-platform-acr-shared-cc-01"
}
```

Data sources referencing shared platform infrastructure take the platform capability as their
role (`platform` above); `current` is conventional for ambient context like
`azurerm_client_config`.

### Variables

Variables are snake_case, always typed and described, with `validation` blocks for constrained
values (see [Variables](../terraform/variables.md)). Naming rules:

- **No negative booleans.** `enable_public_access = false` reads directly;
  `disable_public_access = false` is a double negative that *will* be misread in review. Name
  the positive capability and default it appropriately.
- **Units belong in the name.** `retention_days = 30`, `timeout_seconds = 300`,
  `max_size_gb = 250`. A bare `retention = 30` forces every reader to check the provider docs
  to learn whether that is days or hours.
- **The standard naming inputs are `workload`, `environment`, `location`, `instance`** — the
  exact variables the name-generation locals in [Azure Resource Naming](azure.md) consume. Do
  not alias them (`app_name`, `env`) per repo; module reuse depends on the shared vocabulary.

```hcl
variable "environment" {
  description = "Deployment environment."
  type        = string

  validation {
    condition     = contains(["dev", "stg", "prod"], var.environment)
    error_message = "environment must be one of: dev, stg, prod."
  }
}

variable "enable_public_access" {
  description = "Allow public network access (dev only; stg/prod use private endpoints)."
  type        = bool
  default     = false
}

variable "log_retention_days" {
  description = "Log Analytics retention period in days."
  type        = number
  default     = 30
}
```

### Outputs

Outputs follow `<object>_<attribute>` — the thing, then the property:

```hcl
output "key_vault_uri" {
  description = "URI of the workload Key Vault."
  value       = azurerm_key_vault.main.vault_uri
}

output "container_app_fqdn" {
  description = "Public FQDN of the container app."
  value       = azurerm_container_app.main.ingress[0].fqdn
}

output "sql_server_fqdn" {
  value = azurerm_mssql_server.main.fully_qualified_domain_name
}
```

Outputs are a module's public API — deliberate and minimal (see
[Outputs](../terraform/outputs.md)). The two-part name means a consumer reading
`module.database.sql_server_fqdn` knows both what object and what attribute they hold, without
opening the module. Never name an output after its internal resource label (`main_uri`) or emit
whole resource objects (`sql_server`) — attributes only, so the module's surface stays explicit.

### Locals

Locals are snake_case and named for what they *are*, not how they are computed: `suffix`,
`suffix_compact`, `region`, `common_tags`. The canonical use is name derivation — the
`local.suffix` pattern defined in [Azure Resource Naming](azure.md#generating-names-in-terraform).
A local used once, immediately, may not deserve to exist; a local used by every resource
(`common_tags`, `suffix`) is exactly what locals are for.

### State keys

One state file per deployable workload per environment (see
[Remote State](../terraform/remote-state.md)). State lives in the platform storage account
`sttfstatesharedcc01` with **one container per environment** and the blob key
`<workload>.tfstate`:

```text
Storage account: sttfstatesharedcc01        (rg-platform-tfstate-shared-cc-01)
├── tfstate-dev/
│   └── code-review.tfstate
├── tfstate-stg/
│   └── billing.tfstate
├── tfstate-prod/
│   └── billing.tfstate
└── tfstate-shared/
    └── platform-acr.tfstate               # platform capability states
```

The container carries the environment, so the key does not repeat it — `billing.tfstate` in
`tfstate-prod`, not `billing-prod.tfstate`. Per-environment containers also give RBAC a clean
boundary: the dev deployment identity can be scoped to `tfstate-dev` alone.

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-platform-tfstate-shared-cc-01"
    storage_account_name = "sttfstatesharedcc01"
    container_name       = "tfstate-prod"
    key                  = "billing.tfstate"
    use_azuread_auth     = true
  }
}
```

### tfvars files

Each environment directory carries exactly one `terraform.tfvars` (auto-loaded, no `-var-file`
flag to forget). The file's identity comes from its directory — `environments/prod/terraform.tfvars`
— never from its name, so there is no `prod.tfvars` sitting next to `dev.tfvars` waiting to be
applied against the wrong backend. Supplementary variable files, where genuinely needed, are
`<purpose>.auto.tfvars` (kebab-case is illegal here; filenames use the identifier's snake_case
only if multi-word, e.g. `scaling.auto.tfvars`). Secrets never appear in any tfvars file — see
[Terraform Security](../terraform/security.md).

## Examples

A complete, conventionally named root module for `billing` in prod
(`infra/environments/prod/`):

```hcl
# backend.tf — state key derived from workload + environment container
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-platform-tfstate-shared-cc-01"
    storage_account_name = "sttfstatesharedcc01"
    container_name       = "tfstate-prod"
    key                  = "billing.tfstate"
    use_azuread_auth     = true
  }
}

# main.tf
module "database" {
  source = "../../modules/database"

  workload    = var.workload      # "billing"
  environment = var.environment   # "prod"
  location    = var.location
}

module "key_vault" {
  source = "git::https://github.com/avlon/terraform-azurerm-key-vault.git?ref=v2.1.0"

  name                = "kv-${local.suffix}"          # kv-billing-prod-cc-01
  resource_group_name = azurerm_resource_group.main.name
}

# outputs.tf
output "key_vault_uri" {
  value = module.key_vault.key_vault_uri
}
```

Every identifier is predictable: the reviewer of a plan touching
`module.database.azurerm_mssql_server.main` knows the workload's one SQL server changed, and
anyone can find its state at `tfstate-prod/billing.tfstate` from the workload name alone.

## Anti-patterns

| Anti-pattern | Why it fails |
|--------------|--------------|
| `azurerm_subnet.app_subnet`, `azurerm_storage_account.storage` | Restates the type the address already carries. |
| `azurerm_key_vault.this` (unthinkingly) | `this` pasted from boilerplate carries no meaning; use `main` for the deliberate singleton, a role name otherwise. |
| `azurerm_resource_group.rg_billing_prod` | Environment and workload baked into the identifier — the same code can no longer deploy every environment. |
| `variable "disable_backups"` | Negative boolean; `enable_backups` reads without a double negative. |
| `variable "retention"` | No unit; is it days, hours, GB? |
| `output "main_uri"` | Names the internal resource label, not the object; consumers can't tell what it is. |
| State key `billing-prod-final.tfstate` | Not derivable; environment belongs to the container, not the key. |
| `prod.tfvars` and `dev.tfvars` side by side | Invites applying the wrong file; environment identity comes from the directory. |

## Tradeoffs

- **`main` is information-light.** A file with `azurerm_resource_group.main` tells you less
  than a richly named label might. We accept it because singletons dominate, and one universal
  word beats every team inventing its own (`rg`, `this`, `primary`, `default`).
- **Renaming identifiers requires state moves.** Naming by role means an identifier can go
  stale when a resource's role genuinely changes. That is rare, and `terraform state mv` exists;
  the alternative — vague names that never go stale because they never meant anything — costs
  more every day.
- **Strict input names (`workload`, `environment`) constrain module authors.** A module can't
  call its input `app`. The payoff is composability: any Avlon module wires into any Avlon root
  module without an adapter layer of renamed variables.

## Related Standards

- [General Naming Principles](general.md) — the kebab-case/snake_case boundary
- [Azure Resource Naming](azure.md) — generating the Azure names this code produces
- [Modules](../terraform/modules.md) — module structure, versioning, and publishing
- [Remote State](../terraform/remote-state.md) — the state architecture behind the key scheme
- [Variables](../terraform/variables.md) and [Outputs](../terraform/outputs.md) — full standards for module interfaces
