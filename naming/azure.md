# Azure Resource Naming

**Status:** Active
**Applies to:** All Azure resources across all subscriptions and environments. Based on the [Azure Cloud Adoption Framework naming guidance](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming), adapted and made opinionated for Avlon Technologies.

---

## Purpose

A resource name is the first — and often only — piece of metadata a human sees. Names appear in
the portal, in cost reports, in alerts, in CLI output, and in incident channels at 2 a.m. The
difference between `sql-billing-prod-cc-01` and `sql-billing-dev-cc-01` must be unmistakable,
because ambiguous names cause production incidents.

Naming is cheap to get right at creation time and expensive to fix later — most Azure resources
**cannot be renamed** and must be destroyed and recreated. This standard is therefore mandatory,
not advisory. It exists so that anyone can look at a name and know what a resource is, which
workload it belongs to, which environment it serves, and where it runs — and so Terraform,
scripts, and Azure Policy can construct and predict names from known inputs with no lookups and
no drift between environments.

## Guiding Principles

1. **Deterministic.** Given workload, environment, region, and instance, everyone — human or
   pipeline — produces the same name. See [General Naming Principles](general.md).
2. **Self-describing left to right.** Each segment narrows scope: type, then workload, then
   environment, then region, then instance. Prefix filtering in `az resource list`, cost
   queries, and Azure Policy falls out naturally.
3. **One convention that survives its exceptions.** Where Azure forbids hyphens or shortens
   limits, the same components appear in the same order — only the format adapts.
4. **Nothing volatile in the name.** Owner, subscription, dates, and meaning-laden instance
   numbers are kept out of names and pushed into [tags](../azure/tagging.md), which can change
   freely.
5. **Budget for the tightest limit.** The 24-character storage account cap sets the
   12-character workload name budget, globally.

## Standard

### The pattern

All resources follow this pattern unless listed under [Exceptions](#exceptions):

```text
<resource-type>-<workload>-<environment>-<region>-<instance>
```

Example: `ca-code-review-dev-cc-01`

| Segment | Description | Examples |
|---------|-------------|----------|
| `resource-type` | CAF abbreviation for the Azure resource type (table below). Always first, so resources group by type when sorted. | `rg`, `kv`, `ca` |
| `workload` | Short, stable workload name — lowercase kebab-case, ≤ 12 characters. May include a component suffix for multi-component workloads (`billing-api`, `billing-web`). | `code-review`, `billing` |
| `environment` | One of `dev`, `stg`, `prod`, `shared` — see [Azure Environments](../azure/environments.md). `shared` is reserved for platform resources serving all environments. | `dev`, `prod` |
| `region` | Short region code (table below). | `cc`, `ce` |
| `instance` | Two-digit, zero-padded ordinal starting at `01`. Only meaningful when multiple instances exist in the same scope — but always included, so names never need restructuring when a second instance appears. | `01`, `02` |

**Why this order?** Resource type first groups resources by kind in sorted lists — the most
common way engineers scan the portal and CLI output. Workload next answers "whose is this?".
Environment and region narrow the deployment context. Instance disambiguates. Each segment
narrows scope left to right, which makes prefix filtering natural at every level.

### Resource type prefixes

Where the Cloud Adoption Framework defines an abbreviation, we use it — do not invent
alternatives.

| Prefix | Resource type | | Prefix | Resource type |
|--------|---------------|-|--------|---------------|
| `rg` | Resource group | | `appi` | Application Insights |
| `app` | App Service (web app) | | `log` | Log Analytics workspace |
| `func` | Function App | | `vnet` | Virtual network |
| `asp` | App Service plan | | `snet` | Subnet |
| `ca` | Container App | | `pip` | Public IP address |
| `cae` | Container Apps environment | | `nsg` | Network security group |
| `aks` | Azure Kubernetes Service | | `pep` | Private endpoint |
| `acr` | Container registry | | `agw` | Application Gateway |
| `kv` | Key Vault | | `afd` | Azure Front Door |
| `st` | Storage account | | `apim` | API Management |
| `sql` | SQL Server (logical) | | `lb` | Load balancer |
| `sqldb` | SQL database | | `vm` | Virtual machine |
| `cosmos` | Cosmos DB account | | `vmss` | VM scale set |
| `redis` | Azure Cache for Redis | | `nic` | Network interface |
| `sb` | Service Bus namespace | | `dns` | DNS zone (public) |
| `evhns` | Event Hubs namespace | | `pdns` | Private DNS zone |
| `mi` | User-assigned managed identity | | | |

If a resource type is not listed, use the abbreviation from the
[CAF abbreviation list](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/resource-abbreviations)
and add it here via pull request. Never use two different abbreviations for the same type.

### Environment codes

Use `dev`, `stg`, `prod`, and `shared` — exactly these, defined fully in
[Azure Environments](../azure/environments.md). Not `test`, `uat`, `staging`, `production`, or
`prd`. Consistency is the entire value: automation keys off these strings (Terraform validation,
GitHub environments, Azure Policy assignments, alert routing), and a single variant spelling
breaks filters, budgets, and policies silently.

`shared` appears only in the names of **platform resources that serve every environment** —
Terraform state storage, the shared container registry. A workload resource is never `shared`.

### Region codes

Azure's own region names are too long for resource names (`canadacentral` is 13 characters —
fatal for storage accounts). Use these short codes:

