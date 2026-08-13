# VNet Peering

Built two Virtual Networks and peered them together, then deployed a VM in each to prove real traffic could actually flow across the peering connection, private IP to private IP, no public internet path involved. Along the way, discovered that Azure's own portal field labels for "local" and "remote" peering names don't map the way you'd assume, and that a first connectivity test failing doesn't automatically mean the network path is broken.

## What I Built

- rg-az104-vnet-peering, vnet-az104-hub (10.1.0.0/16, snet-hub), and vnet-az104-spoke (10.2.0.0/16, snet-spoke), both in East US 2
- Established peering from vnet-az104-hub's Add Peering form, which created both directions (peer-hub-to-spoke and peer-spoke-to-hub) in a single action, confirmed by a toast notification naming both sides explicitly
- Confirmed Peering state: Connected and Peering sync status: Fully Synchronized on both sides
- Deployed vm-az104-peertest1 into snet-hub and vm-az104-peertest2 into snet-spoke, checking Region and Virtual network/subnet explicitly on each Review + create summary before deploying
- Attempted a ping from vm-az104-peertest1 to vm-az104-peertest2's private IP, which failed 100%, a red herring caused by Windows' default ICMP blocking rather than a peering failure
- Ran Test-NetConnection against port 3389 instead, confirming TcpTestSucceeded: True, private IP to private IP
- Deleted rg-az104-vnet-peering in full at the end of the session

## Architecture

```
                    rg-az104-vnet-peering
                    Region: East US 2
                              |
                +-------------+--------------+
                |                             |
        vnet-az104-hub               vnet-az104-spoke
        10.1.0.0/16                  10.2.0.0/16
                |                             |
        snet-hub (10.1.1.0/24)       snet-spoke (10.2.1.0/24)
                |                             |
        vm-az104-peertest1 <----- peered ----->  vm-az104-peertest2
        10.1.1.4                                 10.2.1.4

        Peering: bidirectional, Connected, Fully Synchronized
        Verified: Test-NetConnection 10.2.1.4:3389 from VM1
        TcpTestSucceeded: True (private IP to private IP)
```

| Resource | Role | Notes |
|----------|------|-------|
| vnet-az104-hub | Virtual Network | 10.1.0.0/16, snet-hub |
| vnet-az104-spoke | Virtual Network | 10.2.0.0/16, snet-spoke |
| peer-hub-to-spoke / peer-spoke-to-hub | Peering connections | Both created from one portal action, both Connected |
| vm-az104-peertest1 | Test VM | In snet-hub, source of the connectivity test |
| vm-az104-peertest2 | Test VM | In snet-spoke, target of the connectivity test |

## Why These Choices

**Two brand-new VNets instead of recreating the last lab's VNet.** vnet-az104-demo from the NSG lab no longer existed, since that lab's whole resource group was deleted at the end of its session per cost discipline. Recreating it just to peer against it would have added a step that taught nothing new about peering itself. Two purpose-built VNets, hub and spoke, kept this lab focused on the actual new concept.

**Deliberately non-overlapping address spaces (10.1.0.0/16 and 10.2.0.0/16).** Azure will refuse to establish a peering connection at all if the two VNets' address ranges overlap. Choosing clearly distinct ranges up front avoided hitting that failure mode entirely, rather than discovering it mid-lab the way earlier labs discovered region or quota issues.

**Testing TCP connectivity on port 3389 instead of trusting a ping.** The first connectivity test used ICMP (ping) and returned 100% packet loss, which looked like a peering failure but wasn't. Windows Server blocks inbound ICMP by default regardless of network path. Switching to Test-NetConnection against port 3389, a port already known to be open since RDP itself uses it, gave an unambiguous, real application-layer proof of connectivity instead of a result confounded by an unrelated default firewall setting.

## What I Learned (and Why It Matters)

