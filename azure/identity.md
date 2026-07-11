# Identity

**Status:** Active
**Applies to:** All human and workload access to Azure resources across all Avlon subscriptions: Entra ID, RBAC, managed identities, and CI/CD authentication.

---

## Purpose

Almost every serious cloud incident is an identity incident wearing a costume: a leaked
secret, an over-broad role, a service principal nobody remembers creating, a departed
engineer's standing access. Azure gives us the tools to make whole classes of these
impossible — managed identities remove secrets, RBAC scopes remove over-breadth, PIM
removes standing human access — but only if they are the default, not the aspiration.

This document defines that default: who and what can authenticate, what they can touch,
and for how long. The recurring theme is **no credentials to steal and no access that
outlives its reason**.

## Guiding Principles

1. **Roles go to groups, never individuals.** A role assigned to a person is invisible
   at review time ("why does Mike have Contributor here?"), survives their team change,
   and must be hunted down when they leave. A role assigned to an Entra ID group makes
   joiners/leavers a group-membership change handled by one process, and makes an access
   review a readable list: *these groups have these roles at these scopes*. There are no
   exceptions — even a team of one gets a group.
2. **Built-in roles before custom roles.** Azure's built-in roles are maintained by
   Microsoft, understood by every engineer and auditor, and updated as resource types
   evolve. A custom role is a bespoke security artifact Avlon must maintain forever.
   Create one only when no built-in role fits and the gap is a real risk, not an
   aesthetic preference.
3. **Scope as narrow as practical — usually the resource group.** Subscription-scoped
   assignments are for genuinely subscription-wide duties (platform operators, security
   readers). The workload RG (see [Resource Groups](resource-groups.md)) is the natural
   grain for everything else.
4. **Workloads never hold secrets to prove who they are.** Every workload-to-Azure
   authentication uses a managed identity. A secret that exists can be leaked, logged,
   committed, or exfiltrated; an identity that Azure itself vouches for cannot.
5. **Humans get access when they need it, not in case they need it.** Standing write
   access to production is a standing attack surface. Elevation is just-in-time,
   time-boxed, and logged.

## Standard

### RBAC assignment rules

- Every role assignment targets an **Entra ID group** (or a managed identity — see
  below). Direct user assignments are a standards violation and are removed on sight.
- Groups are named for the access they grant, per workload and environment, e.g.
  `billing-team-dev-contributors`, `billing-team-prod-readers`, `platform-team-operators`.
- **Built-in roles** by default. Prefer the narrow built-ins (`Key Vault Secrets User`,
  `AcrPull`, `Storage Blob Data Reader`) over `Contributor`; prefer `Reader` over
  anything, wherever reading suffices.
- Default scope is the **workload resource group**. Narrower (single resource) is fine
  for unusually sensitive resources; broader (subscription) requires a platform-team
  decision.
- Role assignments are Terraform-managed like everything else, so access is reviewable
  in pull requests and reconstructable from code.

### Managed identities for all workload authentication

Every workload authenticates to Azure services (Key Vault, SQL, Storage, the container
registry) with a **managed identity** — no connection strings with embedded keys, no
service principal credentials.

**User-assigned identities are the default** (`mi-billing-prod-cc` pattern, one per
workload per environment). System-assigned identities are acceptable for a single
resource with no shared access needs, but user-assigned is preferred because:

- **It is shared across resources.** An App Service, its deployment slots, and a
  companion Function App can present one identity, so the workload's data-plane access
  is granted once, to one principal.
- **It survives resource recreation.** A system-assigned identity dies with its resource,
  taking every role assignment and SQL user mapping with it — so replacing a resource
  (a routine Terraform event) silently severs access. A user-assigned identity has its
  own lifecycle and its grants persist across recreations.
- **Its lifecycle is explicit.** It is a first-class Terraform resource in the workload
  stack, visible in plans, named per the [naming standard](../naming/azure.md), and
  deleted deliberately when the workload retires.

### No service principal client secrets

Service principals with client secrets (or certificates managed by hand) are prohibited
for both workloads and CI/CD. A client secret is a password: it must be stored somewhere,
rotated on a schedule nobody keeps, and revoked in a hurry when it leaks. Every scenario
that historically needed one now has a secretless alternative — managed identities inside
Azure, Workload Identity Federation outside it. A third-party tool that can *only*
authenticate with a client secret is a procurement red flag, not an exception request.

### CI/CD: Workload Identity Federation

GitHub Actions authenticates to Azure via **Workload Identity Federation (OIDC)** — the
pipeline exchanges a GitHub-issued OIDC token for Azure credentials at run time. Nothing
is stored in GitHub secrets that could authenticate to Azure.

- One deployment identity per workload per environment, `mi-github-billing-prod-cc`
  pattern, with federated credentials bound to the specific repository and GitHub
  environment (`dev`, `stg`, `prod`).
