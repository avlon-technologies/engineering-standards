# Container Registry

**Status:** Active
**Applies to:** All container images built and deployed by Avlon workloads, and the shared registry that holds them.

---

## Purpose

The container registry is where "what we tested" and "what we run" either stay the same
artifact or quietly stop being one. A registry strategy answers three questions: how many
registries exist, what images are called inside them, and who can push and pull. Get the
first one wrong — a registry per environment, a registry per team — and the answers to
everything downstream (promotion, scanning, access) fragment with it.

Avlon runs **one shared registry for the organization**. This document explains why, and
defines the image naming, tagging, access, and lifecycle rules inside it.

## Guiding Principles

1. **Promote images, never rebuild them.** The image that passed staging must be
   byte-for-byte the image that reaches production. Rebuilding "the same" source per
   environment produces a different artifact every time — different base-layer pulls,
   different dependency resolutions, different timestamps — and means production runs
   something that was never tested. One registry makes promotion a retag; per-environment
   registries make it a rebuild or a cross-registry copy that teams inevitably shortcut.
2. **One scanning surface.** Vulnerability scanning, image signing, and quarantine
   policies configured once, covering every image the organization runs. Three registries
   means three configurations, drifting.
3. **The registry is platform infrastructure.** Like Terraform state and hub networking,
   it serves all environments and outlives every workload — so it lives in the platform
   subscription, owned by the platform team, with environment separation expressed
   through *access control*, not through registry multiplication.
4. **Deploys pin what they mean.** A tag is a moving pointer; a digest is an artifact.
   Production runs digests.
5. **Identity-based access only.** Pushing is a CI privilege via OIDC; pulling is a
   workload privilege via managed identity. The admin account stays disabled; no registry
   passwords exist anywhere.

## Standard

### One registry

- **Registry:** `acrplatformsharedcc01`, in `rg-platform-acr-shared-cc-01`, in the
  platform subscription (see [Subscriptions](subscriptions.md)).
- **SKU: Premium.** Required for production use: private link support (the registry gets
  a private endpoint per [Networking](networking.md)), higher storage and throughput
  ceilings, and geo-replication when a second region needs local pulls (add the `ce`
  replica when prod workloads actually run there — replication is a checkbox later, not
  a rename).
- Why not per-environment registries: rebuilding per environment means you don't ship
  what you tested (Principle 1); storage is deduplicated across every workload's shared
  base layers in one registry but duplicated across N; and every registry-level control —
  scanning, retention, network posture, RBAC — must be maintained N times. Environment
  separation is real, but it is an *authorization* problem (who may pull what, which
  digests are deployed where), and RBAC plus digest-pinned deploy manifests solve it
  without forking the artifact store.
- The admin account is disabled. Anonymous pull is disabled.

### Image naming and tagging

Repositories inside the registry are namespaced by workload and component:

```text
<workload>/<component>:<tag>
```

- `billing/billing-api`, `billing/billing-web`, `code-review/code-review` — workload
  names as defined by [the naming standard](../naming/azure.md) (registry-internal paths
  are not Azure resource names, but they reuse the same workload vocabulary).
- **Release tags are SemVer**: `billing/billing-api:1.4.2`, produced by CI from the
  release process defined in [Releases](../github/releases.md). CI may additionally tag
  the git SHA (`:sha-3f2a91c`) for traceability.
- **Tags are never reused or moved.** `1.4.2` points at one digest forever. Enable the
  registry's tag-immutability/locking features where available; treat a moved tag as an
  incident either way.
- **No `latest` in deployments.** `latest` is fine as a human convenience for local
  pulls; it never appears in a Terraform configuration, Container App spec, or deploy
  workflow, because it makes "what is running?" unanswerable and rollbacks a guess.
- **Prod deploys pin the digest.** The deploy manifest references
  `acrplatformsharedcc01.azurecr.io/billing/billing-api@sha256:…` — the tag got the
  digest *found*; the digest is what gets *deployed*. Promotion dev → stg → prod carries
  the digest forward unchanged (see [Environments](environments.md)).

### Access control

All grants are Azure RBAC on the registry, per [Identity](identity.md):

| Principal | Role | Purpose |
|-----------|------|---------|
| CI deployment identities (`mi-github-billing-prod-cc-01` pattern), via OIDC | `AcrPush` | Build-and-push from GitHub Actions. Push rights only for the pipelines that build images — see [GitHub Actions](../github/github-actions.md). |
| Workload managed identities (`mi-billing-prod-cc-01` pattern) | `AcrPull` | Runtime image pulls by Container Apps / App Service. Pull only — a workload never pushes. |
| Platform team group | `AcrPush` + registry management via PIM | Operations, emergency retags, retention configuration. |
| Developer groups | `AcrPull` | Local pulls of shared images. |

