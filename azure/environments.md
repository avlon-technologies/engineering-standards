# Environments

**Status:** Active
**Applies to:** All workloads and platform capabilities across all Avlon subscriptions. This is the canonical definition of Avlon's environments — every other standard that mentions `dev`, `stg`, `prod`, or `shared` defers to this document.

---

## Purpose

"Environment" is the most overloaded word in infrastructure. Without a fixed definition,
teams invent their own — a `test` here, a `uat` there, a per-developer sandbox that never
gets deleted — and every piece of automation that keys off environment names (Terraform
variables, pipeline stages, Azure Policy assignments, budgets, alert routing) fragments
with them. Worse, without a stated *purpose* for each environment, staging quietly stops
resembling production, and the one environment whose entire job is to catch problems
before production stops being able to.

This document fixes the set of environments, what each one guarantees, and how changes
move between them. It exists so that "it works in staging" is a meaningful sentence.

## Guiding Principles

1. **Environments are identical in shape, different in scale.** Every environment of a
   workload deploys the same Terraform configuration and the same topology. What varies
   between environments is captured entirely in per-environment variable values — SKU
   sizes, replica counts, feature flags — never in structurally different code. If dev
   has a resource prod doesn't (or vice versa), the environments have diverged and the
   promotion pipeline is testing a fiction.
2. **Staging exists to be believed.** The only reason to pay for a staging environment
   is that a green result there predicts a green result in production. That prediction
   is only as good as the parity: same SKU family, same topology, same private
   networking, same identity model. Every parity shortcut taken in staging is a class
   of production incident staging can no longer catch.
3. **Every environment has a real carrying cost.** An environment is not just compute
   spend — it is state files, DNS zones, RBAC assignments, monitoring noise, patching,
   and drift surface. Environments must earn their existence, which is why there are
   exactly three plus `shared`, and not five.
4. **Promotion is a pipeline concern, not a human one.** The same built artifact moves
   dev → stg → prod through GitHub Actions with environment protection gates. Nobody
   promotes by hand, and nothing is rebuilt along the way.
5. **Ephemeral beats permanent for individual needs.** Developers who need isolation get
   it from local runs and short-lived ephemeral instances inside `dev` — not from
   long-lived personal environments, which are drift farms with a name on them.

## Standard

### The environment set

These are the only environments. The codes are load-bearing — they appear in resource
names, tags, Terraform directories, state containers, and GitHub environment names —
so use these strings and no others (not `test`, `uat`, `staging`, `stage`,
`production`, `prd`, or `dv`).

| Environment | Code | Purpose |
|-------------|--------|---------|
| Development | `dev` | Day-to-day development and integration. May be unstable. |
| Staging | `stg` | Production-like pre-production. Release validation, UAT, performance tests. |
| Production | `prod` | Live workloads. |
| Shared | `shared` | ONLY for platform resources that serve all environments (Terraform state storage, container registry, hub networking). Never for workload resources. |

### What each environment guarantees

**`dev` — cheap and honest about it.** Dev exists for fast iteration and integration
testing. It runs the smallest viable SKUs, may be torn down and rebuilt at any time, and
carries no uptime expectation. It is allowed documented cost shortcuts — public
endpoints with firewall rules instead of private endpoints, no zone redundancy — per
[Networking](networking.md). What it is *not* allowed is structural divergence: the same
Terraform configuration must deploy it.

**`stg` — production parity, deliberately paid for.** Staging mirrors production:
same topology, private networking, private endpoints on data services, same SKU family
(and same SKU size where affordable — see Tradeoffs). It is where release validation,
UAT, and performance tests happen, because it is the only environment whose results
transfer to production. Staging data is production-*like*, never production data.

**`prod` — locked down.** Production changes arrive only through the pipeline. Human
access is just-in-time via PIM per [Identity](identity.md); networking is fully private
per [Networking](networking.md); deletion is protected by resource locks where the
platform supports them. If it isn't in Terraform, it doesn't exist in prod.

**`shared` — platform only.** `shared` is not a fourth workload environment. It marks
platform resources that, by their nature, serve every environment at once: the Terraform
state storage account `sttfstatesharedcc01`, the container registry
`acrplatformsharedcc01`, hub networking. A workload resource tagged or named `shared` is
a standards violation — workloads always live in exactly one of `dev`, `stg`, `prod`.

### Promotion flow

Changes move **dev → stg → prod**, always via GitHub Actions, never manually:

1. A pull request triggers CI: build, test, `terraform plan` against each environment.
2. Merge to `main` deploys to `dev` automatically.
3. The same artifact (container image by digest, same Terraform code at the same commit)
   is promoted to `stg`, gated by GitHub environment protection rules.
