# Resource Groups

**Status:** Active
**Applies to:** All resource groups in all Avlon subscriptions.

---

## Purpose

The resource group is Azure's everyday unit of organization: the scope where RBAC is
normally granted, where cost rolls up naturally, and — critically — the only scope at
which you can delete a whole system in one deliberate act. Get resource group boundaries
right and most operational questions ("who owns this?", "what does it cost?", "can I
delete it?") answer themselves. Get them wrong and every one of those questions requires
archaeology.

The single most common way to get them wrong is to organize resource groups around
something that changes for unrelated reasons — the git repository layout, the team
structure, the current sprint. This document fixes the boundary: **resource groups are
organized by workload**, and everything inside one shares a lifecycle.

## Guiding Principles

1. **A resource group is a lifecycle, not a folder.** The defining test for RG membership
   is: *when this workload+environment is retired, does this resource die with it?* If
   yes, it belongs. If no — if anything else depends on it — it belongs somewhere else.
   An RG whose contents can be deleted together is safe to operate; an RG that mixes
   lifecycles can never be deleted at all, only picked apart.
2. **Workloads, not repositories.** Repositories are organized around code concerns:
   they split when a codebase grows, merge when a microservice experiment is walked back,
   and get renamed in refactors. None of those events changes what infrastructure exists
   or when it should die. Infrastructure lifecycle follows the *workload* — the deployable
   business capability — which is far more stable than any repo layout. Aligning RGs to
   repos means every repo refactor becomes an infrastructure migration; aligning them to
   workloads means repo refactors change a `repository` tag and nothing else.
3. **Environment and region are hard boundaries.** Dev and prod resources in one RG make
   RBAC impossible to scope ("developers can touch dev" cannot be expressed) and make the
   deletion test fail permanently. Regions are separated so a DR region can be operated —
   failed over, rebuilt, torn down — independently of the primary.
4. **RBAC wants a natural scope, and the RG is it.** Subscription-scoped roles are too
   broad; resource-scoped roles are unmanageable at any scale. A workload-per-environment
   RG makes "the billing team can operate billing in dev" a single group-to-RG assignment.
5. **Platform capabilities are workloads too.** Shared infrastructure gets the same
   discipline — dedicated RGs, clear ownership — not a `misc-shared` dumping ground.

## Standard

### One resource group per workload per environment per region

```text
rg-<workload>-<environment>-<region>-<instance>
```

Every workload gets exactly one RG per environment per region it deploys to. The name
follows [the Azure naming standard](../naming/azure.md); environments are the canonical
set from [Environments](environments.md). All resources belonging to that workload in
that environment and region live in that RG — the app, its database, its Key Vault, its
managed identity, its Application Insights, its spoke VNet.

### The lifecycle rule

**Everything in a resource group dies together.** Before placing a resource in an RG,
apply the test from Principle 1. The corollaries:

- Nothing outside the RG may depend on anything inside it (other than through published,
  environment-scoped interfaces like a private endpoint or an API).
- Shared or longer-lived resources — the container registry, hub networking, Terraform
  state — must **not** be placed in a workload RG, however convenient it seems. They go
  in platform RGs.
- Retiring a workload environment is `terraform destroy` (or, in the limit, deleting the
  RG) with no impact analysis beyond the workload itself. If deleting an RG requires a
  meeting, the RG boundary is wrong.

### Why not repository-aligned resource groups

This deserves spelling out because repo-aligned RGs feel natural to developers and fail
slowly:

- **Repos refactor on code concerns.** Splitting `billing` into `billing-api` and
  `billing-web` repos is a Tuesday. If RGs mirror repos, that Tuesday now requires moving
  production resources between RGs — an operation Azure restricts, that breaks resource
  locks and some resource types entirely, and that no one budgets for.
- **Deletion safety evaporates.** A repo can be archived while its workload lives on, or
  deleted while another repo still deploys into "its" RG. The lifecycle test fails.
- **RBAC stops mapping to responsibility.** Operations own workloads ("billing is down"),
  not repositories. Repo-shaped RGs force RBAC to be assembled from fragments.
- **Cost attribution fragments.** "What does billing cost in prod?" should be one RG's
  cost view, not a sum over however many repos currently produce billing components.

The repository relationship is real — it is just metadata, not structure. It lives in the
`repository` tag (see [Tagging](tagging.md)), where it can change freely.

### Platform resource groups

Platform capabilities get dedicated `rg-platform-*` resource groups, owned by the
platform team, in the platform subscription (see [Subscriptions](subscriptions.md)).
Each platform capability (`platform-network`, `platform-tfstate`, `platform-acr`,
`platform-monitor`) is treated as a workload in its own right: its own RG, its own
Terraform state, its own lifecycle. Platform RGs use `shared` as the environment segment
only when the capability genuinely serves all environments (state storage, the registry);
capabilities with an environment identity (the prod hub network, prod monitoring) use
`prod`.

### RBAC scoping

Role assignments are made at resource group scope by default: Entra ID groups (never
individuals) granted built-in roles on the workload's RG per [Identity](identity.md).
Subscription-scope assignments are reserved for genuinely subscription-wide duties
(platform operators, security readers). Resource-scope assignments are the rare
exception for unusually sensitive single resources.

### Tags

Every RG carries the full required tag set — `workload`, `environment`, `owner`,
`cost-center`, `managed-by`, `repository` — per [Tagging](tagging.md). The RG is where
tag policy is enforced hardest, because RG tags are the index for cost and ownership
queries across the estate.

## Examples

The canonical layout for the `code-review` and `billing` workloads plus platform
capabilities:

```text
sub-avlon-dev
├── rg-code-review-dev-cc-01        # Container Apps workload: cae, ca, kv, sql, mi, appi
└── rg-billing-dev-cc-01            # App Service + SQL workload, dev tier

sub-avlon-prod
├── rg-billing-stg-cc-01            # staging — full prod topology
├── rg-billing-prod-cc-01           # primary region: app, asp, sql, kv, vnet, pep, st
└── rg-billing-prod-ce-01           # DR region — independently operable

sub-avlon-platform
├── rg-platform-network-prod-cc-01  # hub VNet, private DNS zones
├── rg-platform-tfstate-shared-cc-01# sttfstatesharedcc01
├── rg-platform-acr-shared-cc-01    # acrplatformsharedcc01
└── rg-platform-monitor-prod-cc-01  # central Log Analytics
```

Note what the layout makes trivial: deleting `rg-code-review-dev-cc-01` retires that
environment completely; granting the billing team operate rights in dev is one assignment
on `rg-billing-dev-cc-01`; the prod cost of billing is one RG cost view (two with DR).

## Anti-patterns

| Anti-pattern | Why it fails |
|--------------|--------------|
| One RG per git repository | Repo refactors become infrastructure migrations; deletion safety and cost attribution fragment. See "Why not repository-aligned" above. |
| One big RG per environment (`rg-avlon-prod`) | No per-workload RBAC, no per-workload cost view, and the RG can never be deleted. It is a subscription wearing an RG costume. |
| Mixing environments in one RG | "Developers can touch dev" becomes inexpressible; the dev/prod distinction rests on people reading name suffixes carefully at 2 a.m. |
| Shared resources in a workload RG ("the registry can live with billing for now") | The workload RG becomes undeletable and the shared resource inherits the wrong owners, tags, and access. |
| Per-resource-type RGs (`rg-all-keyvaults-prod`) | Organizes by what resources *are* instead of what they're *for*; every workload question now spans many RGs. |
| Moving resources between RGs as routine practice | Azure resource moves are restricted, block some resource types, and invalidate locks and some identities. Get the boundary right at creation. |

## Tradeoffs

- **More resource groups than a coarse layout.** Two workloads across three environments
  and a DR region already produce seven workload RGs. We accept the count because RGs are
  free, and every one of them is a meaningful RBAC, cost, and deletion boundary — the
  alternative saves nothing but the scrolling.
- **Shared components force a decision.** When two workloads want to share a database
  server or a Container Apps environment, this standard forces either duplication (one
  per workload) or promotion to a platform capability with a platform owner. Both cost
  more than quietly sharing — and both are chosen deliberately, which is the point. Quiet
  sharing is how lifecycle coupling sneaks in.
- **The workload boundary requires judgment.** "One workload or two?" has no mechanical
  answer; the deciding question is whether the parts always deploy and retire together.
  We accept occasional boundary debates as far cheaper than the migrations that follow
  boundary mistakes.

## Related Standards

- [Azure Resource Naming](../naming/azure.md) — the `rg-` naming pattern and segments
- [Subscriptions](subscriptions.md) — which subscription each RG lives in
- [Tagging](tagging.md) — required tags on every RG, including `repository`
- [Identity](identity.md) — group-based RBAC assigned at RG scope
- [Environments](environments.md) — the canonical environment set
