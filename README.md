# AVNM Demo Lab

This is a lab to demonstrate and experiment with [Azure Virtual Network Manager](https://learn.microsoft.com/en-us/azure/virtual-network-manager/overview).

## Components
The lab consists of following elements:
- A set of VNETs:
  - Quantity is controlled by the `copies` parameter, (default: 20).
- Network Security Group:
  - Applied to the subnet `vmSubnet` in each VNET.
  - Contains an outbound rule denying traffic to private (RFC1918) ranges.
- Windows Server VMs:
  - In VNETs 0, 1, 2, and `copies`/2 (default: 10), `copies`/2+1 (11), `copies`/2+2 (12).
  - Each VM runs a basic webpage that returns the VM name.
- Bastion Hosts:
  - In VNETs 0 and `copies/2` (10).
- An AVNM instance `avnm`, scoped to the subscription.
- Network Groups:
  -  `production-networkgroup` contains VNETs 1 - `copies`/2-1 (9).
  -  `development-networkgroup` contains `copies`/2+1 (11) - `copies` (20).
- Network Configurations `production-hubspokemesh` and `development-hubspokemesh`, implementing a Hub&spoke with DirectConnectivity topology for the respective Network Groups.
- Security Configuration `secadminrule`,
  - Rule Collections `secadminrulecoll-production` and `secadminrulecoll-development`, each containing Allow rules, permitting communication within the respective Network Groups only (i.e. Production can only send traffic to Production, not to Development)
  - Rule Collection `no-internet` blocking outbound traffic from both Network Groups.
- VNET Gateways in VNETs 0 and `copies/2` (10) (the hubs of the Hub&spoke configurations for both Network Groups), with a VPN tunnel with BGP between them.

![image](images/avnmdemo.png)

## Lab Deployment

Log in to Azure Cloud Shell at https://shell.azure.com/ and select Bash.

Ensure Azure CLI and extensions are up to date:
  
`az upgrade --yes`
  
If necessary select your target subscription:
  
`az account set --subscription <Name or ID of subscription>`
  
Clone the  GitHub repository:

`git clone https://github.com/mddazure/avnm-demo`

Change directory:

`cd ./avnm-demo`

Create a new resource group:

`az group create --name {rgname} --location {location}`

Deploy the bicep template:

`az deployment group create -g {rgname} --template-file templates/main-hub-s2s.bicep`

## AVNM Configuration Deployment
The Network- and Security Configurations need to be deployed to take effect. This may be achieved from the Network Manager page in the portal, under Settings -> Deployments -> Deploy configurations. 
Select Configurations to deploy and target region:

![image](images/selectdeployment.png)

And Deploy:

![image](images/commitdeployment.png)

## Explore

### VM Effective Routes

List the effective routes for the VM in Hub VNET 0:

`az network nic show-effective-route-table --name VMNic-0 -g {rgname} -o table`

```Source                 State    Address Prefix    Next Hop Type          Next Hop IP
---------------------  -------  ----------------  ---------------------  -------------
Default                Active   10.0.0.0/24       VnetLocal
Default                Active   10.0.1.0/24       VNetPeering
Default                Active   10.0.2.0/24       VNetPeering
Default                Active   10.0.3.0/24       VNetPeering
Default                Active   10.0.4.0/24       VNetPeering
Default                Active   10.0.5.0/24       VNetPeering
Default                Active   10.0.6.0/24       VNetPeering
Default                Active   10.0.7.0/24       VNetPeering
Default                Active   10.0.8.0/24       VNetPeering
Default                Active   10.0.9.0/24       VNetPeering
VirtualNetworkGateway  Active   10.0.11.0/24      VirtualNetworkGateway  20.13.72.192
VirtualNetworkGateway  Active   10.0.10.158/32    VirtualNetworkGateway  20.13.72.192
VirtualNetworkGateway  Active   10.0.10.0/24      VirtualNetworkGateway  20.13.72.192
VirtualNetworkGateway  Active   10.0.12.0/24      VirtualNetworkGateway  20.13.72.192
VirtualNetworkGateway  Active   10.0.13.0/24      VirtualNetworkGateway  20.13.72.192
VirtualNetworkGateway  Active   10.0.15.0/24      VirtualNetworkGateway  20.13.72.192
VirtualNetworkGateway  Active   10.0.14.0/24      VirtualNetworkGateway  20.13.72.192
VirtualNetworkGateway  Active   10.0.16.0/24      VirtualNetworkGateway  20.13.72.192
VirtualNetworkGateway  Active   10.0.17.0/24      VirtualNetworkGateway  20.13.72.192
VirtualNetworkGateway  Active   10.0.18.0/24      VirtualNetworkGateway  20.13.72.192
VirtualNetworkGateway  Active   10.0.19.0/24      VirtualNetworkGateway  20.13.72.192
Default                Active   0.0.0.0/0         Internet
```
Observe routes are present to all peered (spoke) VNETs, and to all VNETs in the other network groups via the VNET Gateway:

List the effective routes for the VM in Spoke VNET 1:

`az network nic show-effective-route-table --name VMNic-1 -g {rgname} -o table`

```
Source                 State    Address Prefix                                                                                   Next Hop Type          Next Hop IP
---------------------  -------  -----------------------------------------------------------------------------------------------  ---------------------  -------------
Default                Active   10.0.1.0/24                                                                                      VnetLocal
Default                Active   10.0.0.0/24                                                                                      VNetPeering
Default                Active   10.0.9.0/24 10.0.8.0/24 10.0.7.0/24 10.0.6.0/24 10.0.5.0/24 10.0.4.0/24 10.0.3.0/24 10.0.2.0/24  ConnectedGroup
VirtualNetworkGateway  Active   10.0.11.0/24                                                                                     VirtualNetworkGateway  20.13.72.192
VirtualNetworkGateway  Active   10.0.10.158/32                                                                                   VirtualNetworkGateway  20.13.72.192
VirtualNetworkGateway  Active   10.0.10.0/24                                                                                     VirtualNetworkGateway  20.13.72.192
VirtualNetworkGateway  Active   10.0.12.0/24                                                                                     VirtualNetworkGateway  20.13.72.192
VirtualNetworkGateway  Active   10.0.13.0/24                                                                                     VirtualNetworkGateway  20.13.72.192
VirtualNetworkGateway  Active   10.0.15.0/24                                                                                     VirtualNetworkGateway  20.13.72.192
VirtualNetworkGateway  Active   10.0.14.0/24                                                                                     VirtualNetworkGateway  20.13.72.192
VirtualNetworkGateway  Active   10.0.16.0/24                                                                                     VirtualNetworkGateway  20.13.72.192
VirtualNetworkGateway  Active   10.0.17.0/24                                                                                     VirtualNetworkGateway  20.13.72.192
VirtualNetworkGateway  Active   10.0.18.0/24                                                                                     VirtualNetworkGateway  20.13.72.192
VirtualNetworkGateway  Active   10.0.19.0/24                                                                                     VirtualNetworkGateway  20.13.72.192
Default                Active   0.0.0.0/0                                                                                        Internet

```
Observe single entry for all VNETs in the Network Group with Next Hop Type ConnectedGroup, and routes for all VNETs in the other Network Group via the VNET Gateway in the Hub.

### Effective Security Rules

Listing Effective security rules in the portal, on a VM NIC in one of the Network Groups, shows separate entries for the NSG attached to the subnet and the Admin Rules programmed by AVNM.

#### NSG Rules
![image](images/nsg-rules.png)

#### Security Admin Rules
![image](images/admin-security-rules.png)

### Connectivity
Routes to other VMs exist, but outbound access is controlled by NSG- and the Security Admin Rules. 
- The Security Admin Rule collections contain Allow rules for each Network Groups' prefixes.
- Traffic permitted by Security Admin Rules with action Allow is subsequently evaluated by any NSGs attached to the subnet or NIC.
- The NSG attached to the subnet contains a rule blocking outbound communication to RFC1918 ranges.

#### From a Hub
Use Bastion Host in a Hub VNET to log in to the VM in that Hub.

Use `curl 10.0.{spoke number}.4` to check it is possible to connect to VMs in Spokes in the same Network Group, and in the other Group. Verify that there is internet access from the VM. 

It will not be possible to connect to any Spoke VM from a Hub. 

Reason: The Hubs are not controlled by AVNM as they are not part of any Network Group, so they do not have any Security Admin Rules applied. However, the normal NSG applied to the vmSubnet in all VNETs including Hubs has a rule blocking all outbound traffic to RFC1918 prefixes.

#### From a Spoke
Use Bastion Host in a Hub VNET to log in to the VM in a spoke connected to that Hub.

Use `curl 10.0.{spoke number}.4` to check whether is possible to connect to VMs in Spokes in the same Network Group, and in the other Group. Verify that there is no internet access from the VM. 

It will not be possible to any Spoke. 

Reason: The Spokes have both the Security Admin Rules and the NSG applied. The Security Admin Rules contain rules explicitly permitting outbound traffic to Spokes in the same Network Group. However, the Action on these rules is set to Allow - not Always Allow. This means that traffic permitted is still evaluated by the NSG, which blocks all outbound traffic to RFC1918 prefixec.

Now modify the Action to Always Allow and redeploy the configuration.

Check connectivity from Spoke to other Spokes again. It should now be possible to connect to Spokes in the same Network Group, as the Security Admin Rules allow outbound to the prefixes in the Group and are set to Always Allow.

### Monitoring and Logging

#### NSG Flows Logs

#### Network Watcher Diagnostics

## Live Demo Environment: Trusted / Non-trusted Mesh Topology

> The deployed lab in subscription `5d648a29-5d8e-4da2-a0f8-9a640b313144`, resource group `AVNM`
> (region `swedencentral`) has been re-configured from the original Production/Development
> Hub&Spoke-with-VPN pattern above into a **Trusted / Non-trusted mesh** pattern, matching the
> "global mesh with hub firewall as router" reference design. See `whats-new.md` for the full
> change log.

Summary of the current live topology:

- **Trusted network group** (`trusted-networkgroup`): a subset of VNETs from both the former
  Production and Development sides, connected with a **Mesh** connectivity configuration
  (`trusted-mesh`, global, direct peering, no hub). Any VM in this group can reach any other VM in
  the group directly — this is the "meshed" trust boundary.
- **Non-trusted Production network group** (`nontrusted-production-networkgroup`): remaining
  Production VNETs, still Hub&Spoke (`production-hubspokemesh`, hub = `anm-vnet-0` / `hubgw-0`).
  Spokes in this group can only reach each other through the hub; there is no mesh and no access to
  the Non-trusted Development group.
- **Non-trusted Development network group** (`nontrusted-development-networkgroup`): remaining
  Development VNETs, Hub&Spoke (`development-hubspokemesh`, hub = `anm-vnet-16`). Isolated the same
  way — no VPN, no mesh, no cross-group reachability.
- **Simulated on-premises**: a new VNet `anm-vnet-onprem` (10.100.0.0/24) with its own VPN Gateway
  `hubgw-onprem`, connected via a site-to-site VPN (BGP) to the Production hub gateway `hubgw-0`.
  This is the only VPN connection left in the lab — Development's gateway (`hubgw-16`) and the old
  Production↔Development VPN connections (`conn-high-low`/`conn-low-high`) were removed, since
  transitive routing between the environments is now demonstrated with the mesh instead of a VPN
  shortcut.
- **Security Admin rules**: `secadminrulecoll-production` and `secadminrulecoll-development` now
  scope their "allow within group" rules to the corresponding `nontrusted-*` network group;
  `secadminrulecollall` (deny-all-outbound) applies to all three groups; a new
  `secadminrulecoll-trusted` collection carries `AlwaysAllow` rules permitting intra-mesh traffic
  for `trusted-networkgroup`.

This lets a demo show, side by side: a fully meshed "trusted" zone with flat any-to-any
reachability, two isolated "non-trusted" hub-and-spoke zones with no lateral movement between them,
and a VPN-connected simulated on-premises network landing only in the Production hub — all managed
centrally from one AVNM instance.


