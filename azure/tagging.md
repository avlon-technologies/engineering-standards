# Tagging

**Status:** Active
**Applies to:** All resource groups and resources in all Avlon subscriptions.

---

## Purpose

Names identify resources; tags describe them. A resource name is immutable in practice —
most Azure resources cannot be renamed — so anything that can change over a resource's
life (who owns it, which cost center pays for it, which repo manages it) must live in
tags, where a change is an API call rather than a destroy-and-recreate.

Without a fixed tag set, tags decay into folklore: `Owner`, `owner`, and `OwnedBy` on
three sibling resources, cost reports full of "untagged", and a 2 a.m. incident where
nobody can tell which team to page for the resource that's on fire. Tags are only useful
if they are **consistent, complete, and machine-applied** — which is what this standard
enforces.

## Guiding Principles

1. **Names carry identity; tags carry metadata.** What a resource *is* — its type,
   workload, environment, region — is stable and belongs in the name per
   [the naming standard](../naming/azure.md). Everything volatile — ownership, billing,
   management provenance — belongs in tags. Never duplicate a fact into both places
   unless the duplication is load-bearing (see `workload` and `environment` below, which
   exist in tags because names cannot be *queried* as structured fields).
2. **Every tag must have a consumer.** A tag nobody queries is noise that decays. Each
   required tag below exists because a specific process — cost allocation, incident
   routing, drift detection, audit — reads it.
3. **Tags are applied by Terraform, not by humans.** Hand-applied tags drift and lie.
   The tag values are derived from the same variables that drive naming, so they cannot
   disagree with reality.
4. **Enforce presence on resource groups; audit everywhere else.** Policy that hard-fails
   resource creation over a missing tag breaks more than it fixes (see Standard). The RG
   is where completeness is required; resources inherit the discipline via Terraform.
5. **Lowercase kebab-case keys, controlled values.** `cost-center` not `CostCenter`.
   Tag queries are case-sensitive in enough tools that one canonical form is the only
   safe number of forms.

## Standard

### Required tags

Required on **every resource group**, and applied to every resource via Terraform:

| Tag | Values | Who consumes it |
|-----|--------|-----------------|
| `workload` | Workload name, e.g. `billing`, `code-review`, `platform-acr` | Cost attribution and estate queries: "everything belonging to billing" as a structured filter, across RGs, regions, and subscriptions — which name-prefix matching cannot do reliably. |
| `environment` | `dev` \| `stg` \| `prod` \| `shared` (the canonical set from [Environments](environments.md)) | Policy assignment, budget scoping, alert routing severity ("prod pages, dev emails"). |
| `owner` | Team alias, e.g. `platform-team`, `billing-team` — **never a person** | Incident routing at 2 a.m. People change teams, go on leave, and quit; teams answer pagers. A person's name in `owner` is a dead end waiting to happen. |
| `cost-center` | Finance cost-center code | Chargeback and showback. This is the tag finance actually joins against; without it, cloud spend is one undifferentiated invoice line. |
| `managed-by` | `terraform` \| `manual` | Drift detection and cleanup. Anything `manual` is a standing question ("why does this exist outside IaC?"); anything untagged is presumed abandoned. `manual` is for the short list of genuinely hand-managed exceptions, each of which should be an issue somewhere. |
| `repository` | GitHub repo URL that manages the resource, e.g. `https://github.com/avlon/billing` | The road back to the code. From a portal resource to the Terraform that declares it in one click — during incidents, audits, and offboarding. This is also where the resource↔repo relationship lives, so repo refactors update a tag instead of moving resources (see [Resource Groups](resource-groups.md)). |

Optional where relevant:

| Tag | Values | Purpose |
|-----|--------|---------|
| `data-classification` | `public` \| `internal` \| `confidential` \| `restricted` | Marks data-bearing resources for security review scoping and access decisions. Required in practice on anything holding customer data. |

Tag keys are lowercase kebab-case. Do not invent additional org-wide tags ad hoc —
propose them here first. Workload-local tags are allowed but must not collide with or
shadow the required set.

### Application via Terraform

Tags are built once in `locals` and applied to everything, so a value change is one edit
and the tags can never disagree between sibling resources:

