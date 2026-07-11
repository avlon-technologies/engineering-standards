# Backend Bootstrap

**Status:** Active
**Applies to:** The one-time creation and rare maintenance of Avlon's Terraform state storage.

---

## Purpose

Every Terraform stack at Avlon stores its state in `sttfstatesharedcc01` ([Remote State](remote-state.md)) — which raises the obvious chicken-and-egg problem: the storage account that holds all state cannot store its own state before it exists, and no workload stack should be trusted to create it as a side effect. Without a deliberate answer, teams improvise: a hand-clicked storage account nobody codified, or backend resources buried inside a workload stack where a `terraform destroy` takes the whole organization's state with it. This standard defines the answer: a minimal, dedicated **bootstrap configuration**, run once by a human, whose own state is then migrated into the storage it created.

## Guiding Principles

1. **The backend is created by code, like everything else** — just code with an unusual first run. "It's a bootstrap problem" is not a license for portal clicks.
2. **Bootstrap is minimal.** It creates exactly what Terraform needs to have a backend, and nothing else. Every resource added to bootstrap is a resource that exists below the safety net.
3. **Run once, by a human, deliberately.** Bootstrap is the one configuration that cannot start in CI (the CI identities it authorizes don't exist yet). A platform engineer runs it locally, eyes open.
4. **Bootstrap eats its own cooking.** After the first apply, bootstrap's state migrates into the storage it created. From then on, even bootstrap uses the standard backend.
5. **Changes are rare and loud.** Bootstrap changes the ground everything stands on. Every change is a reviewed PR applied by the platform team, never routine.

## Standard

### What bootstrap is

A minimal root configuration living in the **`platform-tfstate`** repository (platform capability, dedicated repo — see [Repository Layout](repository-layout.md)). It creates, in the `shared` environment:

- `rg-platform-tfstate-shared-cc-01` — the resource group
- `sttfstatesharedcc01` — the storage account, hardened: shared-key access **disabled**, public blob access **disabled**, TLS 1.2 minimum, blob **versioning** and 30-day **soft delete** enabled
- The four state containers: `tfstate-dev`, `tfstate-stg`, `tfstate-prod`, `tfstate-shared`
- RBAC: `Storage Blob Data Contributor` per container for the CI deployment identities, and the platform team's group assignment (see [Identity](../azure/identity.md))

### What bootstrap must NOT contain

No workload resources. No networking, no registries, no monitoring, no "while we're in here" additions — those are ordinary platform stacks (`platform-network`, `platform-acr`, `platform-monitor`) with ordinary remote state in `tfstate-shared`. Bootstrap's scope is frozen at "what Terraform needs before Terraform works." If a resource would survive fine as a normal stack, it is a normal stack.

### The bootstrap procedure

**Step 1 — apply with local state.** A platform engineer, authenticated as themselves:

```bash
az login
az account set --subscription sub-avlon-platform
terraform init          # no backend block yet → local state
terraform plan          # review carefully; this is the foundation
terraform apply
```

**Step 2 — migrate state into the storage just created.** Add the backend block (bootstrap's own state lives with the other platform stacks in `tfstate-shared`):

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-platform-tfstate-shared-cc-01"
    storage_account_name = "sttfstatesharedcc01"
    container_name       = "tfstate-shared"
    key                  = "platform-tfstate.tfstate"
    use_azuread_auth     = true
  }
}
```

Then:

```bash
terraform init -migrate-state   # answer "yes" to copy local state to the new backend
terraform plan                  # must show no changes — the migration moved state, not resources
```

**Step 3 — destroy the evidence.** Delete `terraform.tfstate` and `terraform.tfstate.backup` from the working directory (they were never committed — `*.tfstate` is `.gitignore`d). Commit the backend block. Bootstrap is now a normal remote-state stack.

From this point, everything downstream — every workload, every platform stack — simply uses the standard backend from its first `init`. The chicken-and-egg problem is solved exactly once, permanently.

**Subsequent changes** (a new container for a new environment, an RBAC grant for a new workload's CI identity) are ordinary PRs to `platform-tfstate`, planned and applied by the platform team. They should be rare; if bootstrap changes monthly, something that belongs in a normal stack has leaked into it.

## Examples

The bootstrap `main.tf`, essentially in full:

```hcl
resource "azurerm_resource_group" "tfstate" {
  name     = "rg-platform-tfstate-shared-cc-01"
  location = "canadacentral"
  tags     = local.tags
}

