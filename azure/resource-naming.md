# Azure Resource Naming Standard

**Status:** Active
**Applies to:** All Azure resources across all subscriptions, projects, and client engagements.
**Based on:** [Azure Cloud Adoption Framework — naming and tagging](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming), adapted and made opinionated for Avlon Technologies.

---

## Goals

A resource name is the first — and often only — piece of metadata a human sees. Names appear in the portal, in cost reports, in alerts, in CLI output, and in incident channels at 2 a.m. A good naming convention means:

- **Instant identification.** Anyone can look at a resource name and know what it is, which application it belongs to, which environment it serves, and where it runs — without opening the resource.
- **Deterministic automation.** Terraform, scripts, and policies can construct and predict names from known inputs. No lookups, no guesswork, no drift between environments.
- **Safe operations.** The difference between `sql-billing-prod-cc-01` and `sql-billing-dev-cc-01` is unmistakable. Ambiguous names cause production incidents.
- **Clean cost management and governance.** Names sort and filter predictably in cost analysis, Azure Policy, and resource graphs.

Naming is cheap to get right at creation time and expensive to fix later — most Azure resources **cannot be renamed** and must be destroyed and recreated. This standard is therefore mandatory, not advisory.

## General Naming Pattern

All resources follow this pattern unless listed under [Azure Resource Exceptions](#azure-resource-exceptions):

```
<resource-type>-<application>-<environment>-<region>-<instance>
```

Example: `ca-code-review-dev-cc-01`

| Component | Description | Examples |
|-----------|-------------|----------|
| `resource-type` | Abbreviation for the Azure resource type (see [Resource Type Prefixes](#resource-type-prefixes)). Always first, so resources group by type when sorted. | `rg`, `kv`, `ca` |
| `application` | Short, stable name of the application or workload. Lowercase, hyphen-separated. May include a component suffix for multi-component apps (e.g. `billing-api`, `billing-web`). | `code-review`, `billing` |
| `environment` | Deployment environment (see [Environment Names](#environment-names)). | `dev`, `prod` |
| `region` | Short code for the Azure region (see [Region Codes](#region-codes)). | `cc`, `eus` |
| `instance` | Two-digit, zero-padded ordinal. Start at `01`. Only meaningful when multiple instances of the same resource exist in the same scope — but always included, so names never need restructuring when a second instance appears. | `01`, `02` |

### Why this order?

Resource type first groups resources by kind in sorted lists — the most common way engineers scan the portal and CLI output. Application next answers "whose is this?". Environment and region narrow the deployment context. Instance disambiguates. Each component narrows scope left to right, which makes prefix-filtering (`az resource list`, cost queries, Azure Policy) natural.

## Resource Type Prefixes

Use these abbreviations. Where the Cloud Adoption Framework defines an abbreviation, we use it — do not invent alternatives.

| Prefix | Resource Type |
|--------|---------------|
| `rg` | Resource Group |
| `app` | App Service (Web App) |
| `func` | Function App |
| `asp` | App Service Plan |
| `ca` | Container App |
| `cae` | Container Apps Environment |
| `aks` | Azure Kubernetes Service |
| `acr` | Azure Container Registry |
| `kv` | Key Vault |
| `st` | Storage Account |
| `sql` | SQL Server (logical server) |
| `sqldb` | SQL Database |
| `cosmos` | Cosmos DB Account |
| `redis` | Azure Cache for Redis |
| `sb` | Service Bus Namespace |
| `evhns` | Event Hubs Namespace |
| `mi` | User-Assigned Managed Identity |
| `appi` | Application Insights |
| `log` | Log Analytics Workspace |
| `vnet` | Virtual Network |
| `snet` | Subnet |
| `pip` | Public IP Address |
| `nsg` | Network Security Group |
| `pep` | Private Endpoint |
| `agw` | Application Gateway |
| `afd` | Azure Front Door |
| `apim` | API Management |
| `lb` | Load Balancer |
| `vm` | Virtual Machine |
| `vmss` | Virtual Machine Scale Set |
| `nic` | Network Interface |
| `dns` | DNS Zone (public) |
| `pdns` | Private DNS Zone |

If a resource type is not listed, use the abbreviation from the [CAF abbreviation list](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/resource-abbreviations) and add it here via pull request. Never use two different abbreviations for the same resource type.

## Environment Names

| Environment | Code | Purpose |
|-------------|------|---------|
| Development | `dev` | Day-to-day development and integration. May be unstable. |
| Test | `test` | Automated and manual testing. Rebuilt frequently. |
| User Acceptance | `uat` | Client/stakeholder validation. Production-like configuration. |
| Production | `prod` | Live workloads. |

**Use these codes and no others.** Not `develop`, `dv`, `stg`, `production`, or `prd`. Consistency is the entire value: automation keys off these strings (Terraform workspaces, pipeline stages, Azure Policy assignments, alert routing), and a single variant spelling breaks filters, budgets, and policies silently. If a project needs an additional environment (e.g. a staging slot or a sandbox), propose the code here first — do not improvise one per project.

## Region Codes

Azure's own region names are too long for resource names (`canadacentral` is 13 characters — fatal for storage accounts). Use these short codes:

| Code | Azure Region |
|------|--------------|
| `cc` | Canada Central |
| `ce` | Canada East |
| `eus` | East US |
| `eus2` | East US 2 |
| `cus` | Central US |
| `wus` | West US |
| `wus2` | West US 2 |
| `neu` | North Europe |
| `weu` | West Europe |
| `uks` | UK South |
| `sea` | Southeast Asia |

**Default regions for Avlon projects are Canada Central (`cc`) with Canada East (`ce`) as the paired disaster-recovery region.** Add new codes here (via pull request) before first use — codes must be unique and never reused for a different region.

Include the region code even for single-region deployments. The day a second region is added — for DR, latency, or data residency — nothing needs to be renamed.

## Naming Examples

### Application: Code Review Platform (dev, Canada Central)

```
rg-code-review-dev-cc-01           # Resource group
cae-code-review-dev-cc-01          # Container Apps environment
ca-code-review-dev-cc-01           # Container App — the code review app
kv-code-review-dev-cc-01           # Key Vault
sql-code-review-dev-cc-01          # SQL logical server
sqldb-code-review-dev-cc-01        # SQL database
appi-code-review-dev-cc-01         # Application Insights
log-code-review-dev-cc-01          # Log Analytics workspace
mi-code-review-dev-cc-01           # Managed identity for the app
acrcodereviewdevcc01               # Container registry (no hyphens — see exceptions)
stcodereviewdevcc01                # Storage account (no hyphens — see exceptions)
```

### Application: Billing (prod, Canada Central + Canada East DR)

```
rg-billing-prod-cc-01              # Primary resource group
rg-billing-prod-ce-01              # DR resource group
app-billing-api-prod-cc-01         # App Service — API
asp-billing-prod-cc-01             # App Service plan
sql-billing-prod-cc-01             # SQL server (primary)
sql-billing-prod-ce-01             # SQL server (failover replica)
kv-billing-prod-cc-01              # Key Vault
vnet-billing-prod-cc-01            # Virtual network
snet-billing-app-prod-cc-01        # Subnet — app tier
snet-billing-data-prod-cc-01       # Subnet — data tier
nsg-billing-app-prod-cc-01         # NSG for the app subnet
pep-billing-sql-prod-cc-01         # Private endpoint to SQL
stbillingprodcc01                  # Storage account
```

Note how the storage account and container registry follow the **same components in the same order**, with hyphens removed: `st` + `billing` + `prod` + `cc` + `01` → `stbillingprodcc01`. The convention survives the restriction; only the separator is dropped.

## Azure Resource Exceptions

A minority of Azure resource types impose their own naming rules. The convention still applies — the same components in the same order — but the format adapts.

| Resource | Restrictions | Format | Example |
|----------|--------------|--------|---------|
| **Storage Account** | Globally unique; 3–24 chars; lowercase letters and digits only — **no hyphens** | Strip hyphens: `st<app><env><region><instance>` | `stcodereviewdevcc01` |
| **Container Registry** | Globally unique; 5–50 chars; alphanumeric only — **no hyphens** | Strip hyphens: `acr<app><env><region><instance>` | `acrcodereviewdevcc01` |
| **Key Vault** | Globally unique; 3–24 chars; alphanumerics and hyphens; must start with a letter | Standard pattern, but watch the 24-char limit — shorten the application name if necessary | `kv-code-review-dev-cc-01` (24 chars — at the limit) |
| **SQL Server / App Service / Function App / Front Door / APIM** | Globally unique (each becomes a DNS label, e.g. `*.database.windows.net`, `*.azurewebsites.net`) | Standard pattern; global uniqueness usually falls out of the app+env+region+instance combination | `sql-billing-prod-cc-01` |
| **DNS Zone** | Must be a valid domain name — the convention does not apply | Use the actual domain | `avlon.ca`, `api.avlon.ca` |
| **Private DNS Zone** | Must match the Azure private-link zone name exactly | Use the required zone name | `privatelink.vaultcore.azure.net` |
| **Managed Identity** | Few restrictions, but the name surfaces in RBAC assignments and audit logs | Standard pattern; name it for the workload that uses it, not the resource it accesses | `mi-code-review-dev-cc-01` |
| **Subnet / NSG / other child resources** | Unique only within their parent — no global constraints | Standard pattern with a component qualifier where useful | `snet-billing-data-prod-cc-01` |

**Length budgeting:** the tightest common constraint is the storage account (24 chars) and Key Vault (24 chars). Keep application names short enough that `st<app><env><region><instance>` fits in 24 characters — in practice, **application names should be 12 characters or fewer** (hyphens included). If a name still won't fit, shorten the application name consistently everywhere; never truncate ad hoc in one place.

**Global uniqueness collisions:** when a globally unique name is taken (rare, given the app+env+region+instance combination), increment the instance number. Do not append random characters manually — if entropy is genuinely required, generate it deterministically in Terraform so every plan produces the same name.

## Naming Guidelines

- **Use lowercase everywhere.** Some resource types require it, and mixed case creates two ways to write the same name. One rule, zero exceptions, nothing to remember.
- **No spaces, no underscores.** Hyphens are the only separator (dropped where prohibited).
- **No company names.** `avlon` in a resource name adds no information inside Avlon's own tenant, wastes scarce characters, and becomes wrong the moment a workload is transferred to a client or acquired entity. Ownership belongs in tags, not names.
- **Never include engineer names.** Resources outlive their creators. `sql-mike-test` is unownable the day Mike changes projects.
- **Never encode subscription names.** Subscriptions are a governance boundary that changes during reorganizations, migrations, and acquisitions. The subscription is always visible in context; encoding it in the name just guarantees the name goes stale.
- **Keep names deterministic.** Given the same application, environment, region, and instance, everyone — human or pipeline — must produce the same name. If a name can't be predicted from those four inputs, it's wrong.
- **Prefer readability over aggressive abbreviation.** `code-review` beats `cr`. Abbreviate only when a length limit forces it, and then abbreviate the same way everywhere. The reader at 2 a.m. matters more than the character count.
- **Do not include dates or versions.** Names describe what a resource *is*, not when it was created — creation time lives in resource metadata. Dated names rot instantly and encourage clone-and-forget sprawl.
- **Use instance numbers as ordinals, not meaning.** `01`, `02` disambiguate identical resources; they must never encode meaning ("02 is the DR one"). Meaning goes in the region or component segment.
- **Environment and region are never optional.** Even for a "temporary" single-region dev resource — temporary resources have a way of becoming permanent.

## Terraform

Names must be **generated, never hardcoded**. Hardcoded names drift between environments, defeat module reuse, and hide the convention from review. Derive every name from the same small set of variables:

```hcl
variable "project" {
  description = "Short application/workload name (lowercase, hyphen-separated, max 12 chars)."
  type        = string

  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{1,11}$", var.project))
    error_message = "project must be lowercase alphanumeric/hyphens, start with a letter, max 12 chars."
  }
}

variable "environment" {
  description = "Deployment environment."
  type        = string

  validation {
    condition     = contains(["dev", "test", "uat", "prod"], var.environment)
    error_message = "environment must be one of: dev, test, uat, prod."
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

Centralize the derivation in locals so the convention lives in exactly one place per module:

```hcl
locals {
  # Region short codes — keep in sync with azure/resource-naming.md
  region_codes = {
    canadacentral = "cc"
    canadaeast    = "ce"
    eastus        = "eus"
    westus        = "wus"
  }

  region = local.region_codes[var.location]

  # Standard suffix shared by all resources in this stack
  suffix = "${var.project}-${var.environment}-${local.region}-${var.instance}"

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

With this in place, promoting a stack from `dev` to `prod` or deploying to a second region is a variable change — every name updates correctly and identically, and `terraform plan` output is readable because names carry their context.

For larger estates, factor the locals into a shared naming module so the region map and pattern are defined once per organization rather than once per stack.

## Future Proofing

This convention is designed to survive the changes that break naming schemes:

- **Multiple subscriptions.** Names carry no subscription information, so resources can move between subscriptions (reorganizations, landing-zone migrations, billing changes) without renaming — which matters, because renaming usually means recreating.
- **Multiple regions.** The region code is present from day one. Adding a second region for latency or data residency produces `rg-billing-prod-weu-01` alongside `rg-billing-prod-cc-01` — same convention, zero restructuring.
- **Disaster recovery.** DR resources are just the same names in the paired region (`sql-billing-prod-ce-01`). Failover runbooks and automation can derive DR resource names mechanically from primary names.
- **Acquisitions and client handovers.** Because names contain no company, subscription, or engineer identifiers, a workload transfers to a client tenant or an acquired estate without a single rename. The names describe the workload, not the owner.
- **Scale.** Instance numbers absorb horizontal growth (`ca-code-review-prod-cc-02`), and the component segment of the application name absorbs architectural growth (`ca-code-review-worker-prod-cc-01`) — the pattern never needs to be reinvented, only extended.

What the name deliberately *excludes* is as important as what it includes: everything volatile (owner, subscription, dates, meaning-laden instance numbers) is kept out of names and pushed into **tags**, which can change freely. A companion tagging standard will cover ownership, cost center, and data classification.

---

*Changes to this standard are made via pull request. New resource type prefixes, region codes, or environment names must be added here before first use.*
