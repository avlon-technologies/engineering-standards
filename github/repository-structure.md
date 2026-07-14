# Repository Structure

**Status:** Active
**Applies to:** Every repository in the `avlon-technologies` GitHub organization.

---

## Purpose

A repository is the unit of ownership, automation, and access control at Avlon. When
every repository has the same skeleton — the same files in the same places, the same
protections on `main`, the same review routing — engineers move between repositories
without relearning anything, and automation (CI templates, label provisioning, security
scanning) works everywhere without per-repo special cases.

Without this standard, repositories drift: one has no README, another has branch
protection "temporarily" disabled, a third routes reviews to an engineer who left last
year. Each gap is invisible until it causes an incident — an unreviewed force push, a
production deploy from an unprotected branch, a public repo that should have been
private.

## Guiding Principles

1. **Every repository is self-describing.** A newcomer must be able to answer *what is
   this, why does it exist, and how do I run it* from the README alone. Tribal knowledge
   does not scale and does not survive attrition.
2. **Protection is configuration, not discipline.** We do not rely on engineers
   remembering not to force-push to `main`. Rulesets make the unsafe action impossible,
   which is cheaper than making it forbidden.
3. **Teams own code; individuals do not.** Ownership assigned to a person breaks the day
   that person is on vacation, changes teams, or leaves. Ownership assigned to a team is
   durable.
4. **Application and infrastructure live together.** The code and the Terraform that
   deploys it share a lifecycle, a review process, and a pull request. Splitting them
   into separate repositories guarantees they drift apart.
5. **Private by default, public deliberately.** Publishing is a one-way door — secrets,
   internal hostnames, and client references leaked once are leaked forever.

## Standard

### Required contents

Every repository contains, at minimum:

```text
billing/
├── README.md               # what / why / how to run
├── LICENSE                 # public repos only
├── .gitignore              # language/tool appropriate
├── .editorconfig           # whitespace and encoding baseline
├── .github/
│   ├── CODEOWNERS
│   ├── pull_request_template.md
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
├── infra/                  # Terraform for this workload
└── src/                    # (or the ecosystem's conventional layout)
```

- **README.md** answers three questions in order: *what* the repository is, *why* it
  exists (the problem it solves, one paragraph), and *how to run it* locally —
  prerequisites, setup, test command. Anything a new engineer needs on day one belongs
  here or is linked from here.
- **LICENSE** is required for public repositories and prohibited from being an
  afterthought — choose it before publishing, not after.
- **`.gitignore`** and **`.editorconfig`** exist so that generated files and whitespace
  churn never reach review.
- **`.github/workflows/`** carries at least `ci.yml`; deployable workloads also carry
  `deploy.yml` (see [GitHub Actions](github-actions.md)).
- **`infra/`** holds the workload's Terraform, laid out per
  [Terraform Repository Layout](../terraform/repository-layout.md). An application
  repository without its infrastructure is half a repository.

Repository names follow [GitHub Naming](../naming/github.md) — lowercase
kebab-case, `billing`, `billing-api`, `terraform-azurerm-container-app`.

### Default branch

The default branch is `main`, always. No repository uses `master`, `trunk`, or anything
else as its default. Automation across the organization (deploy triggers, release
tooling, ruleset templates) assumes `main`; a divergent default branch silently breaks
all of it.

GitFlow repositories (those with staged releases or version-pinned consumers — see
[Branching](branching.md)) have a **second long-lived branch, `develop`**, as their
integration branch. `develop` is not the default and is never a substitute for `main`; it
is an additional protected branch that exists only in the GitFlow track. GitHub Flow
repositories (continuously deployed, nothing pinned) have `main` as their only long-lived
branch.

### CODEOWNERS

`.github/CODEOWNERS` exists in every repository and assigns ownership to **teams,
never individuals**:

```text
# .github/CODEOWNERS
*           @avlon-technologies/billing-team
/infra/     @avlon-technologies/platform-team
/.github/   @avlon-technologies/platform-team
```

Teams are durable: membership changes without editing every CODEOWNERS file in the
organization, review requests never land on someone who left, and vacation coverage is
automatic because GitHub requests review from the team, not a person. An individual in
CODEOWNERS is a single point of failure with a calendar.

### Branch protection via rulesets

