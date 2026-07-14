# Code Reviews

**Status:** Active
**Applies to:** Every pull request in the `avlon-technologies` organization.

---

## Purpose

Code review is the only quality gate in our process that can reason about intent —
CI can prove the tests pass, but only a human can notice that the tests assert the
wrong thing. Review is also how knowledge spreads: every review is one more engineer
who can maintain that code at 2 a.m.

This standard exists to keep review being that, and not the two failure modes it decays
into: **rubber-stamping** (approvals without reading, which is process theater) and
**gatekeeping** (style nitpicking and taste disputes, which burn goodwill and slow
delivery while catching nothing). Review is a quality gate, not a power structure.

## Guiding Principles

1. **Review the change, not the author.** Comments address the code ("this query runs
   per-row") never the person ("you always forget indexes"). Seniority flows in neither
   direction: a junior engineer's review of a principal's PR carries the same weight.
2. **Automate style out of review.** Anything a formatter or linter can decide is
   decided by the formatter or linter, in CI. Humans review what machines cannot:
   correctness, design, security, operability.
3. **Reviews are work, not interrupts.** Reviewing is part of the job, scheduled like
   the job — not something squeezed between "real" tasks. A team's throughput is gated
   by its review latency.
4. **Blocking must be distinguishable from preference.** A reviewer who mixes "this
   corrupts data" with "I'd have named this differently" at equal volume teaches
   authors to ignore both.
5. **Disagreements end with a decision, not a winner.** Long comment threads are the
   most expensive possible way to disagree.

## Standard

### Approval requirements

Every PR requires **at least one approval**; **platform and Terraform-module
repositories require two** because their blast radius is every consumer
(enforced by ruleset — see [Repository Structure](repository-structure.md)).
`.github/CODEOWNERS` routes each review to the **owning team** — `@avlon-technologies/billing-team`,
`@avlon-technologies/platform-team` — so requests never depend on one person's calendar and review
load spreads across the team.

### What reviewers must check

- **Correctness.** Does the change do what the description claims? Edge cases, error
  paths, concurrency, off-by-ones. Do the tests actually assert the behavior, or just
  execute the code?
- **Security.** Input validation, authorization on new endpoints, secrets kept out of
  code and logs, injection surfaces. For `infra/` changes: RBAC scope, network exposure,
  anything that loosens the posture in [Terraform Security](../terraform/security.md).
- **Tests.** New behavior has tests; changed behavior has changed tests. A bugfix
  includes the test that would have caught the bug.
- **Operability.** Can this be debugged in production? Logs at failure points, metrics
  for new paths, timeouts on new external calls, a rollback story for migrations.
- **Adherence to these standards.** Naming per [General Naming Principles](../naming/general.md),
  Terraform per its standards, workflow changes per
  [GitHub Actions](github-actions.md).

### What reviewers must NOT do

Do not comment on formatting, import order, brace style, or any rule a linter could
enforce. If style feedback keeps recurring, the correct fix is a PR **to the linter
configuration** — then the rule is enforced forever, for everyone, by a machine that is
never tired or tactless. Style comments in review are a signal of missing automation,
not of reviewer diligence. Likewise, do not relitigate architecture that was already
decided in an ADR — reopen the ADR if you must, in its own PR.

### Response time

**First response within one business day** — a review, a comment, or an honest "I can't
get to this before Thursday, try @avlon-technologies/billing-team". Silence is the one prohibited
response: the author cannot distinguish "busy" from "ignored", and open PRs rot against
their base branch — `main` or `develop` ([Branching](branching.md)). Teams should batch
review time into their day
deliberately (start of day, after lunch) rather than treating requests as interrupts.

### How to give feedback

- **Be specific and actionable.** Not "this is confusing" but "extract the discount
  logic into a named function so the tax path reads linearly".
- **Prefix non-blocking comments with `nit:`.** A `nit:` is a suggestion the author may
  take or leave without re-review. Everything unprefixed is blocking — so keep the
  blocking set honest.
- **Explain the why.** "This allocates per request — under invoice-export load that's
  measurable" teaches; "don't do this" doesn't.
- **Approve with nits.** If all remaining comments are nits, approve. Do not hold a PR
  hostage to preferences.

### Author responsibilities

- **Self-review first.** Read your own diff in the GitHub UI before marking ready — you
  will catch the leftover debug line and the accidental file. A PR that visibly wasn't
  self-reviewed is legitimately bounced.
- **Annotate non-obvious decisions** with PR comments on your own lines: "Retry is
  bounded here because the upstream throttles at 10 rps." Every question you preempt is
  a round-trip saved.
- **Keep PRs small** ([Pull Requests](pull-requests.md)) — PR size is the single
  biggest factor in review quality, and it is the author's to control.
- **Respond to every comment** — fix, discuss, or explicitly decline with reasoning.
  Never silently resolve someone else's comment.

### Disagreement resolution

When a comment thread exceeds roughly three exchanges without converging, stop typing:

1. **Go synchronous.** A 10-minute call resolves what 10 comments cannot — text strips
   tone and doubles misunderstanding.
2. **If still unresolved, the team lead decides.** Someone must own the tiebreak; a
   decision either way beats a stalled PR.
3. **Record significant decisions in an ADR** so the argument is not rerun in the next
   PR (see the repository's `docs/adr/` convention). The PR thread is where decisions
   happen; the ADR is where they are remembered.

## Examples

```text
[blocking] The export query loads all invoices into memory before writing.
A 50k-invoice tenant will OOM the container — stream with a cursor instead
(see how /invoices/list does it).

[blocking] New endpoint has no authorization check — any authenticated user
can export any tenant's invoices.

nit: `exportInvoicesToCsvFile` could just be `exportCsv`; the package name
already says invoices.

Approving with nits — the streaming fix looks right. Nice test on the
quoting edge case.
```

## Anti-patterns

| Anti-pattern | Why it fails |
|---------------|--------------|
| "LGTM" thirty seconds after a 400-line PR opens | Approval theater; the gate exists but gates nothing |
| Style nitpicks in review | Wastes human attention on machine work; fix the linter config instead |
| Blocking a PR over naming preference without saying `nit:` | Authors can't tell signal from taste; everything reads as blocking |
| Letting a review request sit silently for days | Stalls the author, rots the branch, and hides the capacity problem |
| Ten-comment thread arguments | Most expensive way to disagree; go synchronous at three |
| Requesting a specific individual instead of the team | Recreates the single-point-of-failure CODEOWNERS exists to remove |
| Author resolving comments without replying | Reviewer can't tell "fixed" from "dismissed" |

## Tradeoffs

- **Review latency is real latency.** Requiring approval (two for platform repos) adds
  hours to every change. We accept it: the one-business-day response rule bounds the
  cost, and unreviewed changes cost more when they fail.
- **`nit:` discipline lets imperfect code merge.** Some nits never get taken. Accepted —
  the alternative is every preference becoming a blocker.
- **Lead-decides can override the reviewer.** Occasionally the reviewer was right. The
  ADR record means the decision is revisitable with evidence rather than relitigated
  from memory.

## Related Standards

- [Pull Requests](pull-requests.md) — size, titles, and readiness before review
- [Repository Structure](repository-structure.md) — CODEOWNERS and approval-count rulesets
- [GitHub Actions](github-actions.md) — the automation that keeps style out of review
- [Terraform Security](../terraform/security.md) — the security posture reviewers enforce on `infra/`
