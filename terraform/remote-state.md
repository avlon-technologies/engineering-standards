# Remote State

**Status:** Active
**Applies to:** All Terraform state for all Avlon workloads and platform stacks.

---

## Purpose

State is the most dangerous file Terraform produces. It is the single record of what exists and how it maps to code; it contains every secret any resource ever emitted (connection strings, keys, generated passwords) in **plaintext**; and if it is lost, corrupted, or applied concurrently, routine changes become incidents. Local state on a laptop is a single point of failure with no locking and no access control. This standard defines exactly where state lives, how it is partitioned, how it is authenticated, and who can touch it.

## Guiding Principles

1. **State never lives on a machine or in git.** It lives in one hardened Azure Storage account, and nowhere else, from the first `terraform init`.
2. **One state per deployable workload per environment.** State partitioning *is* blast-radius partitioning.
3. **Identity-based access only.** No storage access keys, no SAS tokens. If you can't get to state through Entra ID RBAC, you can't get to state.
4. **State is a secret store, whether we like it or not.** Everything that can read a state file can read the secrets inside it. Access is scoped accordingly.
5. **Recoverability is designed in.** Versioning and soft delete mean a corrupted or deleted state is a rollback, not a disaster-recovery exercise.

## Standard

### Where state lives

All state resides in the storage account **`sttfstatesharedcc01`** in resource group **`rg-platform-tfstate-shared-cc-01`**, owned by the platform team. This account is created and maintained by the bootstrap configuration ([Backend Bootstrap](backend-bootstrap.md)) — no workload stack ever manages it.

Within the account, **one blob container per environment**:

| Container | Holds state for |
|-----------|----------------|
| `tfstate-dev` | All workload dev stacks |
| `tfstate-stg` | All workload stg stacks |
| `tfstate-prod` | All workload prod stacks |
| `tfstate-shared` | Platform stacks (`platform-network`, `platform-acr`, `platform-monitor`) |

Within a container, the blob key is **`<workload>.tfstate`** — e.g. container `tfstate-prod`, key `billing.tfstate`. The environment is encoded in the container (where RBAC applies), not the key, so it never needs repeating. Full key conventions in [Terraform Naming](../naming/terraform.md).

### One state per workload per environment

Never combine workloads into a shared state, and never combine environments. Small states are the point:

- **Blast radius.** A bad refactor or forced `state rm` in `code-review.tfstate` cannot touch billing. A monolithic state makes every apply a bet on everything.
- **Lock contention.** State locks are exclusive. Shared state means the billing deploy queues behind the code-review deploy for no reason.
- **Plan time.** Plan refreshes every tracked resource. Small states plan in seconds; monoliths in tens of minutes, which quietly kills the PR-plan workflow.
- **RBAC.** Access is granted per container (per environment). Per-workload keys additionally keep audit trails and incident scopes legible.

If a single workload grows genuinely independent deployable parts (e.g. `billing-api` and `billing-web` shipping on different cadences), split into `billing-api.tfstate` and `billing-web.tfstate` — the unit is the *deployable*, not the repo.

### Backend configuration

Each environment root's `backend.tf` (see [Repository Layout](repository-layout.md)):

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

**`use_azuread_auth = true` is mandatory.** Without it, the backend falls back to storage access keys — long-lived, unauditable, account-wide credentials that bypass container-level RBAC entirely and grant whoever holds them every environment's state at once. With Entra ID auth, access is per-identity, per-container, logged, and revocable; the account itself has shared-key access disabled (enforced by bootstrap), so the key path simply does not exist.

### Durability, history, and locking

The bootstrap configuration enables on `sttfstatesharedcc01`:

- **Blob versioning** — every state write creates a new version. Recovering from a corrupted state or a destructive mistake is "promote the previous version", with full history of who wrote what, when.
- **Soft delete** (30 days, blobs and containers) — a deleted state blob is recoverable, not gone.
- **Locking** comes free with the `azurerm` backend via **blob leases**: Terraform acquires a lease on the state blob for the duration of plan/apply, so concurrent runs fail fast with a lock error instead of corrupting state. Never use `-lock=false`; if a crashed run leaves a stale lease, break it deliberately (`terraform force-unlock` / break the blob lease) after confirming nothing is running.

### Who gets access

State contains secrets in plaintext — treat read access to a state container as equivalent to read access to that environment's secrets (see [Security](security.md)):

