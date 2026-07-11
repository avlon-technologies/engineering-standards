# Azure Standards

How Avlon Technologies builds on Azure. These documents define the organizational and
security decisions that every workload inherits before a single line of application code
is written: how subscriptions and resource groups are carved up, how resources are tagged,
how identities authenticate, how networks are laid out, and what the environments mean.

The philosophy is consistent across all of them:

- **Workload-oriented organization.** Resource groups, identities, state files, and cost
  reports are all organized around *workloads* — deployable business capabilities — not
  around git repositories, teams, or org charts, which change on a different schedule.
- **Security-first defaults.** Least-privilege RBAC on Entra ID groups, managed identities
  everywhere, no client secrets, private-by-default networking in staging and production.
- **Everything via Terraform.** Every resource described here is created and changed
  through Terraform and GitHub Actions, never by hand in the portal. The portal is for
  reading, not writing. See the [Terraform standards](../terraform/README.md).

## Contents

| Document | What it covers |
|----------|----------------|
| [Subscriptions](subscriptions.md) | Subscription strategy: what subscriptions isolate, the `sub-avlon-*` model, and how to start small without painting yourself into a corner. |
| [Resource Groups](resource-groups.md) | The core organizational rule: one resource group per workload per environment per region, with a shared lifecycle. |
| [Tagging](tagging.md) | The required tag set, why each tag exists, and how tags are applied and enforced. |
| [Identity](identity.md) | Entra ID, RBAC, managed identities, Workload Identity Federation, and human access to production. |
| [Networking](networking.md) | Private-by-default: hub-spoke topology, NSGs, private endpoints, and private DNS. |
| [Container Registry](container-registry.md) | The single shared registry, image naming and promotion, and registry access control. |
| [Environments](environments.md) | The canonical definition of `dev`, `stg`, `prod`, and `shared` — every other document that mentions environments defers to this one. |

**Azure resource naming** is deliberately *not* here: naming conventions for all
disciplines live in the [naming standards](../naming/README.md), and the Azure resource
naming standard specifically is [Azure Naming](../naming/azure.md).

## How these standards fit together

Read [Environments](environments.md) first — it defines the vocabulary (`dev`, `stg`,
`prod`, `shared`) that everything else uses. [Subscriptions](subscriptions.md) and
[Resource Groups](resource-groups.md) then describe the containers resources live in,
from the strongest boundary (subscription) down to the everyday unit of organization and
lifecycle (resource group). [Tagging](tagging.md) adds the metadata that names cannot
carry. [Identity](identity.md) and [Networking](networking.md) define the security
posture every workload gets by default, and [Container Registry](container-registry.md)
applies all of the above to the one piece of shared infrastructure every containerized
workload touches.

A new workload should be able to answer every "where does this go, what is it called,
who can touch it" question from these documents plus [Azure Naming](../naming/azure.md)
alone — if it can't, open a pull request against the standard that fell short.
