# What's New

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
