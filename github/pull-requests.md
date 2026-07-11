# Pull Requests

**Status:** Active
**Applies to:** Every change to every repository in the `avlon-technologies` organization.

---

## Purpose

The pull request is Avlon's **unit of change**: the unit of review, of CI verification,
of merge, of revert, and — because we squash merge — of history. Everything that reaches
`main` arrives as exactly one PR that becomes exactly one commit.

This standard exists because PR quality determines review quality. A 2,000-line PR with
the title "updates" cannot be meaningfully reviewed, cannot be cleanly reverted, and
tells future readers of `git log` nothing. The rules below make PRs small, self-
describing, and mechanically consistent — which is what makes the rest of the toolchain
(generated release notes, clean reverts, readable history) work.

## Guiding Principles

1. **Small PRs get better reviews.** Review quality degrades sharply with size: past a
   few hundred lines, reviewers stop reasoning about the change and start skimming it.
   Ten small PRs get ten real reviews; one huge PR gets one rubber stamp.
2. **The PR title is permanent; the branch is not.** Squash merge makes the title the
   commit subject on `main` forever. Titles are written for `git log`, not for the
   author's convenience.
3. **Machines before humans.** CI must be green before a human is asked to look.
   Reviewer attention is the most expensive resource in the process; never spend it on
   a build that a robot already knows is broken.
4. **A PR explains itself.** Reviewers and future archaeologists should not need to ask
   the author what the change does, why, or how it was verified.
5. **No one merges their own unreviewed code.** Not seniors, not admins, not "trivial"
   changes. The rule survives because it has no exceptions.

## Standard

### Size

Keep PRs **small and focused**: one logical change per PR, with a soft guideline of
**~400 lines changed**. This is a guideline, not a gate — a 600-line mechanical rename
is fine; a 400-line PR mixing a refactor with a behavior change is not. When a piece of
work exceeds the guideline, split it as described in [Branching](branching.md):
refactor first, then behavior; schema first, then code.

The reason is empirical: defect-detection rates in review collapse beyond roughly
400 lines because reviewers lose the working memory needed to trace the change's
consequences. Beyond that point you are not getting a review, you are getting a scroll.

### Title: Conventional Commit format

The PR title follows [Conventional Commits](https://www.conventionalcommits.org/):

```text
<type>: <imperative summary>
```

with types `feat: | fix: | chore: | docs: | refactor: | test: | ci:`. Examples:

```text
feat: add invoice CSV export
fix: handle null due date in billing reminders
ci: pin third-party actions to commit SHAs
```

This matters because **squash merge makes the PR title the commit subject** on `main`,
and [release notes are generated from those subjects](releases.md). A sloppy title is
not a cosmetic issue — it is a permanently wrong line in the changelog. Individual
commits *within* the branch may be as messy as you like (`wip`, `typo`, `argh`); squash
erases them.

Use `!` for breaking changes (`feat!: require api key on export endpoint`) — release
tooling uses it to compute the [SemVer](releases.md) bump.

### Merge strategy: squash only

Repositories allow **squash merge and nothing else** (merge commits and rebase merge
disabled). Squash merge gives us:

- **Linear history** — `git log main` is a clean sequence of reviewed changes, one line
  per PR, bisectable without navigating merge topology.
- **Clean revert units** — reverting a feature is `git revert <one commit>`, not
  untangling which of nineteen commits belonged to it.
- **Invisible WIP** — branch-local noise commits never pollute `main`, so authors can
  commit freely while working instead of curating history as they go.

### Description

Every PR uses the repository's `.github/pull_request_template.md`:

```markdown
## What
<!-- The change, in one or two sentences. -->

## Why
<!-- The problem or requirement. Link context, don't restate it. -->

## How tested
<!-- Unit tests? Manually against dev? Terraform plan output? Be specific. -->

## Breaking changes
<!-- "None", or exactly what breaks and what consumers must do. -->
```

Link the issue with a closing keyword — `Closes #142` — so the issue closes on merge
and the PR ⇄ issue trace exists both directions.

### Draft PRs

Open a **draft PR** when you want early feedback on direction before the work is
finished. Drafts run CI but do not request review or notify CODEOWNERS. Convert to
ready only when CI is green and you would defend every line — "ready for review" is a
statement that the author's own review already happened
(see [Code Reviews](code-reviews.md)).

### Checks before review

**Do not request review on a red build.** Required status checks (`CI` — see
[GitHub Actions](github-actions.md)) must be green first. Asking a human to review
code that a machine has already rejected wastes the most expensive resource in the
process and trains reviewers to ignore review requests.

### No self-merge

A PR is merged only after review approval — including for organization admins, and
including "trivial" changes. Ruleset enforcement applies to everyone
([Repository Structure](repository-structure.md)); admin bypass is disabled. If a
change is genuinely trivial, it will be trivially fast to review.

## Examples

A well-formed PR for `avlon-technologies/billing`:

```text
Title:  feat: add invoice CSV export
Branch: feature/142-invoice-export
Body:
  ## What
  Adds GET /invoices/export returning CSV, streamed for large tenants.

  ## Why
  Finance needs month-end exports without database access. Closes #142.

  ## How tested
  Unit tests for the CSV writer (incl. quoting edge cases); manual run
  against dev with a 50k-invoice tenant; export opened in Excel.

  ## Breaking changes
  None.

Checks: CI ✓            Reviews: @avlon-technologies/billing-team ✓
Merge:  squash → "feat: add invoice CSV export (#157)" on main
```

## Anti-patterns

| Anti-pattern | Why it fails |
|---------------|--------------|
| 1,500-line PR mixing refactor and feature | Unreviewable; unrevertable as a unit; guarantees a rubber stamp |
| Title `updates` or `fix stuff` | Becomes a permanent, meaningless line in `git log` and the release notes |
| Requesting review with failing checks | Burns reviewer time on what a robot already caught |
| Merge commits or rebase-merge enabled | Breaks linear history and the one-PR-one-commit revert model |
| Description = the diff restated | Says *what* twice and *why* never; the diff already shows what |
| Self-merging with admin rights | One unreviewed change is how the "always reviewed" invariant dies |
| PR left open for weeks | Goes stale against `main`; see [Branching](branching.md) on branch lifetime |

## Tradeoffs

- **Small PRs mean more PRs**, more review requests, and more overhead per line of
  change. We accept this: overhead is linear, while review quality loss on big PRs is
  worse than linear.
- **Squash merge discards branch-level history.** A genuinely meaningful intermediate
  commit is lost. We accept this — in practice such commits are rare, and when the
  steps matter they should have been separate PRs.
- **Conventional Commit titles are a vocabulary to learn** and occasionally an awkward
  fit. We accept the constraint because generated release notes and SemVer automation
  depend on it.

## Related Standards

- [Branching](branching.md) — branch naming and lifetime feeding into PRs
- [Code Reviews](code-reviews.md) — what happens between "ready" and "approved"
- [GitHub Actions](github-actions.md) — the required checks that gate review and merge
- [Releases](releases.md) — release notes generated from PR titles
- [GitHub Naming](../naming/github.md) — branch and repository naming
