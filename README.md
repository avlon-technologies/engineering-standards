# Avlon Technologies — Engineering Standards

This repository is the **single source of truth** for engineering practices at Avlon Technologies. It defines how we design, build, name, secure, and operate software and cloud infrastructure across every client engagement and internal project.

## Purpose

Consistency is a force multiplier. When every project follows the same conventions, engineers move between projects without relearning the basics, code reviews focus on substance instead of style, and operational tooling works everywhere. These standards exist to:

- **Remove decisions that don't need to be made twice.** Naming, structure, and tooling choices are made once, here, and reused everywhere.
- **Encode hard-won lessons.** Standards capture what we've learned from real projects so mistakes are not repeated.
- **Make quality the default.** Following the standard should produce a secure, operable, maintainable result without extra effort.

Standards in this repository are grounded in the [Microsoft Azure Well-Architected Framework](https://learn.microsoft.com/azure/well-architected/), the [Microsoft Cloud Adoption Framework](https://learn.microsoft.com/azure/cloud-adoption-framework/), GitHub best practices, and Terraform best practices — adapted and made opinionated for how Avlon works.

## How to Use This Repository

- **Standards are mandatory by default.** Deviations require a documented, justified reason (typically an ADR in the project repository).
- **Standards are living documents.** They evolve as platforms, tooling, and our experience evolve. Propose changes via pull request; discussion happens in review.
- **When a standard is silent, use judgment** — then propose an addition so the next engineer doesn't have to.

## Repository Structure

```
engineering-standards/
├── README.md                    # This document
├── azure/                       # Azure standards
│   └── resource-naming.md       # Azure resource naming standard
├── github/                      # (planned) Repos, branching, PRs, reviews
├── architecture/                # (planned) ADRs, reference architectures, design guidance
├── development/                 # (planned) Coding standards, testing, code review
├── security/                    # (planned) Identity, secrets, network, data protection
└── cicd/                        # (planned) Pipelines, environments, release management
```

### Current Standards

| Area | Standard | Status |
|------|----------|--------|
| Azure | [Resource Naming](azure/resource-naming.md) | Active |

### Planned Sections

- **Azure** — resource naming, tagging, landing zones, subscription design
- **GitHub** — repository conventions, branching strategy, pull requests, code review
- **Architecture** — architecture decision records, reference architectures, design reviews
- **Development** — language standards, testing strategy, dependency management
- **Security** — identity and access, secrets management, network security, data protection
- **CI/CD** — pipeline standards, environment promotion, release and rollback

## Engineering Principles

Every standard in this repository is measured against these principles. When a proposed standard conflicts with one, the principle wins.

1. **Convention over configuration.** Prefer one well-understood default over many configurable options. A convention followed everywhere beats a perfect solution used once.
2. **Infrastructure as Code.** All infrastructure is defined in code (Terraform), version-controlled, and deployed through pipelines. Manual portal changes are for break-glass scenarios only.
3. **Secure by default.** The easiest path must be the secure path: private by default, least privilege, managed identities over credentials, secrets in vaults — never in code.
4. **Automation first.** If a task will be done more than twice, automate it. Humans review and decide; machines execute and repeat.
5. **Simplicity.** Choose the simplest design that meets requirements. Complexity must earn its place — every abstraction, service, and moving part carries a permanent operational cost.
6. **Reusability.** Build modules, templates, and patterns intended for the next project, not just the current one. Solve categories of problems, not instances.
7. **Operational excellence.** Design for the people who run the system: observable by default, documented runbooks, deterministic deployments, and failure modes that are understood before they occur.

## Contributing

1. Open a pull request with the proposed change and its rationale.
2. Include the *why* — standards without rationale don't survive contact with real projects.
3. Changes are reviewed by the engineering leadership team and take effect on merge.

---

*Maintained by Avlon Technologies Engineering. Questions and proposals via pull request or issue.*
