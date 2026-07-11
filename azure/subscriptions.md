# Subscriptions

**Status:** Active
**Applies to:** All Azure subscriptions in the Avlon Technologies tenant.

---

## Purpose

The subscription is the strongest boundary Azure offers short of a separate tenant. It is
simultaneously an **isolation boundary** (RBAC assignments, policy assignments, and many
quotas stop at its edge), a **governance boundary** (Azure Policy, Defender plans, and
budgets are most naturally assigned there), and a **billing boundary** (invoices and cost
exports are per-subscription by default). A blast-radius mistake at subscription scope —
an over-broad role assignment, a runaway quota consumer, a misassigned policy — affects
everything inside it and nothing outside it.

Because subscriptions are hard to restructure after the fact (moving resources between
subscriptions is possible but restricted and disruptive), the subscription model needs to
be decided deliberately, once, and designed so that growth doesn't force a painful
reorganization. This document defines that model.

## Guiding Principles

1. **Subscriptions separate trust tiers, not teams or projects.** The question a
   subscription boundary answers is "what is the blast radius of a compromise or mistake
   here?" — not "who works on this?". Team and workload separation is done with resource
   groups and RBAC (see [Resource Groups](resource-groups.md)), which are cheap and
   reversible; subscription separation is reserved for the boundaries that must be strong.
2. **Production and non-production must never share a subscription.** A subscription-scoped
   mistake in dev must be physically incapable of touching prod. This is the one
   subscription split that is non-negotiable at any scale.
3. **Platform infrastructure is not a workload.** Shared capabilities — Terraform state,
   the container registry, hub networking, central monitoring — serve every environment
   and outlive any individual workload. They get their own subscription with their own,
   tighter access model.
4. **Design for the split before you need it.** Subscription counts should follow scale,
   not fashion. But every standard downstream of this one — naming, resource groups,
   tagging, state layout — must be written so that adding a subscription later moves
   boundaries without renaming or restructuring anything.
5. **Keep governance machinery proportionate.** Management-group hierarchies, landing-zone
   factories, and subscription vending are solutions to problems Avlon does not yet have.
   Adopt them when the estate demands it, not before.

## Standard

### The subscription model

| Subscription | Contains |
|--------------|----------|
| `sub-avlon-platform` | Shared platform capabilities only: Terraform state storage, the container registry, hub networking, central monitoring. |
| `sub-avlon-dev` | All workload `dev` environments. |
| `sub-avlon-prod` | All workload `prod` **and `stg`** environments. |

**Why stg lives with prod.** Staging's entire purpose is production parity (see
[Environments](environments.md)): same policies, same networking posture, same quotas,
same regional capacity behavior. Putting stg in the prod subscription means it is
governed by *exactly* the policy and configuration set that governs prod — parity is
structural rather than maintained by hand across two subscriptions. The tradeoff is
honest: stg and prod no longer have a subscription boundary between them, so their
isolation rests on resource groups and RBAC instead. We accept this because stg is
operated with prod-level discipline anyway (pipeline-only changes, no casual human
access), and because the alternative — a dedicated `sub-avlon-stg` — historically drifts:
policies get updated in prod and forgotten in stg, and the parity that justified staging
quietly erodes. Within `sub-avlon-prod`, stg and prod are separated by distinct resource
groups, distinct Entra ID groups, and distinct deployment identities; no principal holds
write access across both.

### Starting smaller

A small estate may start with a **single subscription**. This is an acceptable starting
point, not a violation — provided everything else in this handbook is followed. The
resource-group-per-workload-per-environment rule and the
[naming standard](../naming/azure.md) mean every resource already carries its workload
and environment identity, RBAC is already scoped at resource groups, and Terraform state
is already split per workload per environment. When scale justifies the split, moving a
workload's resource groups into `sub-avlon-dev` or `sub-avlon-prod` changes *where the
boundary sits*, not what anything is called or how any pipeline works. Names contain no
subscription information for exactly this reason.

The trigger to split is any of: the first real production workload, the first compliance
requirement that wants prod isolated, or subscription-level quota contention between
environments.

### Management groups

At current scale, a single management group `avlon` above all subscriptions is
sufficient. It exists so tenant-wide policy (required tags on resource groups, allowed
regions, audit baselines) is assigned once and inherited everywhere, and so future
subscriptions land under governance automatically. Do not build a deeper hierarchy
(per-department, per-archetype landing zones) until there are enough subscriptions that
the flat model demonstrably fails — a hierarchy of one-child nodes is pure ceremony.