No client secrets, no admin credentials, no docker-login passwords in GitHub secrets —
CI authenticates with Workload Identity Federation and `az acr login` off that token.

### Retention and lifecycle

- **Untagged manifests are deleted automatically** after 30 days via the registry
  retention policy. Every digest-only leftover from a superseded tag or an interrupted
  push is storage with no consumer; 30 days comfortably outlives any in-flight rollback
  window that would reference an old digest from a deploy manifest (digests referenced
  by a *tag* are unaffected).
- Old release tags are kept — they are the rollback and audit trail, and deduplicated
  layers make them cheap. If a workload's tag count becomes a real cost, prune by policy
  (e.g. keep the last N releases) as a deliberate, per-repository decision.
- Images for retired workloads are deleted when the workload retires — the registry
  repository is part of the workload's decommissioning checklist, since it lives outside
  the workload's RG lifecycle.

### Network posture

Per [Networking](networking.md): the registry has a **private endpoint** in stg/prod
spokes with `privatelink.azurecr.io` resolved from the central private DNS zones, and
public network access disabled once all consumers are private. Dev consumers and CI
runners that need public access are handled through the registry firewall/trusted
services as the documented dev-tier exception.

## Examples

CI push (from `deploy.yml`, authenticated via OIDC — see
[GitHub Actions](../github/github-actions.md)):

```bash
az acr login --name acrplatformsharedcc01
docker build -t acrplatformsharedcc01.azurecr.io/billing/billing-api:1.4.2 .
docker push acrplatformsharedcc01.azurecr.io/billing/billing-api:1.4.2

# capture the digest for the deploy manifest
az acr repository show \
  --name acrplatformsharedcc01 \
  --image billing/billing-api:1.4.2 \
  --query digest -o tsv
```

Prod deploy pins the digest; the environment never influences the artifact:

```hcl
# infra/environments/prod/terraform.tfvars
billing_api_image = "acrplatformsharedcc01.azurecr.io/billing/billing-api@sha256:9f8e7d6c..."
```

Workload pull identity wiring:

```hcl
resource "azurerm_role_assignment" "acr_pull" {
  scope                = data.azurerm_container_registry.platform.id
  role_definition_name = "AcrPull"
  principal_id         = azurerm_user_assigned_identity.billing.principal_id  # mi-billing-prod-cc-01
}
```

## Anti-patterns

| Anti-pattern | Why it fails |
|--------------|--------------|
| A registry per environment | Promotion becomes rebuild-or-copy; you no longer ship what you tested. Environment separation belongs in RBAC and digest pinning. |
| Rebuilding the image for prod "from the same tag" | Same source ≠ same artifact. Base images and dependencies moved; the tested binary is gone. |
| Deploying `:latest` (or any mutable tag) in any environment manifest | "What is running?" becomes unanswerable; a push silently changes prod on next restart; rollback has no target. |
| Reusing or moving a release tag | Destroys the audit trail and can silently alter what a pinned-by-tag consumer gets. Tags are immutable. |
| Enabling the admin account or storing registry passwords in CI secrets | A shared, unattributable, rotatable-in-theory credential. OIDC + RBAC exists precisely to delete this. |
| Granting workloads `AcrPush` | A compromised runtime could poison the image supply chain. Runtimes pull; only CI pushes. |
| Basic/Standard SKU for the shared registry | No private link — the registry becomes the one PaaS data service exempt from the private networking standard. |

## Tradeoffs

- **One registry is a single point of failure and a shared blast radius.** A registry
  outage stalls every deploy (though running workloads keep running on pulled images),
  and a supply-chain compromise of the registry touches everything. We accept the
  concentration because Premium's zone redundancy and optional geo-replication address
  availability, and because one perfectly-watched registry is safer in practice than
  three half-watched ones.
- **Premium costs more than Basic.** Roughly an order of magnitude over Basic per month.
  Private link alone justifies it for prod use; scanning surface and throughput make it
  a bargain against multi-registry operational cost.
- **Digest pinning adds pipeline plumbing.** The deploy flow must capture and thread the
  digest rather than just reusing the tag string. We accept the plumbing because it is
  the mechanism that makes "we ship what we tested" literally true rather than
  aspirational.
- **Shared registry means shared naming discipline.** Workload namespacing only works if
  every team follows it; the platform team reviews new top-level repository namespaces.
  A small gate, deliberately kept.

## Related Standards

- [GitHub Actions](../github/github-actions.md) — CI push via OIDC, promotion workflow
- [Networking](networking.md) — private endpoint and DNS for the registry
- [Identity](identity.md) — AcrPush/AcrPull role model, no client secrets
- [Environments](environments.md) — the promotion flow the registry serves
- [Releases](../github/releases.md) — SemVer versioning that release tags mirror
