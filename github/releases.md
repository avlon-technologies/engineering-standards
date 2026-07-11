# Releases

**Status:** Active
**Applies to:** All repositories in the `avlon-technologies` organization that publish versions — applications and Terraform modules alike.

---

## Purpose

A release turns a point in `main`'s history into a named, immutable, consumable fact:
*this* is v2.3.0, forever. Versions let consumers reason about upgrades without reading
diffs, let incident responders answer "what changed?" in seconds, and let Terraform
module consumers pin exactly what they depend on.

This standard exists because the failure modes are quiet and expensive: a "v1.4"
that means something different in each repo, a re-pointed tag that makes two teams'
"v2.1.0" different code, a hand-written CHANGELOG that stopped being true in March, and
a module upgrade that silently breaks every caller because nobody signaled the breaking
change.

## Guiding Principles

1. **Versions carry meaning, not just order.** SemVer's promise is that the version
   number alone tells a consumer whether an upgrade is safe. That promise is only worth
   what our discipline makes it worth.
2. **Tags are immutable facts.** A tag records what was released. Facts do not get
   edited — a wrong release is followed by a new release.
3. **Releases are generated, not written.** Notes and changelogs derived mechanically
   from merged PR titles are always complete and always current; hand-written ones are
   neither.
4. **Deployment and release are different events for apps.** Apps deploy continuously
   from `main`; a tag marks what was released, it does not cause the deploy.
5. **Modules without pinned versions are a shared mutable variable.** Terraform module
   consumers must be able to upgrade deliberately, one caller at a time.

## Standard

### Format

