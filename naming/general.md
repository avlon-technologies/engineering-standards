# General Naming Principles

**Status:** Active
**Applies to:** Every name an Avlon engineer chooses — Azure resources, GitHub repositories and branches, Terraform identifiers, state keys, labels, and anything else that gets a name.

---

## Purpose

Every naming decision at Avlon derives from a small set of shared principles. Without them, each
domain invents its own dialect: the Azure resource says `prod`, the pipeline says `production`,
the branch says `prd`, and the automation that was supposed to connect them breaks silently.
Names are load-bearing metadata — cost reports, RBAC scopes, deployment gates, and incident
triage all key off them — and most names (Azure resources especially) cannot be changed after
creation without destroying the thing they name.

This document defines the principles and the **shared vocabulary** used by the domain-specific
standards ([Azure](azure.md), [GitHub](github.md), [Terraform](terraform.md)). Those documents
apply these rules; they never contradict them.

## Guiding Principles

1. **Names are deterministic.** Given the same inputs — workload, environment, region, instance —
   every engineer and every pipeline must produce the identical name, with no lookup and no
   judgement call. If a name cannot be derived mechanically, it is wrong. Determinism is what
   lets Terraform predict names, runbooks construct DR resource names from primary ones, and
   reviewers spot a misnamed resource in a plan diff.

2. **Names describe what a thing is, not who made it or when.** People change teams, dates rot,
   and versions belong in tags and releases. A name containing any of these is stale the day
   after it is created.

3. **One string, everywhere.** When the same concept appears in multiple systems — an environment
   in an Azure name, a GitHub environment, a Terraform variable — the strings must be
   byte-identical. Automation compares strings, not intent. `stg` and `staging` are two
   different environments as far as a deployment gate is concerned.

4. **Optimize for the reader under pressure.** The most important consumer of a name is an
   engineer scanning an alert at 2 a.m. Readability beats brevity except where a hard length
   limit forces abbreviation — and then the abbreviation is standardized, not improvised.

5. **Budget for the tightest constraint.** Azure storage accounts allow 24 characters; that
   single limit propagates back through the whole system and caps workload names at 12
   characters. It is better to accept one global constraint than to maintain per-resource
   exceptions.

## Standard

### Case and separators

**Lowercase kebab-case is the house style.** Lowercase words separated by hyphens: `code-review`,
`billing-api`, `engineering-standards`. This applies to Azure resource names, GitHub repository
names, workflow files, labels, tag keys, and state containers.

The single sanctioned exception is **snake_case where the tool demands it**: Terraform
identifiers (resource names, variables, outputs, locals) must be valid HCL identifiers, which
cannot contain hyphens — so `billing-api` the workload becomes `variable "workload"` with value
`"billing-api"`, and the resource block is `azurerm_linux_web_app.api`. Never use snake_case,
camelCase, or PascalCase anywhere hyphens are legal. Uppercase is never legal: several Azure
resource types forbid it, and allowing mixed case anywhere creates two ways to write every name.

### Shared vocabulary

These five terms appear throughout the naming standards and mean exactly this:

