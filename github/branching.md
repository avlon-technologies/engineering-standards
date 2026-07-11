# Branching

**Status:** Active
**Applies to:** All repositories in the `avlon-technologies` organization.

---

## Purpose

Branching strategy determines how fast changes flow from an engineer's editor to
production, and how much of the team's time is spent merging instead of building. Avlon
uses **trunk-based development**: one long-lived branch (`main`), short-lived work
branches, and nothing else.

This standard exists because the alternative — GitFlow, environment branches, long-lived
feature branches — has a predictable failure mode: branches that live for weeks
accumulate divergence, merges become multi-day archaeology projects, and "what is
actually deployed?" stops having a simple answer. Merge pain is a function of branch
lifetime; the strategy that minimizes lifetime minimizes pain.

## Guiding Principles

1. **`main` is always releasable.** Every commit on `main` has passed CI and review.
   Releasing is choosing a commit, not stabilizing a branch.
2. **Branch lifetime is the enemy.** The cost of integrating a branch grows superlinearly
   with its age. Days, not weeks — if a branch is a week old, the work was scoped too
   large.
3. **Environments are pipeline stages, not branches.** The same commit is promoted
   through `dev` → `stg` → `prod` by [GitHub Actions](github-actions.md). Code
   never "lives in" an environment.
4. **One process for everything.** Hotfixes use the same flow as features — a special
   emergency path is the least-tested path in the system, exercised only under the worst
   conditions.
5. **Branches are disposable.** A branch is a workspace for producing one squash commit.
   Once merged, it has no further value and is deleted automatically.

## Standard

### The only long-lived branch is `main`

`main` is protected by ruleset (see
[Repository Structure](repository-structure.md)): PR required, approvals, green
status checks, linear history, no force pushes. There is no `develop`, no `release/*`,
no `staging`, no environment branches of any kind.

### Work branches

All work happens on short-lived branches cut from the current tip of `main`:

```text
<type>/<issue>-<slug>
```

with types `feature | fix | chore | docs | hotfix` — for example
`feature/142-invoice-export`. The full naming convention, including the type
definitions, is canonical in [GitHub Naming](../naming/github.md); do not invent
types beyond that set.

Rules:

- **Cut from `main`, merge to `main`.** Branches are never cut from or merged into
  other work branches. Stacked work is sequenced as sequential PRs.
- **Target lifetime: merged within days.** If the work cannot merge within a few days,
  split it — merge the refactoring first, put incomplete features behind a flag, land
  the schema change before the code that uses it. Small increments on `main` beat a
  big-bang branch every time.
- **Stay current.** If `main` moves while your branch is open, bring the branch up to
  date by rebasing onto `main` or merging `main` in — either is acceptable, since squash
  merge erases the branch's internal history anyway. What is not acceptable is reviewing
  and merging a branch that is days behind `main`: the CI results no longer describe
  what will actually land.
- **Delete on merge.** Repositories enable *automatically delete head branches*. A
  merged branch left behind is noise that makes `git branch -r` useless.

### Why no environment branches

Environment branches (`dev`, `staging`, `production` branches with promotion by merging)
look orderly and fail in practice, for two structural reasons:

- **Drift.** The moment anyone commits directly to an environment branch — a "quick fix
  in staging" — the branches stop being the same code plus promotion. Now `production`
  contains changes `main` doesn't, deploys are no longer reproducible from `main`, and
  reconciling the branches is a manual diff exercise that someone always loses.
- **Merge hell.** Every promotion is a merge, every merge can conflict, and conflicts in
  a promotion merge must be resolved by someone who didn't write the code, under time
  pressure, on the branch that feeds production. The failure shows up at the worst
  possible moment by design.

Pipeline promotion has neither problem: one commit on `main` produces one artifact,
and [GitHub environments](github-actions.md) `dev`/`stg`/`prod` gate where that artifact
runs. The environment definitions themselves live in
[Azure Environments](../azure/environments.md).

### Hotfixes

A hotfix is the same flow, executed faster: branch `hotfix/981-null-invoice-crash` from
`main`, PR, review, squash merge, pipeline promotes to `prod`. The `hotfix/` type
signals reviewers to prioritize — first response in minutes, not hours — but nothing is
skipped: CI still runs, review still happens. If `main` has unreleased changes you do
not want to ship with the fix, that is a signal your deploy cadence is too slow, not a
justification for a release branch; fix forward and deploy `main`.

## Examples

The complete lifecycle of a change to `billing`:

```bash
git switch main && git pull
git switch -c feature/142-invoice-export

# ... commit freely; these commits disappear at squash merge ...
git commit -m "wip: export skeleton"
git commit -m "csv writer + tests"

# main moved? stay current:
git fetch origin && git rebase origin/main

git push -u origin feature/142-invoice-export
gh pr create --title "feat: add invoice CSV export" --body-file .github/pull_request_template.md

# after approval: squash merge via GitHub; branch auto-deleted.
# The pipeline deploys the new main commit to dev, then promotes to stg and prod.
```

## Anti-patterns

| Anti-pattern | Why it fails |
|---------------|--------------|
| `develop` branch / GitFlow | Doubles every merge, delays integration, and makes "releasable" a branch state instead of a permanent property of `main` |
| Environment branches (`staging`, `production`) | Drift and promotion-merge conflicts — see above |
| Feature branches living for weeks | Integration cost compounds; review becomes unreviewable; CI results go stale |
| Branching off another work branch | Creates a merge dependency chain; the base branch's squash merge orphans the child |
| A separate expedited path for hotfixes that skips CI or review | The least-tested process gets exercised only during incidents |
| Keeping merged branches "for reference" | The squash commit on `main` is the reference; branch lists become unreadable |

## Tradeoffs

- **Large features must be decomposed** into increments that are individually safe on
  `main`, sometimes behind feature flags. This is real design work. We accept it because
  the decomposition improves the design and every increment gets a real review.
- **`main`-only means discipline about incomplete work.** Feature flags add code paths
  that must eventually be removed. We accept the flag-cleanup chore over the merge-hell
  alternative.
- **Rebasing rewrites branch history**, which can confuse collaborators sharing a
  branch. Since branches are short-lived and single-author by default, this rarely
  bites; pairs sharing a branch should prefer merge-from-main.

## Related Standards

- [GitHub Naming](../naming/github.md) — branch naming: types, issue number, slug
- [Pull Requests](pull-requests.md) — the merge gate every branch passes through
- [GitHub Actions](github-actions.md) — pipeline promotion across environments
- [Azure Environments](../azure/environments.md) — the dev/stg/prod environment definitions
- [Releases](releases.md) — how releasable `main` becomes released versions
