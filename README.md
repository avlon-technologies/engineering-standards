# Avlon Technologies — Engineering Standards

The Avlon Engineering Standards define the recommended engineering practices for building secure, maintainable, and scalable software systems. They exist to promote consistency across projects while allowing teams to make informed architectural decisions when trade-offs are required. These standards are living documentation and evolve alongside our engineering practices.

This repository is the single source of truth for those practices: how we design, build, name, secure, and operate software and cloud infrastructure across every client engagement and internal project. It is written as an engineering handbook, not a collection of notes — every standard explains *why* it exists, shows concrete examples, names its anti-patterns, and is honest about its tradeoffs. These are opinionated guidelines grounded in engineering principles, not arbitrary rules.

## Purpose

Consistency is a force multiplier. When every project follows the same conventions, engineers move between projects without relearning the basics, code reviews focus on substance instead of style, and operational tooling works everywhere. These standards exist to:

- **Remove decisions that don't need to be made twice.** Naming, structure, and tooling choices are made once, here, and reused everywhere.
- **Encode hard-won lessons.** Standards capture what we've learned from real projects so mistakes are not repeated.
- **Make quality the default.** Following the standard should produce a secure, operable, maintainable result without extra effort.

Standards in this repository are grounded in the [Azure Well-Architected Framework](https://learn.microsoft.com/azure/well-architected/), the [Microsoft Cloud Adoption Framework](https://learn.microsoft.com/azure/cloud-adoption-framework/), [HashiCorp's Terraform recommendations](https://developer.hashicorp.com/terraform/language/style), and GitHub engineering practice — adapted and made opinionated for how Avlon works.

## Guiding Principles

Every standard in this repository is measured against these principles. When a proposed standard conflicts with one, the principle wins.

1. **Convention over configuration.** Prefer one well-understood default over many configurable options. A convention followed everywhere beats a perfect solution used once.
2. **Infrastructure as Code.** All infrastructure is defined in Terraform, version-controlled, and deployed through pipelines. Manual portal changes are for break-glass scenarios only.
3. **Secure by default.** The easiest path must be the secure path: private networking by default, least-privilege RBAC, managed identities over credentials, secrets in vaults — never in code.
4. **Automation first.** If a task will be done more than twice, automate it. Humans review and decide; machines execute and repeat.
5. **Simplicity.** Choose the simplest design that meets requirements. Complexity must earn its place — every abstraction, service, and moving part carries a permanent operational cost.
6. **Reusability.** Build modules, templates, and patterns intended for the next project, not just the current one. Solve categories of problems, not instances.
7. **Operational excellence.** Design for the people who run the system: observable by default, documented runbooks, deterministic deployments, and failure modes that are understood before they occur.

## Repository Organization

Standards are organized by discipline. Each directory has its own `README.md` explaining the discipline's philosophy and indexing its documents.

```
engineering-standards/
├── azure/          # Azure platform standards — subscriptions, resource groups,
│                   # tagging, identity, networking, container registry, environments
├── terraform/      # Terraform standards — repository layout, modules, environments,
│                   # remote state, backend bootstrap, providers, variables, outputs, security
├── github/         # GitHub standards — repository structure, branching, pull requests,
│                   # code reviews, GitHub Actions, labels, releases
└── naming/         # Naming conventions — the single canonical home for ALL naming:
                    # general principles, Azure resources, GitHub, Terraform
```

Two organizational rules keep the handbook coherent as it grows:

- **Naming lives in one place.** All naming conventions — Azure resources, repositories, branches, modules, variables — are defined in [`naming/`](naming/README.md). Discipline documents reference them rather than restating them, so a naming change is a one-file change.
- **Each document owns one topic.** Cross-cutting concepts (the environment model, the tag set, the naming pattern) are defined fully in exactly one document; every other document links to it.

### Where to Start

| If you are… | Start with |
|-------------|-----------|
| Creating a new Azure workload | [Resource Groups](azure/resource-groups.md) → [Azure Naming](naming/azure.md) → [Tagging](azure/tagging.md) |
| Writing Terraform | [Repository Layout](terraform/repository-layout.md) → [Remote State](terraform/remote-state.md) → [Modules](terraform/modules.md) |
| Setting up a new repository | [Repository Structure](github/repository-structure.md) → [GitHub Naming](naming/github.md) → [Branching](github/branching.md) |
| Setting up CI/CD | [GitHub Actions](github/github-actions.md) → [Terraform Security](terraform/security.md) → [Identity](azure/identity.md) |
| Naming anything | [Naming — General Principles](naming/general.md) |

## Scope — Phase 1

This first phase covers the foundations every project touches on day one:

| Area | Covers |
|------|--------|
| [Azure](azure/README.md) | Subscriptions, resource groups, tagging, identity (Entra ID, RBAC, managed identity), networking and private endpoints, container registry, environment strategy |
| [Terraform](terraform/README.md) | Repository layout, module design, environment structure, remote state, backend bootstrap, provider configuration, variables, outputs, security |
| [GitHub](github/README.md) | Repository structure, branching, pull requests, code reviews, GitHub Actions, labels, releases |
| [Naming](naming/README.md) | General principles, Azure resources, GitHub, Terraform |

## How to Use This Repository

- **Standards are the default, not a straitjacket.** Follow them unless there is a real reason not to. When a project's requirements genuinely conflict with a standard, make the informed trade-off — and document it (typically an ADR in the project repository) so the reasoning survives the decision.
- **Standards are living documents.** They evolve as platforms, tooling, and our experience evolve. Propose changes via pull request; discussion happens in review. If you deviate for the same reason twice, the standard is probably wrong — propose the change.
- **When a standard is silent, use judgment** — then propose an addition so the next engineer doesn't have to.

## Contributing

1. Open a pull request with the proposed change and its rationale.
2. Include the *why* — standards without rationale don't survive contact with real projects.
3. Follow the document template used throughout (Purpose, Guiding Principles, Standard, Examples, Anti-patterns, Tradeoffs, Related Standards). Every recommendation needs at least one concrete example.
4. Changes are reviewed by the engineering leadership team and take effect on merge.

New resource type prefixes, region codes, environment names, or label groups must be added to the relevant standard **before first use** — conventions only work when the registry is complete.

## Roadmap

Future phases will expand the handbook into:

- **Architecture** — architecture decision records, reference architectures, design reviews
- **.NET** — language standards, project structure, dependency management
- **APIs** — API design, versioning, documentation, contracts
- **Security** — secrets management, data protection, threat modeling, secure SDLC
- **Testing** — testing strategy, coverage expectations, test architecture
- **Observability** — logging, metrics, tracing, alerting, dashboards
- **CI/CD** — pipeline standards beyond GitHub Actions basics, environment promotion, release and rollback

The directory-per-discipline structure means each phase adds a directory without reorganizing what exists.

---

*Maintained by Avlon Technologies Engineering. Questions and proposals via pull request or issue.*