- **Peering requires explicit configuration on both sides, but a single portal action can create both.** Starting the Add Peering flow from one VNet's blade genuinely creates both peering objects at once, confirmed by a toast notification naming both. This is convenient, but the underlying requirement, that both directions must exist and be enabled, is the same whether one portal click or two separate CLI commands create them. Terraform and Bicep make this unmissable since they require two distinct resource blocks.
- **Portal field labels for "local" and "remote" don't always map to the VNet you'd assume.** Starting the peering wizard from vnet-az104-hub, the "Local virtual network summary" section actually named the peering object that ended up on vnet-az104-hub's own side, while the object that logically reads as "from the hub's perspective" landed on the spoke's side instead. Worth verifying which peering object actually exists where, rather than assuming from the field's position in the form.
- **A failed connectivity test doesn't automatically mean the network path is broken.** ICMP being blocked by Windows' default firewall produced a result indistinguishable, at a glance, from a genuine peering failure. Reaching for a second, more targeted test (TCP on a known-open port) rather than concluding "peering doesn't work" from the first failed attempt was the actual correct troubleshooting step.
- **Testing with private IPs specifically, not public IPs, is what actually proves peering.** A successful connection over public IPs would prove nothing about the peering connection at all, since that traffic never needs to use it. The private-IP-to-private-IP test is the only one that isolates and confirms the peering path specifically.

## Deployment Instructions

### Prerequisites
- Azure Free Account or pay-as-you-go subscription
- Azure CLI installed, or PowerShell with the Az module, if repeating this via CLI/PowerShell rather than the portal

### 1. Create the resource group and both VNets
```
az group create --resource-group rg-az104-vnet-peering --location eastus2

az network vnet create \
  --resource-group rg-az104-vnet-peering \
  --name vnet-az104-hub \
  --address-prefix 10.1.0.0/16 \
  --subnet-name snet-hub \
  --subnet-prefix 10.1.1.0/24

az network vnet create \
  --resource-group rg-az104-vnet-peering \
  --name vnet-az104-spoke \
  --address-prefix 10.2.0.0/16 \
  --subnet-name snet-spoke \
  --subnet-prefix 10.2.1.0/24
```

### 2. Establish peering, both directions
```
az network vnet peering create \
  --resource-group rg-az104-vnet-peering \
  --name peer-hub-to-spoke \
  --vnet-name vnet-az104-hub \
  --remote-vnet vnet-az104-spoke \
  --allow-vnet-access

az network vnet peering create \
  --resource-group rg-az104-vnet-peering \
  --name peer-spoke-to-hub \
  --vnet-name vnet-az104-spoke \
  --remote-vnet vnet-az104-hub \
  --allow-vnet-access
```
Note: both commands are required. Skipping either one leaves the connection one-directional even though the first side will show as configured.

### 3. Verify both sides show Connected
```
az network vnet peering list --resource-group rg-az104-vnet-peering --vnet-name vnet-az104-hub --output table
az network vnet peering list --resource-group rg-az104-vnet-peering --vnet-name vnet-az104-spoke --output table
```

### 4. Deploy a VM in each VNet and test real connectivity
Deploy a small VM into snet-hub and another into snet-spoke, each with RDP allowed. From inside the first VM, test connectivity to the second VM's private IP:
```
Test-NetConnection -ComputerName <VM2-private-IP> -Port 3389
```
Expected output: TcpTestSucceeded : True. A plain ping to the same address may fail even when peering is working correctly, since Windows blocks inbound ICMP by default; use the TCP test as the authoritative result.

### 5. Clean up
```
az group delete --name rg-az104-vnet-peering --yes --no-wait
```

## Lessons Learned

This lab's real lesson wasn't about peering configuration itself, that part went smoothly. It was about the gap between a diagnostic test failing and a system actually being broken. The failed ping looked exactly like what a real peering misconfiguration would look like, and treating it as inconclusive rather than as a final answer, then reaching for a more targeted test, was the correct instinct rather than an obvious one. That distinction, between "this test failed" and "this system is broken," is worth carrying into every future networking lab in this series, since Azure's default behaviors (blocked ICMP, silent wizard defaults, swapped field labels) keep producing results that look like failures without actually being the failure they appear to be.

---
*Part of the AZ-104 lab series. See also: [VNet, Subnets, and NSGs](README-az104-vnet-nsg.md)*
