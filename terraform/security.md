# Terraform Security

**Status:** Active
**Applies to:** All Terraform code, state, pipelines, and the identities that run them at Avlon.

---

## Purpose

Terraform concentrates risk: the code can create and destroy anything it's entitled to, the pipeline that runs it holds (or *is*) a privileged identity, and the state it writes contains every secret its resources ever emitted. A leaked pipeline credential or an over-scoped deployment identity isn't a bug — it's tenant-wide compromise waiting for a phish. This standard closes the recurring holes: credentials in code, God-mode service principals, secrets riding through tfvars, state treated as just another file, and plans that print secrets into PR comments.

## Guiding Principles

1. **No credential exists to leak.** The strongest secret management is having no secret: OIDC federation instead of client secrets, managed identities instead of connection strings, Entra ID auth instead of access keys.
2. **Every identity gets exactly what its job requires.** A deployment identity for billing-dev deploys billing to dev. It cannot read prod state, cannot touch code-review's resource group, and is not Owner of anything.
3. **State is a secrets database.** Whatever touches state — storage, RBAC, backups, humans — is handled as if it were Key Vault, because functionally it is.
4. **The pipeline is the perimeter.** Everything reaching Azure goes through reviewed code and a gated pipeline; the security properties of the pipeline *are* the security properties of the infrastructure.
5. **Destruction of stateful resources is opt-in, twice.** Anything holding data that can't be regenerated is guarded in code, so a bad plan needs deliberate human effort — not just a missed review — to destroy it.

## Standard

### No hardcoded credentials — anywhere

No passwords, keys, tokens, or connection strings in `.tf` files, tfvars, pipeline YAML, or GitHub repository/environment variables. Not "encrypted", not "just for dev", not commented out. Git history is forever, tfvars are code, and pipeline variables are readable by everyone who can edit a workflow. Scanning (below) enforces this; the architecture below makes it unnecessary.

### CI/CD authentication: Workload Identity Federation

Pipelines authenticate to Azure via **OIDC / Workload Identity Federation** — GitHub Actions presents a short-lived, cryptographically verifiable token about *which repo, on which environment* is running, and Entra ID exchanges it for Azure credentials. No client secret exists, so none can expire, leak, or be exfiltrated. (Workflow mechanics in [GitHub Actions](../github/github-actions.md).)

**One deployment identity per workload per environment** — a user-assigned managed identity following the `mi-github-<workload>-<env>-<region>` pattern:

```text
mi-github-billing-dev-cc
mi-github-billing-prod-cc
mi-github-code-review-dev-cc
```

Each carries a **federated credential** whose subject claim binds it to exactly one repo + GitHub environment:

```hcl
resource "azurerm_federated_identity_credential" "github_prod" {
  name                = "github-billing-prod"
  resource_group_name = azurerm_resource_group.identity.name
  parent_id           = azurerm_user_assigned_identity.github_billing_prod.id
  audience            = ["api://AzureADTokenExchange"]
  issuer              = "https://token.actions.githubusercontent.com"
  subject             = "repo:avlon-technologies/billing:environment:prod"
}
```

The subject claim is the security boundary: only a job in `avlon-technologies/billing` running against the `prod` GitHub environment — which means it passed that environment's protection rules and required reviewers ([GitHub Actions](../github/github-actions.md)) — can become `mi-github-billing-prod-cc`. A fork, another repo, or a dev job gets nothing. Per-workload-per-environment identities keep the failure mode bounded: compromising the billing dev pipeline yields exactly billing-dev privileges.

### Least-privilege RBAC for deployment identities

Each deployment identity receives (assigned via groups where practical — see [Identity](../azure/identity.md)):

- **`Contributor` scoped to the workload's resource group(s)** for its environment — `rg-billing-prod-cc`, not the subscription. RG-scoped Contributor can build the workload but cannot see or touch neighbors.
- **Specific data-plane roles** the deploy actually needs: `Storage Blob Data Contributor` on its state container ([Remote State](remote-state.md)), `Key Vault Secrets Officer` on the workload's vault if Terraform manages secrets there, `AcrPush` on `acrplatformsharedcc` for the CI identity that publishes images ([Container Registry](../azure/container-registry.md)).
- **`Role Based Access Control Administrator` with a condition** limiting assignable roles, *only if* the stack manages RBAC assignments — never blanket `Owner` or `User Access Administrator`.

Never subscription `Owner`, never tenant-level anything. When a stack needs one privilege beyond its RG (a DNS record in the platform zone, a peering), grant that narrow permission at that narrow scope rather than widening the identity.

### State is sensitive — full stop

State files contain secrets in **plaintext**: generated passwords, connection strings, keys returned by providers. Therefore, as detailed in [Remote State](remote-state.md):

- Container-scoped RBAC; dev identities can never read prod state.
- **No state in git, ever.** Every Terraform repo's `.gitignore` includes `*.tfstate`, `*.tfstate.*`, and `.terraform/`. A state file that reaches a commit is an incident: rotate every secret it contains, then scrub history.
- Blob versioning + soft delete for recovery; shared-key access disabled on the account.

### Secrets flow: Key Vault, one direction

Secrets originate in **Key Vault** (`kv-billing-prod-cc`) and flow toward consumers — never through tfvars, variables, or pipeline configuration:

