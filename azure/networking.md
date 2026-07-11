# Networking

**Status:** Active
**Applies to:** All virtual networks, subnets, private endpoints, and DNS in all Avlon subscriptions.

---

## Purpose

The cheapest network attack surface to defend is the one that doesn't exist. Every public
endpoint on a database, storage account, or key vault is a door that must be guarded
forever — with firewall rules that drift, IP allowlists that grow and never shrink, and
the standing hope that authentication alone holds. Private networking removes the door:
traffic to data services never touches the public internet, and an attacker with stolen
credentials still needs a network position they don't have.

This document defines Avlon's network posture: **private by default in staging and
production**, with a hub-spoke topology, and honest, documented pragmatism in dev where
the cost of privacy exceeds its value.

## Guiding Principles

1. **Private by default, public by documented exception.** In `stg` and `prod`, PaaS
   data services are reachable only over private endpoints, and public network access on
   them is disabled — not "restricted", disabled. Exceptions are written down with an
   owner and a reason.
2. **Hub-spoke, because shared things need one home.** DNS, egress control, and future
   hybrid connectivity are estate-wide concerns. A hub VNet owned by the platform team
   hosts them once; workload spokes peer to the hub and inherit them. The alternative —
   every workload VNet solving DNS and egress alone — produces N inconsistent copies of
   the hardest part.
3. **A VNet is a tool, not a rite of passage.** Workloads get a VNet when something in
   them needs private networking. A dev workload on Container Apps with no private
   endpoints doesn't need one, and giving it one anyway is cost and complexity with no
   consumer.
4. **Deny is the default at every boundary.** NSGs on every subnet, no public IPs on
   compute, egress controlled. Reachability is granted, never assumed.
5. **Parity in what matters.** Staging's networking mirrors production exactly — private
   endpoints, DNS, topology — because connectivity failures are precisely the class of
   bug that only surfaces in the environment where the wiring is real (see
   [Environments](environments.md)).

## Standard

### Topology: hub and spokes

- **Hub:** `vnet-platform-prod-cc`, in `rg-platform-network-prod-cc`, owned by the
  platform team. Hosts the central private DNS zones, shared egress (firewall/NAT as the
  estate requires), and any future hybrid connectivity (VPN/ExpressRoute).
- **Spokes:** one VNet per workload per environment that needs one, in the workload's RG
  and Terraform state — e.g. `vnet-billing-prod-cc` — peered to the hub. Spokes never
  peer with each other; anything that looks like spoke-to-spoke traffic goes through a
  published interface (an API, a private endpoint), not a network path.
- Address space is allocated centrally by the platform team from a non-overlapping plan,
  recorded in the platform repo. Workloads request a block; they do not invent one.

### When a workload needs a VNet

- **Needs a VNet:** anything in `stg`/`prod` touching PaaS data services (which are
  private-endpoint-only there); anything requiring VNet integration for compliance;
  anything with internal-only ingress.
- **May not need one:** simple `dev` workloads. A dev Container App or App Service using
  public data endpoints with firewall rules (the documented dev tradeoff below) has
  nothing to put in a VNet. Container Apps and App Service are VNet-*integrable*, not
  VNet-*requiring* — integrate them when there is something private to reach.

Don't build the VNet "for later": the workload's Terraform makes adding it a planned
change, and an empty VNet is pure carrying cost.

### Subnets and NSGs

- Subnets are per-tier within a workload spoke: `snet-billing-app-prod-cc`,
  `snet-billing-data-prod-cc` (delegated subnets for Container Apps / App Service
  integration as the platform requires).
- **Every subnet gets an NSG** (`nsg-billing-app-prod-cc`), including "internal only"
  subnets — the subnet with no NSG is the one lateral movement will use. Rules allow the
  specific tier-to-tier flows the workload needs and deny the rest.
- NSGs, like everything else, are Terraform-managed; a portal-added allow rule is drift.

### Private endpoints for PaaS data services

In `stg` and `prod`, the following are reached exclusively via private endpoints, with
`public_network_access_enabled = false`:

- Azure SQL (`pep-billing-sql-prod-cc`)
- Storage accounts
- Key Vault
- The container registry `acrplatformsharedcc` (see
  [Container Registry](container-registry.md))

The private endpoint lives in the workload's data subnet, in the workload's RG, in the
workload's state — it shares the workload's lifecycle even when it points at a platform
resource like the registry.

### Private DNS

The `privatelink.*` zones (`privatelink.database.windows.net`,
`privatelink.blob.core.windows.net`, `privatelink.vaultcore.azure.net`,
`privatelink.azurecr.io`, …) are hosted **centrally in the hub**, in
`rg-platform-network-prod-cc`, and linked to every spoke. Workload stacks register
their endpoints' records into the central zones; they never create their own copies of
these zones. Split-brain private DNS — two zones for the same name with different
records — is the single most common cause of "works from one VNet, times out from
another", and central hosting makes it impossible.

### The dev exception