| Code | Region | | Code | Region |
|------|--------|-|------|--------|
| `cc` | Canada Central | | `wus2` | West US 2 |
| `ce` | Canada East | | `neu` | North Europe |
| `eus` | East US | | `weu` | West Europe |
| `eus2` | East US 2 | | `uks` | UK South |
| `cus` | Central US | | `sea` | Southeast Asia |
| `wus` | West US | | | |

**Avlon defaults are Canada Central (`cc`) with Canada East (`ce`) as the paired
disaster-recovery region.** Add new codes here via pull request before first use — codes are
unique and never reused. Include the region code even for single-region deployments: the day a
second region is added, nothing needs to be renamed.

### Exceptions

A minority of Azure resource types impose their own rules. The convention still applies — the
same components in the same order — but the format adapts.

| Resource | Restrictions | Format | Example |
|----------|--------------|--------|---------|
| **Storage account** | Globally unique; 3–24 chars; lowercase alphanumeric only — **no hyphens** | Strip hyphens: `st<workload><env><region><instance>` | `stbillingprodcc01` |
| **Container registry** | Globally unique; 5–50 chars; alphanumeric only — **no hyphens** | Strip hyphens: `acr<workload><env><region><instance>` | `acrplatformsharedcc01` |
| **Key Vault** | Globally unique; 3–24 chars; alphanumerics and hyphens; starts with a letter | Standard pattern; watch the 24-char limit | `kv-billing-prod-cc-01` |
| **SQL Server / App Service / Function App / Front Door / APIM** | Globally unique — each becomes a DNS label (`*.database.windows.net`, `*.azurewebsites.net`) | Standard pattern; uniqueness usually falls out of workload+env+region+instance | `sql-billing-prod-cc-01` |
| **DNS zone** | Must be a valid domain name — the convention does not apply | Use the actual domain | `avlon.ca` |
| **Private DNS zone** | Must match the Azure private-link zone name exactly | Use the required zone name | `privatelink.vaultcore.azure.net` |
| **Managed identity** | Few restrictions, but the name surfaces in RBAC assignments and audit logs | Standard pattern; name it for the workload that uses it, not the resource it accesses | `mi-billing-prod-cc-01` |
| **Subnet / NSG / child resources** | Unique only within their parent | Standard pattern with a component qualifier where useful | `snet-billing-data-prod-cc-01` |