- **Best: app-side references.** The application reads Key Vault at runtime via its managed identity (Key Vault references in App Service / Container Apps). Terraform wires up identity and access, and never touches the secret value at all — it appears in neither plan nor state.
- **Acceptable: data source at deploy time**, when a resource argument genuinely requires the value:

```hcl
data "azurerm_key_vault_secret" "sql_admin_password" {
  name         = "sql-admin-password"
  key_vault_id = data.azurerm_key_vault.billing.id
}
```

  The value now exists in state (one more reason state RBAC is strict) but still never in code, git, or pipeline config, and rotation happens in Key Vault without a code change.

- **Never:** secrets in tfvars or `-var` flags. See [Variables](variables.md).

### Guard stateful resources with `prevent_destroy`

Databases, storage accounts holding data, Key Vaults, and the state storage itself carry `lifecycle { prevent_destroy = true }`. A plan that would destroy them **fails** — even if a reviewer skims the plan, even if a refactor accidentally forces replacement. Removing the guard is a visible, greppable code change in its own right, which is exactly the second deliberate step we want between a mistake and a data loss event. (See the state account in [Backend Bootstrap](backend-bootstrap.md) for the canonical example.)

### Plan output is a leak surface

Our pipelines post `terraform plan` output to pull requests — visible to everyone with repo read access, and quoted into notifications. Anything unmarked in a plan diff is therefore published. Mitigations: `sensitive = true` on secrets-adjacent variables and outputs ([Variables](variables.md), [Outputs](outputs.md)) so Terraform redacts them; prefer the app-side reference pattern so values never enter the graph; and treat a secret spotted in a posted plan as a rotation event, not an embarrassment.

### Scanning in CI

Every Terraform repo runs, as required PR checks alongside fmt/validate/tflint ([Repository Layout](repository-layout.md)):

- A **static security scanner** — `checkov` or `tfsec` — catching insecure configurations before plan: public blob access, missing TLS floors, over-permissive NSG rules, unencrypted disks, hardcoded secrets.
- Findings are fixed or explicitly suppressed **inline with a justification comment**; suppressions without reasons are rejected in review. A permanently-ignored scanner is worse than none — it manufactures false confidence.

Azure Policy provides the backstop on the platform side: even a misconfigured stack cannot create what policy denies.

## Examples

The federated credential above, plus the RBAC grant that pairs with it:

```hcl
resource "azurerm_role_assignment" "billing_prod_rg" {
  scope                = azurerm_resource_group.billing_prod.id   # rg-billing-prod-cc
  role_definition_name = "Contributor"
  principal_id         = azurerm_user_assigned_identity.github_billing_prod.principal_id
}

resource "azurerm_role_assignment" "billing_prod_state" {
  scope                = azurerm_storage_container.tfstate_prod.resource_manager_id
  role_definition_name = "Storage Blob Data Contributor"
  principal_id         = azurerm_user_assigned_identity.github_billing_prod.principal_id
}
```

Two assignments, two narrow scopes — the identity's entire world.

## Anti-patterns

| Anti-pattern | Why it's wrong |
|--------------|----------------|
| A service principal with a client secret in GitHub secrets | A long-lived credential that can be exfiltrated and used from anywhere. OIDC federation makes it unnecessary — see [GitHub Actions](../github/github-actions.md). |
| One `sp-terraform` identity with subscription Owner for all deploys | Every pipeline compromise is now a subscription compromise, and no audit trail distinguishes workloads. One identity per workload per environment. |
| Subject claim `repo:avlon-technologies/billing:ref:refs/heads/main` for prod | Binds to a branch, not to the gated `prod` environment — any push to main mints prod credentials without protection rules. Bind to `environment:prod`. |
| `sql_admin_password = "P@ssw0rd1"` in tfvars | A secret in git, in state, and in every clone. Key Vault, always. |
| Committing state, or granting broad read on state containers "for debugging" | State is plaintext secrets; both actions are secret disclosure. |
| Skipping `prevent_destroy` because "the plan gets reviewed" | Review is a probabilistic control; forced replacement diffs are easy to misread. Stateful resources get a deterministic guard. |
| Blanket `#checkov:skip` / scanner disabled after the first noisy run | The control exists only on the dashboard. Fix findings or justify suppressions inline, one by one. |

## Tradeoffs

- **Per-workload-per-environment identities multiply objects.** Ten workloads × three environments = thirty identities plus federated credentials and role assignments. It's boilerplate — Terraform-managed, naturally — and it buys precise blast radii and legible audit logs. We pay it.
- **Least privilege generates friction at the edges.** New stacks hit missing-permission errors that Owner would have hidden, and each becomes a small platform-team grant. Every such error is the control working; the alternative is discovering the missing boundary during an incident instead.
- **`prevent_destroy` obstructs legitimate teardown.** Retiring an environment means a PR to lift the guard first. Two steps to destroy a database is the intended number.
- **App-side secret references shift complexity to the app.** The application must handle Key Vault access and rotation behavior. In exchange, secrets vanish from Terraform's entire surface — code, plan, and state.

## Related Standards

- [GitHub Actions](../github/github-actions.md) — the OIDC workflow, environments, and protection rules
- [Identity](../azure/identity.md) — the RBAC and managed-identity model these rules apply
- [Remote State](remote-state.md) — state access control in full
- [Variables](variables.md) — `sensitive` inputs and the no-secrets-in-tfvars rule
- [Backend Bootstrap](backend-bootstrap.md) — `prevent_destroy` on the state account itself