- Each deployment identity is granted only the roles its stack needs, at the workload RG
  scope — a compromised dev pipeline cannot touch prod, because the prod identity's
  federation subject only matches the `prod` GitHub environment, which is protected.

Pipeline mechanics live in [GitHub Actions](../github/github-actions.md); the Terraform
side (provider auth, state access) in [Terraform Security](../terraform/security.md).

### Human access to production

- Day-to-day: humans hold **read** roles in prod at most.
- Write or data-plane access is obtained through **Privileged Identity Management (PIM)**:
  eligible (not active) assignments on the relevant groups, activated just-in-time with
  justification, time-boxed to hours, and logged. Prod incidents are the use case;
  convenience is not.
- Routine change reaches production only via pipelines — PIM is for operating, not
  deploying (see [Environments](environments.md)).

### Break-glass accounts

The tenant maintains two cloud-only emergency-access accounts, excluded from conditional
access policies that could lock them out (MFA-method failures, federation outages),
protected with phishing-resistant credentials stored offline, monitored so that any
sign-in pages the platform team. They exist to recover the tenant, not to bypass PIM.
Details live in the platform team's runbook, not in workload documentation.

## Examples

The billing workload's identity plumbing in prod:

```hcl
resource "azurerm_user_assigned_identity" "billing" {
  name                = "mi-billing-prod-cc"
  resource_group_name = azurerm_resource_group.main.name
  location            = var.location
  tags                = local.tags
}

# Workload identity reads secrets — narrow built-in role, single-resource scope
resource "azurerm_role_assignment" "kv_secrets" {
  scope                = azurerm_key_vault.main.id      # kv-billing-prod-cc
  role_definition_name = "Key Vault Secrets User"
  principal_id         = azurerm_user_assigned_identity.billing.principal_id
}

# Workload identity pulls images from the shared registry
resource "azurerm_role_assignment" "acr_pull" {
  scope                = data.azurerm_container_registry.platform.id  # acrplatformsharedcc
  role_definition_name = "AcrPull"
  principal_id         = azurerm_user_assigned_identity.billing.principal_id
}

# Humans: the team group gets read on the RG; write in prod is PIM-eligible only
resource "azurerm_role_assignment" "team_read" {
  scope                = azurerm_resource_group.main.id  # rg-billing-prod-cc
  role_definition_name = "Reader"
  principal_id         = data.azuread_group.billing_team_prod_readers.object_id
}
```

## Anti-patterns

| Anti-pattern | Why it fails |
|--------------|--------------|
| Role assigned to an individual user | Unreviewable, survives team changes, must be hunted at offboarding. Groups only. |
| Service principal with a client secret in GitHub secrets | A durable credential that can leak and must be rotated. WIF makes it unnecessary. |
| `Contributor` at subscription scope "to unblock the team" | The blast radius of every future mistake is now the subscription. Scope to the RG, use the narrow role. |
| System-assigned identity for a workload with grants in SQL/Key Vault | Resource recreation deletes the identity and orphans every grant — access breaks on a routine replace. |
| Connection strings with account keys instead of identity-based access | Reintroduces a secret where the platform offered none. Use managed identity auth (Entra-authenticated SQL, `use_azuread_auth` for storage). |
| Standing `Owner`/`Contributor` on prod for on-call convenience | Standing attack surface for a rare need. PIM activation takes a minute; a compromised standing credential takes a quarter to clean up. |
| Custom role because a built-in has "too many" read permissions | A permanent bespoke artifact to save theoretical breadth. Reserve custom roles for real risk gaps. |

## Tradeoffs

- **Groups add indirection for tiny teams.** Creating `billing-team-dev-contributors`
  for two people feels heavy. We accept it because the cost is one-time and the
  alternative — person-shaped access — fails precisely at the moments (departure,
  incident, audit) when it is most expensive to fail.
- **PIM adds minutes to prod incident response.** Activation with justification is
  friction at the worst time. We accept it because the alternative is every operator
  being a standing prod credential around the clock, and because activation is
  streamlined for the on-call group.
- **User-assigned identities are more Terraform to write.** A resource, its role
  assignments, and its wiring into each consumer — versus a checkbox for system-assigned.
  We accept the verbosity for grants that survive resource recreation and one principal
  per workload instead of one per resource.
- **WIF setup is fiddlier than pasting a secret.** Federated credential subjects must
  exactly match repo and environment. We accept the one-time fiddliness for the permanent
  absence of a stealable CI credential.

## Related Standards

- [GitHub Actions](../github/github-actions.md) — the pipeline side of Workload Identity Federation
- [Terraform Security](../terraform/security.md) — provider and state authentication
- [Resource Groups](resource-groups.md) — the default RBAC scope
- [Container Registry](container-registry.md) — AcrPush/AcrPull assignments in practice
- [Azure Resource Naming](../naming/azure.md) — `mi-*` identity naming