**Length budgeting.** The tightest common constraints are the storage account and Key Vault at
24 characters. `st<workload><env><region><instance>` must fit in 24, which is why **workload
names are 12 characters or fewer** (hyphens included). If a name still won't fit, shorten the
workload name consistently everywhere — never truncate ad hoc in one place.

**Global uniqueness collisions.** When a globally unique name is taken (rare, given the
workload+env+region+instance combination), increment the instance number. Do not append random
characters manually — if entropy is genuinely required, generate it deterministically in
Terraform so every plan produces the same name.

### Generating names in Terraform

Names are **generated, never hardcoded**. Hardcoded names drift between environments, defeat
module reuse, and hide the convention from review. Derive every name from the same small set of
variables:

```hcl
variable "workload" {
  description = "Short workload name (lowercase kebab-case, max 12 chars)."
  type        = string

  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{0,11}$", var.workload))
    error_message = "workload must be lowercase kebab-case, start with a letter, max 12 chars."
  }
}

variable "environment" {
  description = "Deployment environment."
  type        = string

  validation {
    condition     = contains(["dev", "stg", "prod"], var.environment)
    error_message = "environment must be one of: dev, stg, prod."
  }
}

variable "location" {
  description = "Azure region (full name, e.g. canadacentral)."
  type        = string
  default     = "canadacentral"
}

variable "instance" {
  description = "Two-digit instance ordinal."
  type        = string
  default     = "01"
}
```

Platform stacks (and only platform stacks) widen the environment validation to include
`"shared"`. Centralize the derivation in locals so the convention lives in exactly one place per
stack:

```hcl
locals {
  # Region short codes — keep in sync with naming/azure.md
  region_codes = {
    canadacentral = "cc"
    canadaeast    = "ce"
    eastus        = "eus"
    westus        = "wus"
  }

  region = local.region_codes[var.location]

  # Standard suffix shared by all resources in this stack
  suffix = "${var.workload}-${var.environment}-${local.region}-${var.instance}"

  # Hyphen-less variant for storage accounts, container registries, etc.
  suffix_compact = replace(local.suffix, "-", "")
}

resource "azurerm_resource_group" "main" {
  name     = "rg-${local.suffix}"
  location = var.location
}

resource "azurerm_key_vault" "main" {
  name                = "kv-${local.suffix}"
  resource_group_name = azurerm_resource_group.main.name
  location            = var.location
  # ...
}

resource "azurerm_storage_account" "main" {
  name                = "st${local.suffix_compact}"
  resource_group_name = azurerm_resource_group.main.name
  location            = var.location
  # ...
}
```

With this in place, promoting a stack from `dev` to `prod` or deploying to a second region is a
variable change — every name updates correctly and identically, and `terraform plan` output is
readable because names carry their context. For larger estates, factor the locals into a shared
naming module so the region map and pattern are defined once per organization.

## Examples

**Workload: `code-review` — Container Apps, dev, Canada Central**

```text
rg-code-review-dev-cc-01           # Resource group
cae-code-review-dev-cc-01          # Container Apps environment
ca-code-review-dev-cc-01           # Container App
kv-code-review-dev-cc-01           # Key Vault
sqldb-code-review-dev-cc-01        # SQL database
appi-code-review-dev-cc-01         # Application Insights
mi-code-review-dev-cc-01           # Managed identity for the app
stcodereviewdevcc01                # Storage account (hyphens stripped — see exceptions)
```

**Workload: `billing` — App Service + SQL, prod, Canada Central with Canada East DR**