4. After validation in `stg`, the identical artifact is promoted to `prod`, gated by
   required reviewers on the `prod` GitHub environment.

Images are promoted, not rebuilt — see [Container Registry](container-registry.md).
GitHub environments are named `dev`, `stg`, `prod` to match, per the
[GitHub Actions standard](../github/github-actions.md).

### Configuration differences live in tfvars only

All per-environment differences are expressed as variable values in the environment's
Terraform directory (`infra/environments/<env>/terraform.tfvars`) — never as divergent
resource blocks, `count` tricks keyed on environment names scattered through modules, or
manual portal changes. The directory-per-environment layout and the reasoning behind it
are defined in [Terraform Environments](../terraform/environments.md).

### No per-developer long-lived environments

Individual developers do not get standing personal environments. Isolation needs are met
by local development, by ephemeral instances created and destroyed within `dev` (a
branch-scoped Container App, a temporary database), or by feature flags. Long-lived
personal environments accumulate drift, cost, and orphaned resources, and their results
convince nobody.

## Examples

The `billing` workload across the environment set:

```text
rg-billing-dev-cc-01        # dev:  smallest SKUs, public SQL endpoint + firewall rule
rg-billing-stg-cc-01        # stg:  prod topology, private endpoints, prod SKU family
rg-billing-prod-cc-01       # prod: locked down, PIM-gated human access
rg-billing-prod-ce-01       # prod DR pair in Canada East — prod is the only env with DR
```

Per-environment configuration expressed as data, not code:

```hcl
# infra/environments/stg/terraform.tfvars
environment       = "stg"
app_service_sku   = "P1v3"   # same family as prod's P2v3, one size down
sql_sku           = "GP_Gen5_2"
zone_redundant    = true
private_endpoints = true
```

```hcl
# infra/environments/dev/terraform.tfvars
environment       = "dev"
app_service_sku   = "B1"
sql_sku           = "GP_S_Gen5_1"  # serverless, auto-pause
zone_redundant    = false
private_endpoints = false          # documented dev tradeoff — see networking.md
```

## Anti-patterns

| Anti-pattern | Why it fails |
|--------------|--------------|
| Inventing environments (`uat`, `test`, `sandbox`, `demo`) per project | Every consumer of environment codes — names, tags, policies, pipelines, budgets — must now handle a bespoke set. The standard is three environments; needs that feel like a fourth are almost always a use of `stg` or an ephemeral instance in `dev`. |
| Long-lived per-developer environments | Permanent cost and drift surface for a temporary need. Use local runs or ephemeral instances in `dev`. |
| Manual promotion ("copy the settings to prod") | Untracked, unrepeatable, and guaranteed to diverge. Promotion is the pipeline's job. |
| Structural drift between environments (prod has a WAF, stg doesn't) | Staging can no longer predict production behavior — which was its only job. |
| Testing in prod because stg is "too different" | The symptom of eroded parity. Fix staging; don't normalize production experiments. |
| Production data in dev or stg | A compliance incident waiting for a place to happen. Use synthetic or masked data. |
| Tagging a workload resource `environment = shared` | `shared` is reserved for platform capabilities. Workload resources belong to exactly one environment. |

## Tradeoffs

- **Staging parity is expensive, and we pay it knowingly.** Running production-family
  SKUs and private networking in an environment with no users feels wasteful — until the
  first incident that staging would have caught. Where full SKU-size parity is genuinely
  unaffordable, we keep the same SKU *family* and topology and accept that pure
  performance results need scaling judgment; topology and behavior parity are
  non-negotiable because they are what catch correctness and connectivity failures.
- **Exactly three environments is a constraint some projects will chafe at.** A client
  demo or a long UAT cycle occasionally wants its own environment. We accept the
  friction because each additional environment multiplies drift surface, RBAC and
  networking configuration, and standing cost forever — the fixed set stays cheap to
  reason about precisely because it is fixed.
- **No personal environments slows the rare workflow that genuinely needs one.**
  Ephemeral instances take pipeline work to support. We accept that up-front investment
  over an estate of forgotten `dev-mike` resource groups.

## Related Standards

- [Terraform Environments](../terraform/environments.md) — directory-per-environment layout and tfvars mechanics
- [Networking](networking.md) — the private-by-default posture and the dev exception
- [Container Registry](container-registry.md) — image promotion across environments
- [Identity](identity.md) — PIM/JIT human access to prod
- [Azure Resource Naming](../naming/azure.md) — where environment codes appear in names
