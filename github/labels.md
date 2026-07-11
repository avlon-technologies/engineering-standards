# Labels

**Status:** Active
**Applies to:** Issues and pull requests in all repositories in the `avlon-technologies` organization.

---

## Purpose

Labels are the query language of triage. When every repository uses the same small,
prefixed label set, one saved filter finds all blocked work across the organization,
one automation escalates every `priority/p1`, and an engineer moving between
repositories triages identically everywhere.

Without a standard, labels grow organically into chaos: `bug`, `Bug`, `defect`, and
`type: bug` coexist; colors mean nothing; and half the labels were created for one
issue in 2024 and never used again. Labels that cannot be relied on cannot be queried,
and labels that cannot be queried are decoration.

## Guiding Principles

1. **Labels exist for their consumers.** A label earns its place only if a filter, an
   automation, or a report reads it. A label nobody queries is deleted.
2. **Prefixes make labels a schema.** `type/bug` is machine-parseable and groups
   naturally in every UI; a flat soup of words is neither.
3. **One color per prefix group.** Color signals *kind of metadata* at a glance —
   what the item is, how urgent, what state, which component — before reading a word.
4. **Provisioned, never hand-created.** Hand-created labels drift in spelling, casing,
   and color within a week. Automation makes every repository identical by
   construction.
5. **Small enough to memorize.** If engineers must look up the label set, they will
   guess instead, and guesses create variants.

## Standard

### Taxonomy

Four prefix groups, one color each:

| Group | Labels | Color | Hex |
|-------|--------|-------|-----|
| `type/*` | `type/bug`, `type/feature`, `type/chore`, `type/docs` | Red | `#d73a4a` |
| `priority/*` | `priority/p1`, `priority/p2`, `priority/p3` | Orange | `#e36209` |
| `status/*` | `status/blocked`, `status/needs-review` | Yellow | `#fbca04` |
| `area/*` | `area/<component>`, e.g. `area/billing-api`, `area/infra` | Blue | `#0366d6` |

**`type/*`** — what the item is. Exactly one per issue. `type/bug` is defective
existing behavior; `type/feature` is new behavior; `type/chore` is work with no
user-visible change (dependency bumps, refactors, CI); `type/docs` is documentation.
These mirror the [Conventional Commit types](pull-requests.md) deliberately, so an
issue's type and its PR's title prefix agree.

**`priority/*`** — how urgent, with definitions everyone shares:

- `priority/p1` — drop other work. Production is broken or materially degraded, or a
  security issue is exploitable. Worked continuously until resolved or downgraded.
- `priority/p2` — this sprint. Important defect or commitment with a real deadline;
  scheduled, not interrupt-driven.
- `priority/p3` — when capacity allows. Real work (or it would be closed), but nothing
  breaks if it waits a quarter.

There is no `p0` and no `p4`: a scale everyone inflates to the top of is not a scale,
and a "lower than lowest" tier is a euphemism for *close it*.

**`status/*`** — current state that someone should act on. `status/blocked` means
externally stuck — the blocker goes in a comment, and triage reviews every blocked item
weekly. `status/needs-review` means waiting on reviewer attention. Status labels are
removed the moment they stop being true; a stale status label is worse than none.

**`area/<component>`** — which part of the system, matching the repository's actual
components (`area/billing-api`, `area/billing-web`, `area/infra`). The only group with
repo-specific values; keep it to components that actually exist, and use it to route
triage to the right owners.

### Provisioning

Labels are **provisioned by automation from the organization template repository** —
a labels manifest applied to every repository by a scheduled platform workflow (see
[GitHub Actions](github-actions.md)). Nobody creates or edits labels by hand in
individual repositories: manual labels are how `Bug` and `type/bug` come to coexist.
Changing the taxonomy means a PR to the manifest — reviewed once, applied everywhere.

### Keep the set small

Every label must name its consumer at creation time: the filter, automation, or report
that reads it. The standing consumers today are triage queries (below), the p1
escalation workflow, and the weekly blocked-items review. Proposed labels without a
consumer are rejected; labels whose consumer disappears are removed from the manifest.

## Examples

Triage queries used in practice:

```text
# Org-wide: exploitable or broken-in-prod work, oldest first
org:avlon-technologies is:open label:priority/p1 sort:created-asc

# Weekly blocked review for the billing team
org:avlon-technologies is:open label:status/blocked team-review-requested:avlon-technologies/billing-team

# Sprint planning: candidate bugs in the API component
repo:avlon-technologies/billing is:issue is:open label:type/bug label:area/billing-api -label:status/blocked

# PRs waiting on reviewer attention
org:avlon-technologies is:pr is:open label:status/needs-review sort:updated-asc
```

A well-labeled issue: `type/bug` + `priority/p2` + `area/billing-api` — one type, one
priority, the affected component, and a status label only while something is stuck.

## Anti-patterns

| Anti-pattern | Why it fails |
|---------------|--------------|
| Hand-creating labels per repository | Spelling/color drift breaks every cross-repo query |
| Unprefixed labels (`bug`, `urgent`) | Unparseable by automation; collides with the prefixed set |
| `priority/p0`, `priority/critical-really` | Priority inflation destroys the scale for everyone |
| A label per adjective (`easy`, `weird`, `tech-debt-ish`) | No consumer; noise that buries the labels that matter |
| Stale `status/blocked` on unblocked work | Poisons the blocked report; worse than no label |
| Multiple `type/*` labels on one issue | If it is genuinely two types, it is two issues |
| Encoding assignee or milestone in labels | GitHub has native fields for both; labels duplicate them badly |

## Tradeoffs

- **A small fixed set cannot express everything.** Some nuance ("needs design input")
  has no label. Accepted: nuance lives in comments; labels are for queries, and an
  expressive-but-unqueryable set is the worse failure.
- **Central provisioning adds process** — a manifest PR instead of two clicks.
  Accepted: the two clicks are exactly how taxonomies rot.
- **Priority definitions require judgment calls** at the p1/p2 boundary. Accepted: a
  fuzzy boundary with shared definitions beats precise-looking labels nobody agrees on.

## Related Standards

- [GitHub Naming](../naming/github.md) — label naming conventions
- [Pull Requests](pull-requests.md) — Conventional Commit types that `type/*` mirrors
- [GitHub Actions](github-actions.md) — the automation that provisions and consumes labels
- [Repository Structure](repository-structure.md) — repository setup that includes label provisioning
