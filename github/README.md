# GitHub Standards

How Avlon Technologies uses GitHub: repositories, branches, pull requests, reviews,
CI/CD, labels, and releases.

## Philosophy

Avlon practices **trunk-based development**. `main` is the only long-lived branch and it
is **always releasable** — every commit on `main` has passed the same checks a release
would, so shipping is a decision, not a project. Short-lived branches, small pull
requests, and squash merges keep history linear and every change revertible as a single
unit.

We are **automation-first**: anything a machine can check — formatting, linting, tests,
security scanning, deployment — is checked by a machine, on every pull request, before a
human looks at the code. This is not about distrust of engineers; it is about spending
scarce human attention where it matters. Which leads to the third principle: **reviews
are a quality gate, not gatekeeping**. Reviewers verify correctness, security, and
operability — they do not relitigate style (linters own style) and they do not act as
approval theater. A review that only says "LGTM" on a red build has failed twice.

Deployment environments (`dev`, `stg`, `prod`) are reached by **pipeline promotion**,
never by branches. There is no `develop`, no `release/*`, no environment branches — the
same artifact built from `main` moves through environments gated by GitHub environment
protection rules.

## Contents

| Document | Covers |
|----------|--------|
| [Repository Structure](repository-structure.md) | What every repository must contain: README, CODEOWNERS, branch protection rulesets, visibility policy, archiving |
| [Branching](branching.md) | Trunk-based development: short-lived branches off `main`, no GitFlow, no environment branches |
| [Pull Requests](pull-requests.md) | Small focused PRs, Conventional Commit titles, squash-merge-only, description template |
| [Code Reviews](code-reviews.md) | Review culture: what to check, what not to nitpick, response times, resolving disagreement |
| [GitHub Actions](github-actions.md) | CI/CD workflows, OIDC authentication to Azure, action pinning, least-privilege permissions |
| [Labels](labels.md) | The standard label taxonomy: prefix groups, colors, and the automation that consumes them |
| [Releases](releases.md) | SemVer tags, GitHub Releases, generated notes, Terraform module versioning |

## How these standards fit together

The documents form one pipeline. [Repository structure](repository-structure.md) sets up
the guardrails (branch protection, CODEOWNERS); [branching](branching.md) defines how
work reaches `main`; [pull requests](pull-requests.md) and
[code reviews](code-reviews.md) define the quality gate at the merge boundary;
[GitHub Actions](github-actions.md) automates verification and deployment;
[labels](labels.md) make triage uniform across every repository; and
[releases](releases.md) turn merged history into versioned, consumable artifacts. The
Conventional Commit PR title required by the pull request standard is what makes
generated release notes possible — the standards depend on each other deliberately.

Naming conventions for repositories, branches, workflow files, GitHub environments, and
labels are defined in [GitHub Naming](../naming/github.md) — the documents here
reference those conventions rather than restating them.