In `dev`, PaaS data services **may** use public endpoints with service-level firewall
rules (Azure service access plus the office/VPN egress IPs) instead of private
endpoints. This is a deliberate, documented tradeoff, not an oversight:

- Private endpoints have a real per-endpoint monthly cost plus data processing charges;
  across every service of every dev workload this is meaningful spend protecting
  non-production data.
- Private-only dev services complicate local development — engineers can't reach the dev
  database without VPN-to-VNet plumbing that itself must be built and operated.

The boundaries of the exception: dev-only, firewall rules still required (never
`0.0.0.0/0`), Entra-based authentication still required, and **no production or customer
data in dev, ever** — the data classification is what makes the relaxed posture
acceptable. The workload's Terraform expresses this as the `private_endpoints = false`
tfvar in dev only; the code paths are identical.

### Compute and egress

- **No public IPs on compute.** No `pip-*` attached to VMs, and no VM-shaped ingress at
  all in the normal case. Ingress arrives through fronting services (Front Door,
  Application Gateway, Container Apps ingress); administrative access uses Bastion or
  equivalent, not exposed ports.
- **Egress is a controlled path.** Prod workload egress flows via the hub (NAT/firewall
  as deployed), giving stable outbound IPs and a single place to restrict and log
  outbound destinations. At minimum, workloads must not rely on default
  internet-everything egress in prod.

## Examples

The billing production spoke:

```text
rg-platform-network-prod-cc                # platform-owned hub
├── vnet-platform-prod-cc                  # hub: DNS, egress, future hybrid
└── privatelink.database.windows.net          # central private DNS zones (+ blob, vault, acr)

rg-billing-prod-cc                         # workload-owned spoke
├── vnet-billing-prod-cc                   # peered to hub
│   ├── snet-billing-app-prod-cc           #   app tier (App Service integration)
│   │   └── nsg-billing-app-prod-cc
│   └── snet-billing-data-prod-cc          #   data tier (private endpoints)
│       └── nsg-billing-data-prod-cc
├── pep-billing-sql-prod-cc                # SQL private endpoint → central DNS record
└── sql-billing-prod-cc                    # public_network_access_enabled = false
```

The same workload in dev: no VNet at all — `sql-billing-dev-cc` with firewall rules,
Entra auth, synthetic data, and `private_endpoints = false` in `dev/terraform.tfvars`.

## Anti-patterns

| Anti-pattern | Why it fails |
|--------------|--------------|
| Public data endpoints in stg/prod "temporarily" | Temporary public endpoints are permanent public endpoints with better intentions. The standard is disabled, not guarded. |
| Workload-local `privatelink.*` DNS zones | Split-brain DNS: the same FQDN resolves differently per VNet, producing unreproducible connectivity failures. Central zones only. |
| Spoke-to-spoke peering for convenience | Turns the topology into a mesh where reachability is unknowable. Workloads integrate through published interfaces. |
| Subnets without NSGs ("it's internal") | Internal is where attackers move after the first foothold. Every subnet, every time. |
| A VNet for a dev workload with nothing private in it | Standing cost and address-space consumption with no consumer. |
| Hand-picked address spaces per workload | Overlaps surface years later, when peering or hybrid connectivity makes them unfixable without re-IP-ing. Central allocation only. |
| Public IPs on VMs for SSH/RDP | The most-scanned attack surface on the internet. Bastion or bust. |
| Private endpoints in stg skipped to save money | Deletes the parity that makes staging predictive — connectivity bugs will now debut in prod (see [Environments](environments.md)). |

## Tradeoffs

- **Private endpoints cost real money and we pay it in stg and prod.** Per-endpoint
  charges across every data service of every workload add up. We pay in prod because
  that is the point, and in stg because networking parity is exactly where staging earns
  its keep — and we decline to pay in dev, explicitly, as documented above.
- **The dev exception is a genuine posture reduction.** Dev data services are reachable
  from allowlisted internet IPs. We accept it because dev holds no real data and the
  alternative taxes every developer's daily loop with VPN plumbing; the exception is
  bounded by the no-real-data rule, which is therefore load-bearing and non-negotiable.
- **Central DNS and address allocation make the platform team a dependency.** A new
  spoke needs an address block and zone links before first deploy. We accept the
  coordination point because the failure modes it prevents (overlaps, split-brain DNS)
  are among the most expensive in networking to unwind.
- **Hub egress is a shared fate.** A misconfigured hub firewall rule can affect many
  workloads at once. We accept the concentration because the alternative — per-workload
  egress — means outbound control exists nowhere in practice.

## Related Standards

- [Environments](environments.md) — parity reasoning and the canonical environment set
- [Container Registry](container-registry.md) — the registry's private endpoint in prod
- [Resource Groups](resource-groups.md) — spokes in workload RGs, hub in the platform RG
- [Subscriptions](subscriptions.md) — where hub and spokes live
- [Azure Resource Naming](../naming/azure.md) — `vnet`/`snet`/`nsg`/`pep` naming