resource "azurerm_storage_account" "tfstate" {
  name                = "sttfstatesharedcc01"
  resource_group_name = azurerm_resource_group.tfstate.name
  location            = azurerm_resource_group.tfstate.location

  account_tier             = "Standard"
  account_replication_type = "GZRS"      # state survives a regional outage

  shared_access_key_enabled       = false # Entra ID auth only — no key path exists
  allow_nested_items_to_be_public = false # no public blobs, ever
  min_tls_version                 = "TLS1_2"

  blob_properties {
    versioning_enabled = true            # every state write is a recoverable version

    delete_retention_policy {
      days = 30                          # soft delete: blobs
    }
    container_delete_retention_policy {
      days = 30                          # soft delete: containers
    }
  }

  lifecycle {
    prevent_destroy = true               # see security.md — this account is unkillable by plan
  }

  tags = local.tags
}

resource "azurerm_storage_container" "state" {
  for_each = toset(["tfstate-dev", "tfstate-stg", "tfstate-prod", "tfstate-shared"])

  name                  = each.key
  storage_account_id    = azurerm_storage_account.tfstate.id
  container_access_type = "private"
}
```

Because shared-key access is disabled, the bootstrap provider block sets `storage_use_azuread = true` so the engineer's own `az login` identity is used for data-plane operations during the apply (see [Providers](providers.md) for the full provider standard).

## Anti-patterns

| Anti-pattern | Why it's wrong |
|--------------|----------------|
| Creating the state storage account by hand in the portal | The most critical storage account in the org would be the only one not in code — unreviewable, unreproducible, config drift guaranteed. |
| Backend resources inside a workload or general platform stack | A `terraform destroy` of that stack deletes every team's state. The backend must sit outside everything it protects. |
| Skipping the state migration ("bootstrap state can stay local") | Bootstrap's local state sits on one laptop; the storage account becomes unmanageable the day that laptop or engineer leaves. Migrate, always. |
| Committing bootstrap's pre-migration local state to git | State is plaintext secrets and is never committed — bootstrap gets no exception. |
| Letting bootstrap grow (registry here, Log Analytics there) | Every added resource lives below the safety net with a manual-first-run legacy. Bootstrap is the backend, full stop. |
| Re-running bootstrap from scratch to "fix" it | It exists and has remote state; changes are ordinary plan/apply PRs. Re-bootstrapping risks orphaning every existing state file. |

## Tradeoffs

- **One manual apply exists in the org's history.** Bootstrap's first run contradicts "everything through CI." It is unavoidable — the CI identities don't exist before bootstrap grants them — and it is contained: one configuration, one run, then migrated onto the standard rails.
- **Bootstrap has privileged, human-run maintenance forever.** Platform engineers applying locally is a wider trust grant than pipeline-only applies. The configuration's tiny, frozen scope and mandatory PR review keep that surface small.
- **Its own state lives inside the thing it manages.** After migration, bootstrap's state is stored in the account bootstrap manages — a benign circularity in normal operation (changes don't recreate the account, and `prevent_destroy` blocks the fatal case), but recovering from a truly destroyed account means one fresh local-state bootstrap run plus `terraform import`. We document it rather than pretend it away.

## Related Standards

- [Remote State](remote-state.md) — the layout and rules of the storage bootstrap creates
- [Security](security.md) — `prevent_destroy`, state secrecy, deployment identities
- [Providers](providers.md) — provider configuration for the bootstrap root
- [Identity](../azure/identity.md) — the RBAC model bootstrap's grants follow
- [Azure Resource Groups](../azure/resource-groups.md) — platform RG conventions
