# GitHub Standards

How Avlon Technologies uses GitHub: repositories, branches, pull requests, reviews,
CI/CD, labels, and releases.

## Philosophy

Avlon matches the **branching strategy to how a repository releases** — its cadence and
whether consumers pin its versions, not what layer it sits in. A repository whose
releases are staged or whose versions are pinned uses **GitFlow** — a `develop`
integration branch and `release/*` branches that stabilize a version before it ships, so
consumers get a deliberate version to pin instead of whatever just merged. A repository
that deploys continuously with one live version nobody pins uses **GitHub Flow with
feature flags** — one always-deployable `main`, short-lived branches, continuous deploy,
and incomplete work held behind flags rather than branches. Layer is only a hint (most
version-pinned APIs land on GitFlow, most web front-ends on GitHub Flow); the full
decision lives in [Branching](branching.md). One track per repository, chosen at
creation. What both tracks share: `main` always tells the truth about production, feature
branches are short-lived, and small pull requests keep every change reviewable and
revertible.

We are **automation-first**: anything a machine can check — formatting, linting, tests,
security scanning, deployment — is checked by a machine, on every pull request, before a
human looks at the code. This is not about distrust of engineers; it is about spending
scarce human attention where it matters. Which leads to the next principle: **reviews
are a quality gate, not gatekeeping**. Reviewers verify correctness, security, and
operability — they do not relitigate style (linters own style) and they do not act as
approval theater. A review that only says "LGTM" on a red build has failed twice.

How deployment environments (`dev`, `stg`, `prod`) are reached depends on the track. On
GitHub Flow repositories, they are reached by **pipeline promotion** — the same artifact
built from `main` moves through environments gated by GitHub environment protection
rules, with no environment branches. On GitFlow repositories, environments map to branches
by design — `develop` → `dev`, `release/*` → `stg`, `main` → `prod` — bounded by the
short life of release branches and the mandatory back-merge to `develop`.

## Contents

| Document | Covers |
|----------|--------|
| [Repository Structure](repository-structure.md) | What every repository must contain: README, CODEOWNERS, branch protection rulesets, visibility policy, archiving |
| [Branching](branching.md) | Two tracks by release model: GitFlow for staged/version-pinned repos, GitHub Flow + feature flags for continuously deployed ones |
| [Pull Requests](pull-requests.md) | Small focused PRs, Conventional Commit titles, squash-merge for features (merge commits for GitFlow releases), description template |
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
