# What's New

## Hub1/Hub2 × Trusted/Non-trusted 4-Group Redesign + Global Backup Mesh (live demo environment)

Second redesign pass on the same deployed lab (sub `5d648a29-5d8e-4da2-a0f8-9a640b313144`, rg
`AVNM`, network manager `AVNM-Demo`, region `swedencentral`), replacing the single Trusted/Non-trusted
mesh (below) with a 4-group topology that mirrors a reference slide showing two hubs, each split
into meshed ("trusted") and hub-and-spoke ("non-trusted") zones, plus a cross-hub backup mesh.

### Added
- **`hubfirewall-16`**: a second Azure Firewall Premium, deployed into Hub2 (`anm-vnet-16`), so each
  hub now routes its own non-trusted spoke↔spoke traffic independently (firewall-as-router pattern
  in both hubs, not just Hub1).
- 4 new network groups replacing the previous single `trusted-networkgroup`:
  `trusted-hub1-networkgroup` (anm-vnet-2,8,9,10), `nontrusted-hub1-networkgroup` (anm-vnet-1,3-7,11-15),
  `trusted-hub2-networkgroup` (anm-vnet-18,25,26,27), `nontrusted-hub2-networkgroup`
  (anm-vnet-17,19-24,28-31).
- `global-backup-networkgroup` (anm-vnet-2 + anm-vnet-18, one trusted VNet per hub) with a new Mesh
  connectivity config `global-backup-mesh` — a cross-hub / cross-region DR backup link between the
  two trusted zones.
- Updated architecture diagram (`avnm-architecture.excalidraw` / `.png`, session artifacts)
  reflecting the full 4-group + dual-firewall + on-prem-VPN + global-backup-mesh topology.

### Changed
- `production-hubspokemesh` and `development-hubspokemesh` retargeted (via repeated
  `--applies-to-groups` CLI flags — see note below) to their respective trusted+non-trusted hub
  group pairs.
- `secadminrulecoll-trusted` retargeted to both `trusted-hub1-networkgroup` and
  `trusted-hub2-networkgroup`; `secadminrulecollall` retargeted to all 4 hub network groups;
  `secadminrulecoll-production`/`-development` retargeted to `nontrusted-hub1-networkgroup` /
  `nontrusted-hub2-networkgroup`.
- SecurityAdmin configuration redeployed (`post-commit --commit-type SecurityAdmin`) to
  `swedencentral` after the rule collection updates.

### Removed
- Old single-group `trusted-networkgroup`, `nontrusted-production-networkgroup`,
  `nontrusted-development-networkgroup` network groups (superseded by the 4 hub-scoped groups).
- Old `trusted-mesh` connectivity config (superseded by `global-backup-mesh`, scoped to only the two
  cross-hub trusted VNETs rather than the whole trusted set).

### Note: `az network manager` CLI quirks discovered during this change
- `security-admin-config rule-collection update --applies-to-groups` does **not** accept multiple
  `network-group-id=X` values space-separated within a single flag instance — only the last one is
  kept. Fix: repeat the whole flag once per group, e.g.
  `--applies-to-groups network-group-id=A --applies-to-groups network-group-id=B`.
- `connect-config delete` takes `--configuration-name` (not `--name`).
- Deleting a network group requires `az network manager group delete` (not `network-group delete`).
- A network group can't be deleted while still referenced by a *deployed* config version, even if
  the latest saved config no longer references it — redeploy (post-commit) both Connectivity and
  SecurityAdmin configs first.

## Trusted / Non-trusted Mesh Redesign (live demo environment)

Applied directly to the deployed lab (sub `5d648a29-5d8e-4da2-a0f8-9a640b313144`, rg `AVNM`,
region `swedencentral`) to align it with a "global mesh + hub firewall as router" reference demo.

### Added
- `anm-vnet-onprem` VNet (10.100.0.0/24) with a `GatewaySubnet`, simulating an on-premises network.
- `hubgw-onprem` VPN Gateway (VpnGw1AZ, RouteBased, ASN 65010) + `hubgw-onprem-pip` public IP.
- `trusted-networkgroup` network group with a new **Mesh** connectivity configuration
  (`trusted-mesh`) providing direct any-to-any connectivity for its members.
- `nontrusted-production-networkgroup` and `nontrusted-development-networkgroup` network groups,
  replacing the old `production-networkgroup`/`development-networkgroup`.
- `secadminrulecoll-trusted` security admin rule collection with `AlwaysAllow` rules
  (`allowtrustedmesh-in`/`allowtrustedmesh-out`) permitting intra-mesh traffic.
- `conn-prod-onprem` / `conn-onprem-prod` VPN connections between `hubgw-0` (Production hub) and
  the new `hubgw-onprem` gateway, so the simulated on-premises network only lands in Production.

### Changed
- `production-hubspokemesh` and `development-hubspokemesh` connectivity configs retargeted from the
  old network groups to `nontrusted-production-networkgroup` / `nontrusted-development-networkgroup`.
- `secadminrulecoll-production`, `secadminrulecoll-development`, `secadminrulecollall` retargeted to
  the new `nontrusted-*` / `trusted-*` groups.
- `allowwithinprod` / `allowwithindev` rule address prefixes updated from the old `NetworkGroup`
  references to the new `nontrusted-production-networkgroup` / `nontrusted-development-networkgroup`.

### Removed
- Old `production-networkgroup` and `development-networkgroup` network groups.
- `allowprodtodev` / `allowdevtoprod` cross-environment security admin rules (no longer needed — the
  mesh now provides controlled cross-environment reachability for trusted members only).
- `conn-high-low` / `conn-low-high` VPN connections (the original Production↔Development VPN link).
- `hubgw-16` VPN Gateway (Development's gateway) — Development is fully disconnected from VPN;
  only Production keeps a gateway, now facing the simulated on-premises network instead of
  Development.

### Why
The original lab used a VPN tunnel between the Production and Development hubs to achieve
cross-group reachability. Since AVNM connectivity configurations now support hub route-tables /
mesh patterns, the same (and better) demo of transitive routing and centrally managed connectivity
can be shown without a VPN between the two environments — freeing the VPN gateway to instead
represent a realistic on-premises connection, landing in the Production ("trusted-facing") hub only.
