# Azure Resource Naming

**Status:** Active
**Applies to:** All Azure resources across all subscriptions and environments. Based on the [Azure Cloud Adoption Framework naming guidance](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming), adapted and made opinionated for Avlon Technologies.

---

## Purpose

A resource name is the first — and often only — piece of metadata a human sees. Names appear in
the portal, in cost reports, in alerts, in CLI output, and in incident channels at 2 a.m. The
difference between `sql-billing-prod-cc` and `sql-billing-dev-cc` must be unmistakable,
because ambiguous names cause production incidents.

Naming is cheap to get right at creation time and expensive to fix later — most Azure resources
**cannot be renamed** and must be destroyed and recreated. This standard is therefore mandatory,
not advisory. It exists so that anyone can look at a name and know what a resource is, which
workload it belongs to, which environment it serves, and where it runs — and so Terraform,
scripts, and Azure Policy can construct and predict names from known inputs with no lookups and
no drift between environments.

## Guiding Principles

1. **Deterministic.** Given workload, environment, and region, everyone — human or
   pipeline — produces the same name. See [General Naming Principles](general.md).
2. **Self-describing left to right.** Each segment narrows scope: type, then workload, then
   environment, then region. Prefix filtering in `az resource list`, cost
   queries, and Azure Policy falls out naturally.
3. **One convention that survives its exceptions.** Where Azure forbids hyphens or shortens
   limits, the same components appear in the same order — only the format adapts.
4. **Nothing volatile in the name.** Owner, subscription, dates, and meaning-laden instance
   numbers are kept out of names and pushed into [tags](../azure/tagging.md), which can change
   freely.
5. **Budget for the tightest limit.** The 24-character storage account cap sets the
   12-character workload name budget, globally.
6. **No dead segments.** A segment that would carry the same value on every resource
   (an `-01` on a fleet of singletons) is noise, not information. The instance ordinal
   exists, but it is appended only when it disambiguates something real.

## Standard

### The pattern

