# Outputs

**Status:** Active
**Applies to:** Output declarations in all Avlon Terraform modules and environment roots.

---

## Purpose

Outputs are the half of the module contract everyone under-designs. Variables get types, validation, and review scrutiny; outputs get whatever `main.tf` happened to produce. But every output is a **commitment**: the moment a module exposes a value, some consumer — another module, a pipeline step, a human reading `terraform output`, a `terraform_remote_state` reader — can depend on it, and removing or renaming it becomes a breaking change. Undisciplined outputs leak secrets into plan logs, couple consumers to provider internals, and turn refactors into cross-repo migrations. This standard treats outputs as what they are: a public API, designed deliberately.

## Guiding Principles

1. **Every output is a promise.** Publishing a value invites dependencies you cannot see. Expose what consumers need, and nothing "just in case."
2. **Minimal beats complete.** Adding an output later is trivial and non-breaking; removing one requires finding every consumer first. Asymmetry says: start small.
3. **Outputs expose meaning, not mechanism.** Consumers should receive "the Key Vault URI", not "the azurerm resource object" — the value, not the provider schema behind it.
4. **Sensitivity is contagious by design.** If a value derives from something sensitive, the output says so, and Terraform enforces it. Fighting that propagation is a red flag, not an obstacle.
5. **A root's outputs are the workload's published interface.** What a workload's root outputs is what the rest of Avlon may know about it.

## Standard

### Naming and shape

- Outputs are named **`<object>_<attribute>`**: `key_vault_uri`, `container_app_fqdn`, `resource_group_name`, `sql_server_fqdn`. The object half matches the module's vocabulary, the attribute half says which fact about it. Full conventions in [Terraform Naming](../naming/terraform.md).
- **Every output has a `description`** — one line stating what the value is and what a consumer would use it for. `terraform output` and generated docs are only as useful as these lines.
- Output **scalar values and small purpose-built objects** — never entire resource objects. `value = azurerm_key_vault.main` exposes forty attributes you never meant to promise, couples every consumer to the azurerm schema, and turns a provider upgrade into a consumer-breaking event. Export `azurerm_key_vault.main.vault_uri` and its friends individually.

### Sensitive propagation

Mark `sensitive = true` on any output whose value is, contains, or derives from a sensitive value. Terraform forces this — an output built from a sensitive variable or attribute fails to plan without the flag — and the correct response is to comply, not to launder the value through `nonsensitive()`. Sensitive outputs are redacted in plan/apply output and in the PR-posted plans our pipelines produce (see [Security](security.md) on why plan output is a leak surface). Remember the limits: `sensitive` redacts console output, but the value still sits in state in plaintext — one more reason state access is locked down ([Remote State](remote-state.md)).

### Root-module outputs: the workload's interface

An environment root's `outputs.tf` is the workload's **published interface** for that environment. Its consumers are concrete:

- **Pipelines** — the deploy workflow reads `terraform output -raw container_app_fqdn` for smoke tests, or the static site endpoint for cache purges.
- **Humans** — `terraform output` is the first thing an engineer runs to orient in an unfamiliar stack.
- **Other stacks** — the rare, deliberate `terraform_remote_state` consumer reads exactly these values ([Remote State](remote-state.md) explains why name-based data sources are usually better).

Design root outputs with the same restraint as module outputs: the workload's entry points, identifiers other stacks legitimately need, and operationally useful facts. A root that outputs nothing is suspicious; a root that outputs thirty values is publishing its internals.

### Change discipline

Because outputs are API, changes follow API rules:

- **Additive changes are free** — new outputs need no ceremony.
- **Renames and removals are breaking.** In shared modules (`terraform-azurerm-*`), that means a **major version bump** under SemVer ([Releases](../github/releases.md)), release notes naming the removed output, and ideally one overlapping minor release where old and new names coexist.
- In workload roots, grep the repo's workflows (and ask before assuming nobody reads a value) before deleting an output.

## Examples

`outputs.tf` from the `terraform-azurerm-container-app` module — minimal, described, meaningfully named:

```hcl
output "container_app_fqdn" {
  description = "Public FQDN of the Container App's ingress endpoint."
  value       = azurerm_container_app.main.ingress[0].fqdn
}

output "container_app_identity_principal_id" {
  description = "Principal ID of the app's user-assigned identity, for RBAC grants by the caller."
  value       = azurerm_user_assigned_identity.main.principal_id
}

output "key_vault_uri" {
  description = "URI of the workload Key Vault, for app configuration references."
  value       = azurerm_key_vault.main.vault_uri
}

output "resource_group_name" {
  description = "Name of the resource group containing the workload."
  value       = azurerm_resource_group.main.name
}

output "sql_connection_string" {
  description = "ADO.NET connection string (Entra ID auth). Sensitive: redacted in output, present in state."
  value       = local.sql_connection_string
  sensitive   = true
}
```

And a pipeline consuming the root's interface after apply:

```bash
FQDN=$(terraform output -raw container_app_fqdn)
curl --fail "https://${FQDN}/healthz"
```

## Anti-patterns

| Anti-pattern | Why it's wrong |
|--------------|----------------|
| `output "key_vault" { value = azurerm_key_vault.main }` | Publishes the whole provider object: forty accidental promises, schema coupling, and any sensitive attribute comes along for the ride. Output named scalars. |
| Outputting everything "so consumers have options" | Every value is now load-bearing forever. You cannot refactor what you cannot stop exposing. |
| Missing descriptions | `terraform output` becomes a quiz; terraform-docs tables go blank. Describe every output. |
| `nonsensitive()` to silence a sensitivity error | Deliberately stripping the safety marking so a secret prints in CI logs. If a consumer needs the value, propagate `sensitive = true` end to end. |
| Names like `fqdn`, `uri`, `id` without the object half | Ambiguous the moment a module has two resources. `<object>_<attribute>`, always. |
| Renaming an output in a shared module's minor release | Breaks every pinned consumer that upgrades. Renames are major-version events. |
| Treating root outputs as a dumping ground for debugging values | The workload's published interface fills with noise nobody may ever remove. Debug with `terraform console`, not permanent API. |

## Tradeoffs

- **Minimalism creates follow-up PRs.** Consumers will occasionally need a value the module didn't expose, and must wait for an additive release. That friction is mild and one-directional — far cheaper than the un-removable sprawl of defensive over-exposure.
- **Scalar outputs are verbose.** Exporting five attributes of one resource takes five blocks where `value = resource` takes one. The verbosity is the API surface being written down explicitly — exactly the property we want reviewed.
- **Sensitive redaction hinders debugging.** A redacted output is annoying mid-incident. The escape hatch (`terraform output -raw`, under an identity already entitled to state) exists; logs that never contain secrets are worth the extra step.

## Related Standards

- [Modules](modules.md) — outputs as half of the module contract
- [Variables](variables.md) — the input half, including `sensitive` inputs
- [Remote State](remote-state.md) — `terraform_remote_state` and why name lookups usually beat it
- [Security](security.md) — plan output as a leak surface; state secrecy
- [Terraform Naming](../naming/terraform.md) — output naming conventions