```text
rg-billing-prod-cc-01              # Primary resource group
rg-billing-prod-ce-01              # DR resource group
app-billing-api-prod-cc-01         # App Service — API component
app-billing-web-prod-cc-01         # App Service — web component
asp-billing-prod-cc-01             # App Service plan
sql-billing-prod-cc-01             # SQL server (primary)
sql-billing-prod-ce-01             # SQL server (failover replica)
kv-billing-prod-cc-01              # Key Vault
mi-billing-prod-cc-01              # Managed identity
mi-github-billing-prod-cc-01       # GitHub OIDC deployment identity
vnet-billing-prod-cc-01            # Spoke virtual network
snet-billing-app-prod-cc-01        # Subnet — app tier
snet-billing-data-prod-cc-01       # Subnet — data tier
nsg-billing-app-prod-cc-01         # NSG for the app subnet
pep-billing-sql-prod-cc-01         # Private endpoint to SQL
appi-billing-prod-cc-01            # Application Insights
stbillingprodcc01                  # Storage account
```

**Platform capabilities — the `shared` and platform-`prod` resources**

```text
rg-platform-tfstate-shared-cc-01   # Terraform state resource group
sttfstatesharedcc01                # Terraform state storage account
rg-platform-acr-shared-cc-01       # Container registry resource group
acrplatformsharedcc01              # The org's ONE shared container registry
rg-platform-network-prod-cc-01     # Hub networking resource group
vnet-platform-prod-cc-01           # Hub virtual network
rg-platform-monitor-prod-cc-01     # Central monitoring resource group
log-platform-monitor-prod-cc-01    # Log Analytics workspace
```

Note how `stbillingprodcc01` and `acrplatformsharedcc01` follow the **same components in the
same order** with hyphens removed: `st` + `billing` + `prod` + `cc` + `01`. The convention
survives the restriction; only the separator is dropped.

## Anti-patterns

| Anti-pattern | Why it fails |
|--------------|--------------|
| `BillingProdKV`, `billing_kv` | Mixed case and underscores — several resource types reject them; all names are lowercase kebab-case. |
| `sql-billing-uat-cc-01` | `uat` is not a valid environment; staging validation happens in `stg`. |
| `rg-avlon-billing-prod` | Company name wastes characters and goes stale on transfer; also missing region and instance. |
| `st-billing-prod` | Hyphens are illegal in storage accounts, and the missing segments make the name unpredictable. |
| `kv-billing-prod-cc-dr` | Meaning smuggled into the instance slot. DR is expressed by the region code: `kv-billing-prod-ce-01`. |
| `sql-mike-test`, `vm-billing-20260711` | Personal names and dates — see [General Naming Principles](general.md). |
| Hardcoded name strings in Terraform | Drift between environments; the convention disappears from review. Generate from locals. |

## Tradeoffs

- **Names are long.** `snet-billing-data-prod-cc-01` is verbose next to `data-subnet`. We pay
  the characters to buy context: every name is unambiguous in any listing, alert, or log line
  without opening the resource.
- **The instance segment is usually dead weight.** Most resources are forever `01`. It stays,
  because adding it later means destroying and recreating resources, while carrying it costs
  three characters.
- **A fixed prefix and region table needs maintenance.** New resource types and regions require
  a pull request before first use. That gate is the feature — it is how two teams avoid
  inventing two abbreviations for the same thing.
- **The convention is designed to survive change, and that shapes it.** Names carry no
  subscription (resources move during reorganizations without renaming), region is present from
  day one (a second region is additive: `rg-billing-prod-weu-01`), DR names are mechanically
  derivable (`sql-billing-prod-ce-01`), and instance numbers absorb horizontal growth. What the
  name deliberately *excludes* — owner, subscription, dates — lives in
  [tags](../azure/tagging.md), which can change freely.

## Related Standards

- [General Naming Principles](general.md) — vocabulary and cross-cutting rules
- [Azure Environments](../azure/environments.md) — the full `dev`/`stg`/`prod`/`shared` definition
- [Azure Resource Groups](../azure/resource-groups.md) — how workloads map to RGs
- [Azure Tagging](../azure/tagging.md) — the metadata that stays out of names
- [Terraform Naming](terraform.md) and [Remote State](../terraform/remote-state.md) — name generation and state keys