All resources follow this pattern unless listed under [Exceptions](#exceptions):

```text
<resource-type>-<workload>-<environment>-<region>[-<instance>]
```

Example: `ca-code-review-dev-cc`

| Segment | Description | Examples |
|---------|-------------|----------|
| `resource-type` | CAF abbreviation for the Azure resource type (table below). Always first, so resources group by type when sorted. | `rg`, `kv`, `ca` |
| `workload` | Short, stable workload name — lowercase kebab-case, ≤ 12 characters. May include a component suffix for multi-component workloads (`billing-api`, `billing-web`). | `code-review`, `billing` |
| `environment` | One of `dev`, `stg`, `prod`, `shared` — see [Azure Environments](../azure/environments.md). `shared` is reserved for platform resources serving all environments. | `dev`, `prod` |
| `region` | Short region code (table below). | `cc`, `ce` |
| `instance` | **Optional.** Two-digit, zero-padded ordinal, appended only when it disambiguates something real — see [The instance segment](#the-instance-segment). | `02`, `03` |

**Why this order?** Resource type first groups resources by kind in sorted lists — the most
common way engineers scan the portal and CLI output. Workload next answers "whose is this?".
Environment and region narrow the deployment context. Each segment narrows scope left to
right, which makes prefix filtering natural at every level.

### The instance segment

Most resources are singletons within their scope, and a number that is always `01` carries no
information — so **the default is no instance segment at all**: `rg-billing-prod-cc`,
`sql-billing-prod-cc`, `stbillingprodcc`. Append an ordinal only in these cases:

- **Planned multiples.** When a scope is designed from the start to hold several instances of
  the same type (a pool of VMs, sharded servers), number *all* of them from `01`:
  `vm-billing-prod-cc-01` … `vm-billing-prod-cc-04`. Known multiples deserve symmetric names.
- **An unplanned second instance.** The existing resource keeps its unnumbered name (Azure
  resources cannot be renamed — only destroyed and recreated, which is rarely worth it) and the
  newcomer takes `-02`, reading naturally as "the second": `sql-billing-prod-cc` and
  `sql-billing-prod-cc-02`. The asymmetry is accepted deliberately; see Tradeoffs.
- **A global-uniqueness collision.** When a globally unique name (storage account, ACR, Key
  Vault, anything DNS-labelled) is already taken by another tenant, append the lowest free
  ordinal starting at `01`: `stbillingprodcc` taken → `stbillingprodcc01`. Do not append random
  characters — if entropy is genuinely required, generate it deterministically in Terraform so
  every plan produces the same name.

In Terraform, the `instance` input therefore defaults to *empty/null* and is set per resource
only when one of the cases above applies.

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
| **Storage account** | Globally unique; 3–24 chars; lowercase alphanumeric only — **no hyphens** | Strip hyphens: `st<workload><env><region>` | `stbillingprodcc` |
| **Container registry** | Globally unique; 5–50 chars; alphanumeric only — **no hyphens** | Strip hyphens: `acr<workload><env><region>` | `acrplatformsharedcc` |
| **Key Vault** | Globally unique; 3–24 chars; alphanumerics and hyphens; starts with a letter | Standard pattern; watch the 24-char limit | `kv-billing-prod-cc` |
| **SQL Server / App Service / Function App / Front Door / APIM** | Globally unique — each becomes a DNS label (`*.database.windows.net`, `*.azurewebsites.net`) | Standard pattern; uniqueness usually falls out of workload+env+region | `sql-billing-prod-cc` |
| **DNS zone** | Must be a valid domain name — the convention does not apply | Use the actual domain | `avlon.ca` |
| **Private DNS zone** | Must match the Azure private-link zone name exactly | Use the required zone name | `privatelink.vaultcore.azure.net` |
| **Managed identity** | Few restrictions, but the name surfaces in RBAC assignments and audit logs | Standard pattern; name it for the workload that uses it, not the resource it accesses | `mi-billing-prod-cc` |
| **Subnet / NSG / child resources** | Unique only within their parent | Standard pattern with a component qualifier where useful | `snet-billing-data-prod-cc` |

**Length budgeting.** The tightest common constraints are the storage account and Key Vault at
24 characters. `st<workload><env><region>` must fit in 24 — 2 for the prefix, up to 6 for the
environment (`shared`), up to 4 for the region — which is why **workload names are 12
characters or fewer** (hyphens included). Leave one character of slack where you can: a
collision ordinal (below) costs two more. If a name still won't fit, shorten the workload name
consistently everywhere — never truncate ad hoc in one place.

**Global uniqueness collisions.** When a globally unique name is taken by another tenant (rare,
given the workload+env+region combination), append the lowest free instance ordinal starting at
`01` — `stbillingprodcc` taken → `stbillingprodcc01`. Do not append random characters manually —
if entropy is genuinely required, generate it deterministically in Terraform so every plan
produces the same name.

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
  description = "Optional two-digit instance ordinal. Null (the default) omits the segment; set it only for planned multiples, an unplanned second instance, or a global-name collision."
  type        = string
  default     = null

  validation {
    condition     = var.instance == null || can(regex("^(0[1-9]|[1-9][0-9])$", var.instance))
    error_message = "instance must be null or a two-digit, zero-padded ordinal (01-99)."
  }
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

  # Standard suffix shared by all resources in this stack. The instance
  # segment appears only when var.instance is set (see The instance segment).
  suffix = join("-", compact([var.workload, var.environment, local.region, var.instance]))

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
rg-code-review-dev-cc           # Resource group
cae-code-review-dev-cc          # Container Apps environment
ca-code-review-dev-cc           # Container App
kv-code-review-dev-cc           # Key Vault
sqldb-code-review-dev-cc        # SQL database
appi-code-review-dev-cc         # Application Insights
mi-code-review-dev-cc           # Managed identity for the app
stcodereviewdevcc               # Storage account (hyphens stripped — see exceptions)
```

**Workload: `billing` — App Service + SQL, prod, Canada Central with Canada East DR**

```text
rg-billing-prod-cc              # Primary resource group
rg-billing-prod-ce              # DR resource group
app-billing-api-prod-cc         # App Service — API component
app-billing-web-prod-cc         # App Service — web component
asp-billing-prod-cc             # App Service plan
sql-billing-prod-cc             # SQL server (primary)
sql-billing-prod-ce             # SQL server (failover replica)
kv-billing-prod-cc              # Key Vault
mi-billing-prod-cc              # Managed identity
mi-github-billing-prod-cc       # GitHub OIDC deployment identity
vnet-billing-prod-cc            # Spoke virtual network
snet-billing-app-prod-cc        # Subnet — app tier
snet-billing-data-prod-cc       # Subnet — data tier
nsg-billing-app-prod-cc         # NSG for the app subnet
pep-billing-sql-prod-cc         # Private endpoint to SQL
appi-billing-prod-cc            # Application Insights
stbillingprodcc                 # Storage account
```

**Platform capabilities — the `shared` and platform-`prod` resources**

```text
rg-platform-tfstate-shared-cc   # Terraform state resource group
sttfstatesharedcc               # Terraform state storage account
rg-platform-acr-shared-cc       # Container registry resource group
acrplatformsharedcc             # The org's ONE shared container registry
rg-platform-network-prod-cc     # Hub networking resource group
vnet-platform-prod-cc           # Hub virtual network
rg-platform-monitor-prod-cc     # Central monitoring resource group
log-platform-monitor-prod-cc    # Log Analytics workspace
```

Note how `stbillingprodcc` and `acrplatformsharedcc` follow the **same components in the
same order** with hyphens removed: `st` + `billing` + `prod` + `cc`. The convention
survives the restriction; only the separator is dropped.

**With instances, where they earn their place:**

```text
vm-render-prod-cc-01            # a planned pool — all members numbered from 01
vm-render-prod-cc-02
sql-billing-prod-cc             # the original singleton…
sql-billing-prod-cc-02          # …and the unplanned second instance added later
stbillingprodcc01               # global name collision: stbillingprodcc was taken
```

## Anti-patterns

| Anti-pattern | Why it fails |
|--------------|--------------|
| `BillingProdKV`, `billing_kv` | Mixed case and underscores — several resource types reject them; all names are lowercase kebab-case. |
| `sql-billing-uat-cc` | `uat` is not a valid environment; staging validation happens in `stg`. |
| `rg-avlon-billing-prod` | Company name wastes characters and goes stale on transfer; also missing the region. |
| `st-billing-prod` | Hyphens are illegal in storage accounts, and the missing segments make the name unpredictable. |
| `rg-billing-prod-cc-01` for a singleton | A dead segment: `-01` on a resource with no siblings says nothing and costs three characters on every name in the estate. Instance ordinals are appended only when they disambiguate — see [The instance segment](#the-instance-segment). |
| `kv-billing-prod-cc-dr` | Meaning smuggled into the instance slot. DR is expressed by the region code: `kv-billing-prod-ce`. |
| `sql-mike-test`, `vm-billing-20260711` | Personal names and dates — see [General Naming Principles](general.md). |
| Hardcoded name strings in Terraform | Drift between environments; the convention disappears from review. Generate from locals. |

## Tradeoffs

- **Names are long.** `snet-billing-data-prod-cc` is verbose next to `data-subnet`. We pay
  the characters to buy context: every name is unambiguous in any listing, alert, or log line
  without opening the resource.
- **Omitting the instance by default trades symmetry for cleanliness.** Because Azure resources
  cannot be renamed, an unplanned second instance produces an asymmetric pair
  (`sql-billing-prod-cc` + `sql-billing-prod-cc-02`) unless the original is destroyed and
  recreated — which is rarely worth it. We accept the occasional asymmetric pair to keep the
  overwhelmingly common case (singletons, i.e. almost everything) free of a segment that says
  nothing. Where multiples are *planned*, numbering everything from `01` up front restores
  symmetry — anticipating that is the engineer's judgment call, and misjudging it costs an
  asymmetric name, not an outage.
- **A fixed prefix and region table needs maintenance.** New resource types and regions require
  a pull request before first use. That gate is the feature — it is how two teams avoid
  inventing two abbreviations for the same thing.
- **The convention is designed to survive change, and that shapes it.** Names carry no
  subscription (resources move during reorganizations without renaming), region is present from
  day one (a second region is additive: `rg-billing-prod-weu`), DR names are mechanically
  derivable (`sql-billing-prod-ce`), and instance ordinals absorb horizontal growth when it
  arrives. What the
  name deliberately *excludes* — owner, subscription, dates — lives in
  [tags](../azure/tagging.md), which can change freely.

## Related Standards

- [General Naming Principles](general.md) — vocabulary and cross-cutting rules
- [Azure Environments](../azure/environments.md) — the full `dev`/`stg`/`prod`/`shared` definition
- [Azure Resource Groups](../azure/resource-groups.md) — how workloads map to RGs
- [Azure Tagging](../azure/tagging.md) — the metadata that stays out of names
- [Terraform Naming](terraform.md) and [Remote State](../terraform/remote-state.md) — name generation and state keys
