# Naming Standards

Names are the cheapest infrastructure Avlon owns and the most expensive to change. A resource
name appears in the portal at 2 a.m., a repository name appears in every clone URL forever, and
a Terraform state key silently decides whether two deployments collide. This directory is the
**single canonical home for every naming convention at Avlon Technologies** — Azure resources,
GitHub repositories and branches, Terraform code and state. If a name is being chosen anywhere
in the company, the rule for it lives here.

Other sections of this handbook (`azure/`, `terraform/`, `github/`) *reference* these documents;
they never restate them. When a discipline doc shows an example name inline, that name conforms
to — and links back to — the standard defined here. This keeps every prefix table, environment
code, and pattern defined exactly once, so a change lands in one pull request instead of five
contradictory documents.

## Contents

| Document | Covers |
|----------|--------|
| [General Naming Principles](general.md) | Cross-cutting rules and shared vocabulary (workload, component, environment, region, instance) that every other naming standard builds on. |
| [Azure Resource Naming](azure.md) | The canonical pattern for every Azure resource: prefixes, region codes, exceptions, length budgets, Terraform name generation. |
| [GitHub Naming](github.md) | Repositories, branches, workflow files, GitHub environments, labels, and teams. |
| [Terraform Naming](terraform.md) | Module repos, resource and data blocks, variables, outputs, locals, state keys, and tfvars files. |

## How these standards fit together

[General Naming Principles](general.md) defines the vocabulary and the house rules — lowercase
kebab-case, deterministic names, the `dev`/`stg`/`prod`/`shared` environment codes — and the
other three documents apply them to their domain. The dependencies are deliberate: the Azure
standard constrains **workload names to 12 characters** because of Azure length limits, and that
constraint flows outward — a GitHub repository named `billing` and a Terraform state key
`billing.tfstate` use the *same* workload name as the resource group `rg-billing-prod-cc-01`.
Environment strings are shared even more strictly: GitHub environments, Terraform variables, and
Azure resource names all use the identical `dev`/`stg`/`prod` codes, because CI/CD automation
keys off string equality between them.

Read [General Naming Principles](general.md) first; it is short and everything else assumes it.
