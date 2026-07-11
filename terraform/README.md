# Terraform Standards

Terraform is how Avlon Technologies builds and operates Azure infrastructure. Everything that runs in an Avlon subscription is described in code, reviewed in a pull request, and applied by a pipeline — the portal is for reading, not writing. This gives us three things we refuse to give up: **reviewability** (every infrastructure change has a diff, an author, and an approver), **reproducibility** (dev, stg, and prod are the same code with different values), and **recoverability** (any environment can be rebuilt from the repository).

Four convictions shape every standard in this directory. First, **state is sacred**: the state file is the single source of truth about what exists, it contains secrets, and losing or corrupting it turns a routine change into an incident — so it lives in hardened, versioned, access-controlled Azure Storage and nowhere else. Second, **small blast radii**: one state file per deployable workload per environment, so a bad plan for `code-review` in dev can never touch `billing` in prod. Third, **reusable modules with thin environment roots**: the logic lives in modules that solve categories of problems; environment roots are thin compositions that differ only in values. Fourth, **environments are directories, not workspaces**: each environment is an independent root with its own backend, its own state, and its own RBAC.

## Contents

| Document | What it covers |
|----------|----------------|
| [Repository Layout](repository-layout.md) | The standard `infra/` structure, when infra lives with the app vs. a dedicated repo, CI formatting and lint gates. |
| [Modules](modules.md) | Module design: contracts, secure defaults, local vs. shared modules, versioning, documentation, testing. |
| [Environments](environments.md) | Directory-per-environment roots, why we do not use workspaces, promotion flow, structural symmetry. |
| [Remote State](remote-state.md) | State in Azure Storage: layout, Entra ID auth, locking, versioning, access control, cross-state references. |
| [Backend Bootstrap](backend-bootstrap.md) | Creating the state storage itself: the one-time bootstrap configuration and its state migration. |
| [Providers](providers.md) | Terraform and provider version pinning, lock files, provider blocks, subscriptions, upgrade cadence. |
| [Variables](variables.md) | Typing, validation, defaults policy, sensitive inputs, tfvars discipline. |
| [Outputs](outputs.md) | Outputs as public API: minimalism, naming, descriptions, sensitivity. |
| [Security](security.md) | OIDC authentication, least-privilege deployment identities, state secrecy, secret flow, scanning. |

## How these standards fit together

A workload starts with [Repository Layout](repository-layout.md): an `infra/` directory containing local [Modules](modules.md) and one root per environment as described in [Environments](environments.md). Each root points its backend at the shared state storage defined in [Remote State](remote-state.md) — storage that exists because of the one-time [Backend Bootstrap](backend-bootstrap.md). Inside the code, [Providers](providers.md), [Variables](variables.md), and [Outputs](outputs.md) govern the mechanical contracts, and [Security](security.md) governs how pipelines authenticate and how secrets stay out of everything. Naming conventions for modules, resources, variables, outputs, and state keys are defined once in [Terraform Naming](../naming/terraform.md) — the documents here reference them and never restate them.