| Term | Definition | Examples |
|------|------------|----------|
| **workload** | A deployable unit of business function with its own lifecycle, state file, and resource groups. The stable, human-meaningful name everything else derives from. | `code-review`, `billing`, `platform-tfstate` |
| **component** | A part of a multi-part workload, expressed as a suffix on the workload name. | `billing-api`, `billing-web` |
| **environment** | The deployment tier, always one of the four codes below. | `dev`, `stg`, `prod`, `shared` |
| **region** | Short code for an Azure region, defined in [Azure Naming](azure.md#region-codes). | `cc`, `ce` |
| **instance** | Two-digit zero-padded ordinal disambiguating otherwise-identical resources. Always `01` unless a second instance actually exists. | `01`, `02` |

Use the term **workload**, not "application" or "project". "Application" suggests only
user-facing software and excludes platform capabilities like `platform-network`; "project"
suggests something that ends. Workloads are what we deploy, monitor, and bill.

### Environment codes

The complete environment set is `dev`, `stg`, `prod`, and `shared` — defined fully in
[Azure Environments](../azure/environments.md). Exactly these codes and no others: not `test`,
`uat`, `staging`, `stage`, `production`, `prd`, or `dv`.

Why exactly these? Three deployable tiers is the smallest set that separates "may be broken"
(`dev`) from "release validation against production-like config" (`stg`) from "live" (`prod`);
every additional tier doubles infrastructure cost and halves the traffic that exercises each
copy. The codes are three or four characters because they appear inside names with hard length
budgets, and they are fixed strings because automation — GitHub environment protection rules,
Terraform validation, Azure Policy, alert routing — keys off exact string equality. A single
variant spelling breaks filters and budgets silently, which is the worst way to break.

`shared` is not a fourth deployment tier. It marks **platform resources that serve all
environments** — Terraform state storage, the container registry, hub networking. A workload
resource is never `shared`; if a workload resource seems to need it, that resource belongs to a
platform capability instead.

### Workload names

Workload names are lowercase kebab-case, start with a letter, and are **at most 12 characters**
including hyphens. The limit is not aesthetic: an Azure storage account name has 24 characters
total and no hyphens, and the pattern `st<workload><env><region><instance>` spends 2 on the
prefix, up to 4 on environment, up to 4 on region, and 2 on the instance — leaving roughly 12
for the workload. One global limit, enforced once in Terraform validation, beats discovering the
problem at `terraform apply` time in the one environment with the longest environment code.

Choose the name once, at workload inception, and treat it as immutable — it will be embedded in
resource names, repository names, state keys, and dashboards.

### What never goes in a name

- **Personal names.** `sql-mike-test` is unownable the day Mike changes projects. Ownership is a
  tag (`owner: platform-team`), and it names a team, not a person.
- **The company name.** `avlon` adds no information inside Avlon's own tenant and organization,
  wastes scarce characters, and becomes wrong if a workload is transferred to a client. The one
  exception is namespaces that are globally shared by design, such as the GitHub organization
  `avlon` itself.
- **Dates and versions.** Creation time lives in metadata; versions live in git tags and image
  tags. `billing-2026` and `billing-v2` both rot instantly and encourage clone-and-forget
  sprawl. If a rewrite truly needs to coexist with its predecessor, it is a different workload
  and deserves a real name.
- **Codenames.** `project-falcon` is opaque to everyone hired after the kickoff meeting.

### Abbreviate consistently or not at all

Prefer full words: `code-review` beats `cr`. Abbreviate only when a length limit forces it, and
then register the abbreviation in the relevant standard (resource-type prefixes and region codes
in [Azure Naming](azure.md)) so it is used identically everywhere. Two abbreviations for the same
thing — or one thing sometimes abbreviated and sometimes not — is worse than either choice made
consistently, because it breaks the determinism principle: nobody can predict the name.

## Examples

The same workload name flowing through every system, unchanged:

```text
billing                                  # the workload name, chosen once
├── github.com/avlon/billing             # repository
├── rg-billing-prod-cc-01                # Azure resource group
├── stbillingprodcc01                    # storage account (hyphens stripped, name survives)
├── kv-billing-prod-cc-01                # Key Vault
├── tfstate-prod / billing.tfstate       # Terraform state container + key
├── mi-github-billing-prod-cc-01         # deployment identity for CI/CD
└── workload = "billing"                 # tag and Terraform variable value
```

And the environment string crossing system boundaries byte-identically:

```text
GitHub environment:        prod
Terraform variable:        environment = "prod"
Azure resource name:       app-billing-api-prod-cc-01
State container:           tfstate-prod
Tag:                       environment = prod
```

## Anti-patterns

| Anti-pattern | Why it fails |
|--------------|--------------|
| `BillingAPI`, `billing_api` (outside HCL) | Mixed case and underscores create multiple spellings of one name; some Azure types reject them outright. |
| `staging`, `production`, `prd` | Not in the canonical set; automation comparing strings treats them as unknown environments. |
| `sql-mike-test`, `project-falcon` | Personal names and codenames are unownable and opaque. |
| `avlon-billing-prod` | Company name wastes characters and adds nothing inside our own tenant. |
| `billing-v2`, `billing-2026` | Versions and dates rot; they belong in tags and releases. |
| `customer-billing-platform` (20 chars) | Exceeds the 12-character workload budget; will not fit in a storage account name. |
| `cr` for `code-review` in one repo only | Inconsistent abbreviation breaks name derivability. |

## Tradeoffs

- **The 12-character workload limit forces terse names.** Some workloads would read better with
  longer names. We accept this because the alternative — per-resource-type truncation rules —
  makes names non-deterministic exactly where determinism matters most.
- **Fixed environment codes are inflexible.** A team wanting a `sandbox` or `perf` tier cannot
  improvise one. That friction is intentional: new tiers have org-wide cost and automation
  implications, so they are proposed against [Azure Environments](../azure/environments.md)
  first, not invented per project.
- **Kebab-case-except-Terraform means one context switch.** Engineers write `billing-api` in a
  resource name and `billing_api` never (values stay kebab-case; only HCL *identifiers* are
  snake_case). The rule is learnable in one sentence, which is cheaper than fighting HCL's
  grammar.

## Related Standards

- [Azure Resource Naming](azure.md) — the pattern, prefixes, and length budgets this vocabulary feeds
- [GitHub Naming](github.md) — repositories, branches, environments, labels
- [Terraform Naming](terraform.md) — identifiers, state keys, module names
- [Azure Environments](../azure/environments.md) — the full definition of `dev`/`stg`/`prod`/`shared`
- [Azure Tagging](../azure/tagging.md) — where volatile metadata (owner, cost center) lives instead of names