Releases are annotated git tags `v<major>.<minor>.<patch>` — `v2.3.0` — each with a
**GitHub Release** using generated notes. [SemVer](https://semver.org/) governs the
bump:

**For applications:**

- **major** — breaking behavior or API change: an endpoint removed, a response shape
  changed, a config value renamed. Anything that forces a consumer or operator to act.
- **minor** — new features, backwards compatible.
- **patch** — bug fixes only, no new behavior.

**For Terraform modules** — the critical case (see
[Terraform Modules](../terraform/modules.md)):

- **major** — any breaking change to the **variable or output contract**: a variable
  removed or renamed, a new required variable, a changed variable type, an output
  removed, or a change that forces resource replacement. The test is concrete:
  *if any caller must edit code (or face a destructive plan) to upgrade, it is major.*
- **minor** — new optional variables, new outputs, new capabilities with safe defaults.
- **patch** — fixes that alter no contract and destroy no resources.

Module majors matter more than app majors because the failure is silent until apply:
a consumer bumping `terraform-azurerm-container-app` across an unmarked breaking change
gets a broken plan — or worse, a plan that quietly destroys production resources.

### Applications: deploy from main, tag what shipped

Apps deploy continuously from `main` via the pipeline
([GitHub Actions](github-actions.md)) — deployment needs no tag. Tags are the
**record**: when a `main` commit is promoted to `prod`, it is tagged and a GitHub
Release is published, so "what is running in prod" and "what changed since last
release" are answerable from the repository alone.

### Terraform modules: release by tag, consume by tag

Module repositories (`terraform-azurerm-<purpose>`) **MUST release exclusively by
tag**, and consumers **MUST pin an exact tag**:

```hcl
module "container_app" {
  source = "git::https://github.com/avlon-technologies/terraform-azurerm-container-app.git?ref=v2.3.0"
  # ...
}
```

Never `?ref=main`, never a branch, never tagless. An unpinned source means every
`terraform init` may pull different code — the module repo's `main` becomes a live
dependency of every environment that references it, and a merged module change deploys
itself everywhere on the next plan, unreviewed by any consumer.

### Release notes: generated from Conventional Commit PR titles

GitHub Releases use **generated release notes**. Because every merge is a squash whose
subject is a [Conventional Commit PR title](pull-requests.md), the generated notes
group mechanically — `feat:` under Features, `fix:` under Fixes, `!` flagged as
breaking. **This is why the PR title standard exists**: the discipline is paid once per
PR and collected at every release. Categories are configured in
`.github/release.yml`:

```yaml
# .github/release.yml
changelog:
  categories:
    - title: Breaking changes
      labels: [breaking]
    - title: Features
      labels: [type/feature]
    - title: Fixes
      labels: [type/bug]
    - title: Other
      labels: ["*"]
```

The **CHANGELOG is generated, never hand-written**. The release history *is* the
changelog; a parallel hand-maintained `CHANGELOG.md` starts complete and decays into a
lie. If a file is wanted in-repo, it is produced by tooling from the same titles.

### Tags are immutable — no re-tagging, ever

A pushed tag is never moved, deleted, or reused. If `v2.3.0` shipped broken, release
`v2.3.1`. Re-pointing a tag makes the same version name mean different code for
different consumers — Terraform module caches, lockfiles, and humans all disagree about
what `v2.3.0` is, and no audit trail can untangle it. Rulesets block tag deletion and
updates in module repositories.

### Pre-releases

When staged validation needs a named build — a release candidate soaking in `stg`
before a major — use SemVer pre-release tags: `v3.0.0-rc.1`, `v3.0.0-rc.2`, then
`v3.0.0`. Pre-release tags are just as immutable; `rc.2` follows `rc.1`, never replaces
it. Mark the GitHub Release as *pre-release* so generated notes and consumers treat it
accordingly.

## Examples

Releasing `terraform-azurerm-container-app` after a breaking variable rename
(`container_image` → `image`):

```bash
# The PR was titled:  feat!: rename container_image variable to image
git tag -a v3.0.0 -m "v3.0.0"
git push origin v3.0.0
gh release create v3.0.0 --generate-notes
```

The generated notes flag the breaking change; consumers upgrade deliberately:

```hcl
# avlon-technologies/billing — infra/environments/prod/main.tf, upgraded in its own reviewed PR
module "container_app" {
  source = "git::https://github.com/avlon-technologies/terraform-azurerm-container-app.git?ref=v3.0.0"
  image  = "acrplatformsharedcc01.azurecr.io/billing-api:9f3ab12"   # was: container_image
}
```

Meanwhile `avlon-technologies/billing` itself deploys from `main` all week and tags `v1.8.0` when
the invoice-export feature reaches `prod`.

## Anti-patterns

| Anti-pattern | Why it fails |
|---------------|--------------|
| Module consumed with `?ref=main` | Module `main` becomes a live, unreviewed dependency of every environment |
| Moving a tag to "fix" a bad release | Same version now means different code for different consumers; nothing is auditable |
| Hand-written CHANGELOG | Complete on day one, fiction by month three |
| Vague PR titles (`updates`) | Directly become garbage release notes — the cost lands here |
| Breaking module change shipped as minor | Consumers upgrade "safely" into a destructive plan |
| Tagging apps to trigger deployment | Couples release bookkeeping to deploy mechanics; the pipeline deploys `main`, tags record it |
| `v1.4-final-2` style ad-hoc versions | Unparseable by tooling; unorderable by humans |

## Tradeoffs

- **SemVer judgment calls are real** — is a behavior fix someone depended on a patch or
  a major? Accepted: we bump conservatively (when in doubt, go bigger), because a
  surprise breaking change costs more than a too-cautious major.
- **Exact module pinning means upgrades are manual** across every consumer — no
  automatic fleet-wide fixes. Accepted deliberately: Dependabot raises the upgrade PRs,
  and each consumer reviews its own plan.
- **Generated notes are only as good as PR titles**, and occasionally read mechanical.
  Accepted: mechanically accurate beats artisanally stale, and title quality is
  enforced upstream in [Pull Requests](pull-requests.md).

## Related Standards

- [Pull Requests](pull-requests.md) — the Conventional Commit titles release notes are built from
- [Terraform Modules](../terraform/modules.md) — module contract and versioning details
- [GitHub Actions](github-actions.md) — the pipeline that deploys `main` and promotes environments
- [Branching](branching.md) — why `main` is always in a releasable state
- [GitHub Naming](../naming/github.md) — repository and tag naming