### What belongs in the platform subscription

`sub-avlon-platform` contains only capabilities that serve all environments and are owned
by the platform team:

- **Terraform state**: `rg-platform-tfstate-shared-cc` with the state storage account
  `sttfstatesharedcc` — see [Remote State](../terraform/remote-state.md).
- **Container registry**: `rg-platform-acr-shared-cc` with `acrplatformsharedcc` —
  see [Container Registry](container-registry.md).
- **Hub networking**: `rg-platform-network-prod-cc` with the hub VNet, private DNS
  zones, and shared egress — see [Networking](networking.md).
- **Central monitoring**: `rg-platform-monitor-prod-cc` with the central Log Analytics
  workspace.

No workload resources, ever. The platform subscription has the tightest access model in
the tenant: only the platform team's Entra ID groups and platform deployment identities
hold roles there, because a compromise of platform infrastructure (state files, images,
DNS) is a compromise of every environment at once.

## Examples

Target-state layout:

```text
mg: avlon
├── sub-avlon-platform
│   ├── rg-platform-tfstate-shared-cc     # sttfstatesharedcc
│   ├── rg-platform-acr-shared-cc         # acrplatformsharedcc
│   ├── rg-platform-network-prod-cc       # hub vnet, private DNS
│   └── rg-platform-monitor-prod-cc       # central Log Analytics
├── sub-avlon-dev
│   ├── rg-code-review-dev-cc
│   └── rg-billing-dev-cc
└── sub-avlon-prod
    ├── rg-billing-stg-cc                 # stg colocated with prod, isolated by RG + RBAC
    ├── rg-billing-prod-cc
    └── rg-billing-prod-ce                # DR pair
```

Single-subscription starting point — same resource groups, one container:

```text
mg: avlon
└── sub-avlon-platform                        # doubles as the only subscription initially
    ├── rg-platform-tfstate-shared-cc
    ├── rg-code-review-dev-cc
    └── rg-billing-prod-cc
```

The later split is a resource-group move, not a rename.

## Anti-patterns

| Anti-pattern | Why it fails |
|--------------|--------------|
| A subscription per team or per project | Subscriptions multiply governance overhead (policy assignments, budgets, networking, deployment identities) while resource groups would have provided the isolation for free. |
| Dev and prod sharing a subscription past the first production workload | One subscription-scoped mistake — a role assignment, a policy, an exhausted quota — spans both. This is the boundary the model exists to enforce. |
| Workload resources in `sub-avlon-platform` | Dilutes the tightest access boundary in the tenant and entangles workload lifecycle with platform lifecycle. |
| Encoding subscription names into resource names | Guarantees names go stale on any reorganization. The [naming standard](../naming/azure.md) excludes subscription identity deliberately. |
| A deep management-group hierarchy at three subscriptions | Governance ceremony with no governed population. One `avlon` MG is enough until it isn't. |
| Splitting subscriptions "later, when we have time" without following the RG standard now | The RG and naming standards are what make the split non-breaking. Skip them and the future split becomes a migration project. |

## Tradeoffs

- **Stg-with-prod trades a subscription boundary for structural parity.** We rely on
  RG-level RBAC to keep stg and prod apart, which is a weaker wall than a subscription.
  We accept it because policy/config drift between a dedicated stg subscription and prod
  is a quieter, more damaging failure mode than the residual RBAC risk — and because stg
  is already operated pipeline-only.
- **Three subscriptions cost more governance than one.** Each needs budgets, policy
  assignments, Defender configuration, and deployment identity plumbing. We accept this
  because the dev/prod blast-radius separation is worth it the moment real production
  traffic exists.
- **Restraint on management groups means some future rework.** If the estate grows into
  many subscriptions, a hierarchy will need to be introduced and policies re-homed. We
  accept deferred, proportionate work over speculative structure that must be maintained
  from day one.

## Related Standards

- [Resource Groups](resource-groups.md) — the everyday isolation and lifecycle boundary inside each subscription
- [Environments](environments.md) — the canonical dev/stg/prod/shared definitions
- [Identity](identity.md) — RBAC model that makes RG-level isolation trustworthy
- [Remote State](../terraform/remote-state.md) — state storage in the platform subscription
- [Azure Resource Naming](../naming/azure.md) — why names carry no subscription identity