`main` is protected by an organization-level **ruleset** (not legacy per-repo branch
protection — rulesets are centrally managed and auditable) that enforces:

| Rule | Setting |
|------|---------|
| Pull request required | Yes — no direct pushes, no exceptions for admins |
| Approvals required | 1 minimum; **2 for platform and Terraform-module repositories** |
| CODEOWNERS review required | Yes |
| Required status checks | `CI` must pass (see [GitHub Actions](github-actions.md)) |
| Linear history | Required **for GitHub Flow repositories** (pairs with squash-merge-only); **not required for GitFlow repositories**, whose `release`/`hotfix` merge commits are the record |
| Force pushes | Blocked |
| Branch deletion | Blocked for `main` |

In **GitFlow repositories**, the same ruleset also protects **`develop`** — PR required,
CI green, CODEOWNERS review — because `develop` is a long-lived integration branch that
features merge into directly. Linear history is not enforced on either `main` or
`develop` in GitFlow repos: the `--no-ff` merge commits that tie `release`/`hotfix`
branches to `main` and back to `develop` are deliberate history, not clutter (see
[Branching](branching.md)). GitHub Flow repositories protect only `main` and keep linear
history on, since every merge there is a squash.

Platform and Terraform-module repositories require two approvals because their blast
radius is every consumer: a bad merge to `terraform-azurerm-container-app` breaks every
workload that upgrades to it.

### Visibility

Repositories are **private by default**. A repository goes public only by deliberate
decision: it has a LICENSE, its history has been reviewed for secrets and internal
references, and the owning team accepts that it is now a permanent publication. "It
might be useful to someone" is not a reason; "we intend to maintain this in the open"
is.

### Archiving

Dead repositories are **archived, never deleted**. Archiving makes a repository
read-only while preserving history, issues, and links — deletion destroys audit trail,
breaks every cross-reference, and erases context that incident reviews and audits
depend on. Before archiving, update the README's first line to say what replaced it.

## Examples

A compliant new-workload checklist for the `billing-web` front-end (GitHub Flow):

```text
[x] Repo avlon-technologies/billing-web created private, default branch main
[x] README.md: what/why/how-to-run written before first feature PR
[x] .github/CODEOWNERS -> @avlon-technologies/billing-team, infra/ -> @avlon-technologies/platform-team
[x] Track chosen: GitHub Flow (front-end) — main only, squash-merge, linear history
[x] Organization ruleset applies: PR + 1 approval + CI check + linear history
[x] infra/ scaffolded per ../terraform/repository-layout.md
[x] Labels provisioned by automation (see labels.md)
```

A `billing-api` service would instead check *Track chosen: GitFlow* — adding a protected
`develop` branch, merge commits for `release`/`hotfix`, and linear history **off** (see
[Branching](branching.md)).

## Anti-patterns

| Anti-pattern | Why it fails |
|---------------|--------------|
| CODEOWNERS lists `@mike-avlon` | Reviews stall on one person's availability; file goes stale when they move on |
| Admins bypass branch protection "just this once" | The exception becomes the norm; the one unreviewed push is the one that breaks prod |
| Separate `billing-infra` repository | App and infra changes that must ship together get split across two PRs that merge at different times |
| Deleting a retired repository | History, issues, and every inbound link destroyed; archive instead |
| Making a repo public to share with one external party | Publication is permanent; use collaborator access instead |
| Empty or auto-generated README | The repository is unusable to anyone outside the founding team |

## Tradeoffs

- **Rulesets slow down trivial changes.** Even a one-character fix needs a PR and an
  approval. We accept this: the cost is minutes, and a bypass mechanism that exists
  will be used exactly when it shouldn't be.
- **Two approvals on platform repos add latency** to module releases. We accept this
  because module defects multiply across every consumer.
- **Archiving accumulates clutter** in the organization's repository list. We accept
  this and rely on the archived filter — clutter is cheaper than lost history.

## Related Standards

- [GitHub Naming](../naming/github.md) — repository and branch naming
- [Branching](branching.md) — how work reaches `main`
- [GitHub Actions](github-actions.md) — the required status checks
- [Terraform Repository Layout](../terraform/repository-layout.md) — the `infra/` layout
- [Labels](labels.md) — labels provisioned into every repository
