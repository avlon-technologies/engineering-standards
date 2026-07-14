# Branching

**Status:** Active
**Applies to:** All repositories in the `avlon-technologies` organization.

---

## Purpose

Branching strategy determines how fast changes flow from an engineer's editor to
production, and how much of the team's time is spent merging instead of building. The
right strategy is not universal — it depends on **how a repository releases**: whether it
ships continuously or on a deliberate cadence, and whether anything downstream pins a
specific version of it. Avlon uses **two** strategies, chosen by those two facts:

- **Repositories with a deliberate release cadence, or whose versions are pinned by
  consumers, use [GitFlow](#gitflow--versioned-deliberately-released-repositories).** An
  explicit `develop` integration branch, `release/*` stabilization branches, and `main`
  as the record of what is in production. A release branch is where a set of changes is
  stabilized, validated in `stg`, and tagged before it ships — the staging ground a
  versioned release needs so consumers get a deliberate version to pin, not whatever
  happened to merge.
- **Repositories that deploy continuously with no pinned versions use
  [GitHub Flow with feature flags](#github-flow-with-feature-flags--continuously-deployed-repositories).**
  One long-lived branch (`main`), short-lived branches off it, continuous deploy, and
  incomplete work hidden behind flags. When there is one live version and nobody pins it,
  the fastest safe path to production is a short branch and a flag, not a release train.

The choice follows from **release cadence and version-pinning**, not from what language
or layer the code is in. The two typically line up with layer — a public or
multi-consumer API usually has a real release cadence and pinned versions (GitFlow),
and a web front-end usually has one continuously-shipped live version (GitHub Flow) — but
the *cadence and pinning* are what decide, not the "backend" or "front-end" label. A
backend service deployed ten times a day that nobody pins belongs on GitHub Flow; a
versioned SDK or a desktop/mobile app with several releases live at once belongs on
GitFlow regardless of having no web UI. Match the strategy to how the thing releases;
using one strategy for everything is how a team ends up fighting its own process.

## Guiding Principles

1. **Strategy follows how the repo releases, not what layer it is.** Release cadence and
   version-pinning decide the track; "backend" and "front-end" are only the usual
   correlates. A continuously-deployed service and a versioned library have different
   release physics even though both are "backend."
2. **`main` always tells the truth about production.** In both tracks, `main` reflects
   what is running (or about to run) in `prod` — never work in progress. In GitFlow only
   `release/*` and `hotfix/*` reach `main`; in GitHub Flow every merge to `main` is
   deployable.
3. **Branch lifetime is the enemy.** Whichever track, *feature* branches are short-lived
   — days, not weeks. Integration cost grows superlinearly with branch age; a week-old
   feature branch was scoped too large in either model.
4. **One track per repository, fixed at creation.** A repo is GitFlow or GitHub Flow and
   does not mix the two. The track is a property of what the repo ships, decided when it
   is created, not per-PR.
5. **Branches are disposable workspaces.** Feature, release, and hotfix branches exist to
   produce merges and are deleted once merged. The permanent record is `main` (both
   tracks) and `develop` (GitFlow) — everything else is scaffolding.

## Standard

### Choosing a track

The track is decided by **two questions about how the repository releases**, not by what
layer it sits in:

1. **Does anything downstream pin a specific version of this?** Consumers that reference
   `v2.3.0` — other services, client apps, external integrators, Terraform module callers
   — cannot absorb a breaking change the instant it merges.
2. **Does a release ever need to be held and stabilized before it ships?** A deliberate
   cadence, more than one released version live at once, or a soak in `stg` before
   promotion.

| If… | Track | Long-lived branches |
|-----|-------|---------------------|
| **Either** answer is yes — versions are pinned, or releases are staged | **GitFlow** | `main`, `develop` |
| **Both** are no — one live version, deployed continuously, nobody pins | **GitHub Flow + feature flags** | `main` |

Layer is a strong *hint*, not the rule: a multi-consumer or public API (`billing-api`)
usually answers yes and lands on GitFlow, and a web front-end (`billing-web`) usually
answers no and lands on GitHub Flow — which is why the repository name (see
[GitHub Naming](../naming/github.md)) is a good first guess at the track. But the
answers, not the name, decide. A backend service deployed continuously with no pinned
consumers is GitHub Flow despite being backend; a versioned SDK, a Terraform module, or a
desktop/mobile app with several releases live at once is GitFlow despite having no web UI.

A repository does not run both, and the track is fixed at creation. A workload whose parts
have different release answers is split into separate repositories — commonly a
continuously-deployed UI (`billing-web`, GitHub Flow) and a version-pinned API
(`billing-api`, GitFlow) — each on its own track; see
[Repository Structure](repository-structure.md) on splitting by lifecycle.

### GitFlow — versioned, deliberately released repositories

Two long-lived branches, three kinds of supporting branch:

- **`main`** mirrors production. Nothing is committed to it directly; only `release/*`
  and `hotfix/*` merge in, and **every merge to `main` is tagged** with a release version
  (see [Releases](releases.md)). Reading `main` tells you exactly what has shipped.
- **`develop`** is the integration branch — the bleeding edge where completed features
  accumulate between releases. `develop` is what deploys to the `dev` environment.
- **`feature/<issue>-<slug>`** is cut from `develop` and merged back to `develop`. This
  is where day-to-day work happens.
- **`release/<version>`** is cut from `develop` when a set of features is ready to
  stabilize — e.g. `release/1.4.0`. It deploys to `stg`, receives only stabilization
  fixes (no new features), and when validated merges to **`main`** (tagged) **and back to
  `develop`** so the fixes are not lost.
- **`hotfix/<version>`** is cut from `main` for an urgent production fix — e.g.
  `hotfix/1.4.1`. It merges to **`main`** (tagged) **and back to `develop`**.

The lifecycle of a change:

```text
feature/142-invoice-export ──▶ develop ──▶ release/1.4.0 ──▶ main (tag v1.4.0)
        (cut from develop)         │              │              │
                                   ◀──────────────┘  back-merge  │
                                   ◀─────────────────────────────┘  (hotfix back-merge)
```

**Merges within GitFlow:**

- **`feature` → `develop`: squash.** One feature becomes one reviewed commit on
  `develop`, keeping integration history readable.
- **`release` → `main`, `hotfix` → `main`, and both back-merges to `develop`: `--no-ff`
  merge commits.** The merge commit preserves the release (or hotfix) as a discoverable
  point in history and keeps `develop` and `main` reconciled. GitFlow repositories
  therefore **do not enforce linear history** — the merge topology *is* the record
  (see [Repository Structure](repository-structure.md)).

**Environments map to branches in GitFlow** — `develop` → `dev`, `release/*` → `stg`,
`main` → `prod`. This is the one place Avlon deliberately couples an environment to a
branch, and it is safe *because the coupling is bounded*: a `release/*` branch is
short-lived, never receives direct feature commits, and is mandatorily merged back to
`develop` when it ships. The failure mode of *permanent* environment branches — a
`staging` or `production` branch that quietly accumulates commits `main` never sees — is
prevented by the back-merge rule, not by luck. A release branch that is never merged
back, or one that lives for weeks accepting new features, has stopped being a release
branch and reintroduces exactly the drift GitFlow exists to avoid.

### GitHub Flow with feature flags — continuously deployed repositories

One long-lived branch. Everything else is short-lived and cut from it.

- **`main`** is the only long-lived branch and is **always deployable**. Every commit on
  `main` has passed CI and review, and is continuously promoted through
  `dev` → `stg` → `prod` as one artifact by the pipeline
  ([GitHub Actions](github-actions.md)) — environments are pipeline stages, not
  branches.
- **`feature/<issue>-<slug>`** is cut from the tip of `main`, and squash-merged back to
  `main`. That is the entire branch model.

**Feature flags — not branches — isolate incomplete work.** Because there is no
`develop` or release branch to park half-finished features on, unfinished work merges to
`main` *dark*: it ships to production disabled behind a flag and is turned on by
configuration when it is ready, decoupling *deploy* (continuous) from *release* (a flag
flip). This is what makes GitHub Flow safe without a release train — the flag, not a
long-lived branch, holds back what isn't done. Flags are code paths that must eventually
be removed; retiring a flag once its feature is fully launched is a required chore, not
optional (see Tradeoffs).

**Hotfixes are just fast changes.** Branch `fix/<issue>-<slug>` from `main`, PR, review,
squash merge, and the pipeline promotes to `prod`. There is no separate emergency path —
fix forward and deploy `main`.

### Work-branch rules (both tracks)

- **Cut from the track's base, merge back to it.** GitFlow features branch from and merge
  to `develop`; GitHub Flow features branch from and merge to `main`. Feature branches
  are never cut from other feature branches — stacked work is sequenced as sequential
  PRs.
- **Target lifetime: merged within days.** If a feature cannot merge within a few days,
  split it — land a refactor first, put an incomplete feature behind a flag (GitHub Flow)
  or defer it to the next `develop` cycle, ship the schema change before the code that
  uses it.
- **Stay current.** If the base branch moves while your branch is open, rebase onto it or
  merge it in before requesting review — reviewing a branch that is days behind its base
  means the CI results no longer describe what will land.
- **Delete on merge.** Repositories enable *automatically delete head branches*. `main`
  and (in GitFlow) `develop` persist; every other branch is deleted the moment it merges.

Branch naming — the `<type>/<issue>-<slug>` pattern, the type set, and the version-named
`release/*` and `hotfix/*` branches — is canonical in
[GitHub Naming](../naming/github.md); do not invent types beyond that set.

## Examples

**GitFlow — a change to `billing-api`:**

```bash
# Feature work branches off develop:
git switch develop && git pull
git switch -c feature/142-invoice-export
# ... commit freely; squashed at merge ...
git push -u origin feature/142-invoice-export
gh pr create --base develop --title "feat: add invoice CSV export"
# squash-merged into develop; deploys to dev.

# When a release is ready, cut a release branch off develop:
git switch develop && git pull
git switch -c release/1.4.0
git push -u origin release/1.4.0          # deploys to stg for validation
# ... only stabilization fixes land here ...

# Ship: merge to main (tagged) AND back to develop:
gh pr create --base main --title "release: 1.4.0"   # --no-ff merge, tag v1.4.0 → deploys prod
git switch develop && git merge --no-ff release/1.4.0   # back-merge; delete release branch
```

**GitHub Flow — a change to `billing-web`:**

```bash
git switch main && git pull
git switch -c feature/318-export-button
# ... build the button behind a feature flag if it isn't launch-ready ...
git push -u origin feature/318-export-button
gh pr create --title "feat: add invoice export button"
# after approval: squash merge; branch auto-deleted.
# The pipeline deploys the new main commit to dev, then stg, then prod —
# the button ships dark and is enabled by flag when ready.
```

## Anti-patterns

| Anti-pattern | Why it fails |
|--------------|--------------|
| Mixing both tracks in one repository | The rules contradict (squash-only vs. merge commits, `develop` vs. no `develop`); nobody can tell how work reaches production. One repo, one track. |
| GitFlow on a repo that ships continuously and nobody pins | Release branches and back-merges add ceremony that buys nothing when there is no version to stabilize — pure overhead. Choosing it because the repo is "backend" is the classic mistake. |
| GitHub Flow on a repo whose versions are pinned or whose releases are staged | A breaking change reaches consumers the moment it merges, with no staging branch to hold it — they get surprised instead of a released version to pin. |
| **GitFlow:** committing features straight to `develop` | Skips review and CI-before-merge; `develop` stops being an integration of reviewed changes. |
| **GitFlow:** a `release/*` branch that lives for weeks taking new features | Stops being a stabilization branch and becomes a permanent environment branch — the drift GitFlow exists to prevent. |
| **GitFlow:** forgetting the back-merge to `develop` | Release/hotfix fixes exist on `main` but not `develop`; the next release silently regresses them. |
| **GitHub Flow:** long-lived feature branches instead of flags | Reintroduces merge-hell that flags were meant to eliminate; incomplete work belongs behind a flag on `main`, not on a branch. |
| **GitHub Flow:** environment branches (`staging`, `production`) | Drift and promotion-merge conflicts — environments are pipeline stages, promoted as one artifact. |
| Feature branch cut from another feature branch (either track) | Creates a merge dependency chain; the base branch's merge orphans the child. |
| Keeping merged branches "for reference" | The commit on `main`/`develop` is the reference; branch lists become unreadable. |

## Tradeoffs

- **GitFlow has more moving parts** — two long-lived branches, release branches,
  mandatory back-merges. We accept the overhead only where a version is stabilized and
  pinned, because that is the one thing the machinery buys: a place to hold and validate a
  release before consumers depend on it. Where nothing is pinned and deploys are
  continuous, the cost is pure overhead — which is exactly the case the other track
  serves.
- **GitHub Flow demands feature-flag discipline.** Flags add code paths that must
  eventually be removed, and a flag left on forever is tech debt. We accept the
  flag-cleanup chore over the merge-hell alternative, and treat retiring a launched
  flag as part of finishing the feature.
- **The track isn't always obvious from the outside.** Because cadence and version-pinning
  decide it — not the repo's layer — a backend repo could be on either track. The base
  branch (`develop` vs. `main`) makes it concrete the moment you branch, and the repo's
  README should state its track so nobody has to infer it; the repository name is only a
  hint (see [Choosing a track](#choosing-a-track)).
- **Large features must still be decomposed** in both tracks — into increments safe on
  `develop`, or behind flags on `main`. This is real design work we accept because the
  decomposition improves the design and every increment gets a real review.

## Related Standards

- [GitHub Naming](../naming/github.md) — branch naming: types, issue numbers, slugs, and version-named `release/*` and `hotfix/*` branches
- [Repository Structure](repository-structure.md) — `main`/`develop` protection rulesets and the track-dependent merge/linear-history settings
- [Pull Requests](pull-requests.md) — the merge gate every branch passes through, and the track-dependent squash vs. merge-commit strategy
- [GitHub Actions](github-actions.md) — how each track deploys: pipeline promotion vs. branch-mapped environments
- [Releases](releases.md) — how `release/*` branches and continuous `main` become released versions
- [Azure Environments](../azure/environments.md) — the dev/stg/prod environment definitions