- **CI deployment identities**: each workload's per-environment identity (e.g. `mi-github-billing-prod-cc-01`) gets `Storage Blob Data Contributor` scoped to its environment's container only. The dev identity cannot see `tfstate-prod`.
- **Platform team**: data-plane access via Entra ID group, PIM-elevated for prod/shared containers (see [Identity](../azure/identity.md)).
- **Everyone else**: nothing. Engineers debugging state go through plan output, `terraform state list` under an authorized identity, or the platform team — not ad-hoc blob downloads.

### Cross-state references

Sometimes a workload needs something another stack created — the hub VNet from `platform-network`, the shared registry from `platform-acr`. Two mechanisms exist; prefer the second:

- `terraform_remote_state` data source — reads another stack's *entire* state output set. Use sparingly: it requires read access to the other state (and therefore to its secrets), and it couples you to that stack's output names and refresh cadence.
- **Explicit provider data sources looking up well-known names** — because names are deterministic ([Azure Naming](../naming/azure.md)), you can look resources up directly without touching anyone's state:

```hcl
data "azurerm_virtual_network" "hub" {
  name                = "vnet-platform-prod-cc-01"
  resource_group_name = "rg-platform-network-prod-cc-01"
}
```

This grants no state access, leaks no secrets, and couples only to the naming standard — which is exactly the contract deterministic naming exists to provide.

## Examples

The `code-review` dev root:

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-platform-tfstate-shared-cc-01"
    storage_account_name = "sttfstatesharedcc01"
    container_name       = "tfstate-dev"
    key                  = "code-review.tfstate"
    use_azuread_auth     = true
  }
}
```

The full state layout for the canonical workloads:

```text
sttfstatesharedcc01/
├── tfstate-dev/       code-review.tfstate, billing.tfstate
├── tfstate-stg/       code-review.tfstate, billing.tfstate
├── tfstate-prod/      code-review.tfstate, billing.tfstate
└── tfstate-shared/    platform-network.tfstate, platform-acr.tfstate, platform-monitor.tfstate
```

## Anti-patterns

| Anti-pattern | Why it's wrong |
|--------------|----------------|
| Local state, even "temporarily" | No locking, no backup, no access control, secrets on a laptop. Configure the backend before the first apply. |
| State committed to git | Plaintext secrets in history forever, guaranteed merge corruption. `*.tfstate` is `.gitignore`d in every repo. |
| Access keys or SAS tokens for backend auth | Account-wide, long-lived, bypasses container RBAC. `use_azuread_auth = true`, always; shared-key access is disabled on the account. |
| One `avlon.tfstate` for everything | Maximum blast radius, one global lock, glacial plans. One state per workload per environment. |
| A second state storage account per team/workload | Fragments RBAC, backup posture, and bootstrap. There is one account; partitioning happens at containers and keys. |
| `terraform_remote_state` as the default integration mechanism | Grants secret-equivalent state access for what is usually a name lookup. Use provider data sources against well-known names. |
| Routine `-lock=false` or casual `force-unlock` | The lock is the only thing standing between two concurrent applies and corrupted state. |

## Tradeoffs

- **One account is a shared dependency.** All Terraform operations depend on `sttfstatesharedcc01`; an outage blocks every deploy (running infrastructure is unaffected). We accept this over per-team accounts because a single account gets versioning, RBAC, monitoring, and bootstrap discipline applied once and correctly.
- **Many small states mean orchestration seams.** What a monolith did in one apply may take two ordered applies across stacks. Deterministic naming and data sources absorb most of it; the rest is documented apply order — a fair price for bounded blast radii.
- **Entra ID auth requires identity plumbing before day one.** You can't init until an identity has container RBAC. [Backend Bootstrap](backend-bootstrap.md) and the workload onboarding process exist precisely to make this a solved, one-time problem.

## Related Standards

- [Backend Bootstrap](backend-bootstrap.md) — how the state storage itself comes to exist
- [Security](security.md) — state secrecy, deployment identities, OIDC
- [Environments](environments.md) — why each environment root points at its own container
- [Identity](../azure/identity.md) — RBAC, groups, PIM
- [Azure Naming](../naming/azure.md) — deterministic names that make data-source lookups possible
- [Terraform Naming](../naming/terraform.md) — state key conventions