```hcl
locals {
  tags = {
    workload    = var.workload            # "billing"
    environment = var.environment          # "prod"
    owner       = "billing-team"
    cost-center = var.cost_center          # "cc-2140"
    managed-by  = "terraform"
    repository  = "https://github.com/avlon/billing"
  }
}

resource "azurerm_resource_group" "main" {
  name     = "rg-${var.workload}-${var.environment}-${local.region}-01"
  location = var.location
  tags     = local.tags
}

resource "azurerm_key_vault" "main" {
  name                = "kv-${var.workload}-${var.environment}-${local.region}-01"
  resource_group_name = azurerm_resource_group.main.name
  location            = var.location
  tags                = local.tags
  # ...
}
```

Every taggable resource gets `tags = local.tags` (merged with resource-specific
additions via `merge(local.tags, {...})` where needed). Provider-level `default_tags`
is acceptable where the provider supports it, but the explicit pattern is preferred
because it survives module boundaries visibly.

### Enforcement via Azure Policy

- **Resource groups: require.** A policy at the `avlon` management group denies creation
  of resource groups missing any required tag. RGs are created deliberately and rarely,
  by Terraform, so a hard gate costs nothing and guarantees the estate's index is
  complete.
- **Resources: audit, not deny.** A companion policy *audits* resources missing required
  tags and surfaces them in compliance reports. It does **not** deny. Deny-on-resources
  causes real pain for phantom benefit: many resources are created implicitly by Azure
  itself (managed disks, network watchers, App Service slots, resources created by
  resource providers mid-operation) with no opportunity to tag at creation, so deny
  policies break platform operations and failovers in ways that surface as inexplicable
  provider errors mid-deploy. Since Terraform applies tags to everything it creates
  anyway, the audit report is a *drift detector* — its findings are either a workload
  skipping `tags = local.tags` (fix the code) or a hand-made resource (fix the process).

Do not use tag *inheritance* policies (auto-copying RG tags onto resources) as a
substitute for Terraform-applied tags: modify-effect policies fight Terraform's view of
the resource and generate perpetual plan noise.

## Examples

Tags on the canonical billing production RG:

```text
rg-billing-prod-cc-01
  workload            = billing
  environment         = prod
  owner               = billing-team
  cost-center         = cc-2140
  managed-by          = terraform
  repository          = https://github.com/avlon/billing
  data-classification = confidential
```

Queries these tags make trivial:

```bash
# What does billing cost across all environments and regions?
az graph query -q "resources | where tags['workload'] == 'billing' | project name, resourceGroup, tags['environment']"

# Everything not managed by Terraform (cleanup candidates)
az graph query -q "resources | where tags['managed-by'] != 'terraform' or isnull(tags['managed-by']) | project name, resourceGroup"
```

## Anti-patterns

| Anti-pattern | Why it fails |
|--------------|--------------|
| A person's name in `owner` | People move; teams persist. The tag is for paging, not credit. |
| Hand-editing tags in the portal on Terraform-managed resources | The next `terraform apply` reverts it, or worse, the drift persists and the tag lies. Change the code. |
| Free-form keys (`Owner`, `Env`, `CostCentre`) | Case- and spelling-variants split every query. One canonical key set, lowercase kebab-case. |
| Encoding tag-shaped data in names (`sql-billing-prod-teamx-cc2140`) | Names are immutable; the data isn't. This is exactly the volatile-metadata mistake the names-vs-tags doctrine exists to prevent. |
| Deny-effect tag policy on resources | Breaks implicitly-created resources and mid-operation provider calls. Audit on resources, deny on RGs only. |
| Tags with no consumer ("let's add `project-phase`") | Unconsumed tags decay into misinformation. Every tag earns its place with a named consumer. |

## Tradeoffs

- **Six required tags is real ceremony for small resources.** The `locals` pattern
  reduces the cost to near zero after the first resource, and the alternative — partial
  tagging — makes every estate-wide query return "mostly, except…", which is barely
  better than nothing.
- **Audit-not-deny on resources means brief windows of untagged resources.** A workload
  that forgets `tags = local.tags` isn't blocked; it shows up in the next compliance
  report instead. We accept eventual consistency here because the deny alternative
  breaks legitimate platform operations, and Terraform review catches most omissions
  before they ship.
- **`repository` goes stale when repos move.** Renaming or splitting a repo requires a
  tag update — a one-line Terraform change. That is precisely the deal: volatile data in
  the mutable store. The alternative (repo identity in names or RG structure) turns the
  same event into a migration.

## Related Standards

- [Azure Resource Naming](../naming/azure.md) — the identity half of the names-vs-tags doctrine
- [Resource Groups](resource-groups.md) — the RG as the primary tagged scope
- [Environments](environments.md) — canonical values for the `environment` tag
- [Terraform Variables](../terraform/variables.md) — the variables tag values derive from
