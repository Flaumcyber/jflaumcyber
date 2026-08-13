# VNet, Subnets, and NSGs

The first lab in the Networking domain: a Virtual Network with two subnets, a Network Security Group with explicit inbound rules, and a real traffic test against a deployed VM to prove the rule actually worked, not just that it looked correct in the portal. Along the way, caught and fixed a genuine misconfiguration, a rule named Deny that was actually set to Allow, which turned out to be the most valuable part of the lab.

## What I Built

- rg-az104-vnet-nsg, a dedicated resource group in East US 2
- vnet-az104-demo (10.1.0.0/16) with snet-frontend (10.1.1.0/24) and snet-backend (10.1.2.0/24)
- nsg-az104-frontend with two explicit inbound rules: Allow-HTTP-Inbound (priority 100) and Deny-RDP-Inbound (priority 110)
- Associated nsg-az104-frontend with snet-frontend at the subnet level
- Deployed vm-az104-nsgtest into snet-frontend to test the rules against real traffic, redeploying once after the first attempt landed in the wrong region and an auto-generated VNet
- Discovered and corrected Deny-RDP-Inbound's Action field, which had been set to Allow despite the rule's name
- Confirmed a genuinely blocked RDP connection attempt after the fix
- Deleted rg-az104-vnet-nsg in full at the end of the session

## Architecture

```
                    rg-az104-vnet-nsg
                    Region: East US 2
                              |
                    vnet-az104-demo
                    10.1.0.0/16
                              |
                +-------------+--------------+
                |                             |
        snet-frontend                snet-backend
        10.1.1.0/24                  10.1.2.0/24
                |
        nsg-az104-frontend (associated at subnet level)
        100  Allow-HTTP-Inbound   TCP 80    Allow
        110  Deny-RDP-Inbound     TCP 3389  Deny
        65000-65500  Azure defaults
                |
        vm-az104-nsgtest (test VM, deleted after verification)
        RDP connection attempt: blocked, confirmed
```

| Resource | Role | Notes |
|----------|------|-------|
| vnet-az104-demo | Virtual Network | 10.1.0.0/16, two subnets |
| snet-frontend | Subnet | 10.1.1.0/24, NSG associated here |
| snet-backend | Subnet | 10.1.2.0/24, no NSG in this lab |
| nsg-az104-frontend | Network Security Group | Explicit allow and deny rules, subnet-level association |
| vm-az104-nsgtest | Test VM | Deployed to verify rules against real traffic, deleted after |

## Why These Choices

**Two subnets from the start, not one.** A single flat subnet doesn't demonstrate anything about network segmentation, the actual point of using subnets at all. Splitting into snet-frontend and snet-backend from the outset, even though this lab only builds on the front-end side, sets up the natural next step for a future lab: back-end-tier rules that only accept traffic from the front-end subnet rather than from the internet directly.

**Subnet-level NSG association instead of NIC-level.** Subnet-level association is the more common real-world pattern for a public-facing tier: one NSG governs every resource in that subnet consistently, rather than needing to remember to attach an NSG to every individual NIC as new VMs get added. It also made the later troubleshooting cleaner, since there was only one place the rule could actually be attached, not two possible locations to check.

**Deploying a real test VM instead of trusting the rule configuration alone.** A rule that looks correct in the portal is not the same thing as a rule that is actually enforced. This distinction turned out to matter directly: the Deny-RDP-Inbound rule was accidentally created with its Action field set to Allow, which the rule's name alone gave no indication of. Only a real connection attempt against a real VM exposed the mistake; reading the rule list again would not have.

## What I Learned (and Why It Matters)

- **A rule's name is documentation, not enforcement.** Deny-RDP-Inbound, set to Allow, still shows up in the rules list looking exactly like a working deny rule at a glance. Only the Action field actually determines behavior. This is the single most transferable lesson from this lab: always verify the Action/Access value directly, on every rule, every time, rather than trusting a well-named rule to mean what it says.
- **Configuration correctness and enforcement correctness are two different checks.** The NSG's rule list, the subnet association, and the priority ordering were all genuinely correct throughout this lab. The actual bug was invisible to every one of those checks and only surfaced through a real connection attempt. Reading configuration is necessary but not sufficient; testing against real traffic is what actually proves a security control works.
- **Multi-tab creation wizards can silently default to the wrong thing more than once in the same lab.** The test VM landed in the wrong region and an auto-generated VNet on the first attempt, purely from clicking through the wizard without checking each tab. This is the same category of mistake as the Windows VM resource-group mixup back in Lab 3.1, a pattern worth actively watching for in every future portal-based lab.
- **Subnet-level NSG association keeps troubleshooting simpler than NIC-level association would.** When the RDP block first appeared not to work, there was only one place to check the association (the subnet), not two possible locations (subnet and NIC) that could each be silently wrong or silently correct in a way that masked the real issue.

## Deployment Instructions

### Prerequisites
- Azure Free Account or pay-as-you-go subscription
- Azure CLI installed, or PowerShell with the Az module, if repeating this via CLI/PowerShell rather than the portal

### 1. Create the resource group and VNet
```
az group create --resource-group rg-az104-vnet-nsg --location eastus2

az network vnet create \
  --resource-group rg-az104-vnet-nsg \
  --name vnet-az104-demo \
  --address-prefix 10.1.0.0/16 \
  --subnet-name snet-frontend \
  --subnet-prefix 10.1.1.0/24

az network vnet subnet create \
  --resource-group rg-az104-vnet-nsg \
  --vnet-name vnet-az104-demo \
  --name snet-backend \
  --address-prefix 10.1.2.0/24
```

### 2. Create the NSG and rules
```
az network nsg create --resource-group rg-az104-vnet-nsg --name nsg-az104-frontend --location eastus2

az network nsg rule create \
  --resource-group rg-az104-vnet-nsg \
  --nsg-name nsg-az104-frontend \
  --name Allow-HTTP-Inbound \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 80

az network nsg rule create \
  --resource-group rg-az104-vnet-nsg \
  --nsg-name nsg-az104-frontend \
  --name Deny-RDP-Inbound \
  --priority 110 \
  --direction Inbound \
  --access Deny \
  --protocol Tcp \
  --destination-port-ranges 3389
```
Note: double-check `--access Deny` explicitly. This exact field, set incorrectly to Allow, was the real bug in this lab's portal build.

### 3. Associate the NSG with the subnet
```
az network vnet subnet update \
  --resource-group rg-az104-vnet-nsg \
  --vnet-name vnet-az104-demo \
  --name snet-frontend \
  --network-security-group nsg-az104-frontend
```

### 4. Deploy a test VM into snet-frontend and verify
Deploy any small VM into snet-frontend with no NIC-level NSG, then attempt an RDP connection to its public IP. Expected result: the connection should fail to establish entirely, not reach a login or certificate prompt.

To check the actual enforced rule set rather than just the NSG's configured list:
```
az network nic list-effective-nsg --resource-group rg-az104-vnet-nsg --name <nic-name>
```

### 5. Clean up
```
az group delete --name rg-az104-vnet-nsg --yes --no-wait
```

## Lessons Learned

This lab was designed to prove a network rule works, and it ended up proving something more useful: that proving a rule works and confirming a rule is configured are two genuinely different activities. Every configuration screen throughout this lab looked correct, the rule existed, was named clearly, was prioritized correctly, was associated with the right subnet, right up until a real connection attempt exposed that its Action field had been set backward. That gap between "looks right" and "is right" is exactly the kind of thing a certification exam question can test with a screenshot, and exactly the kind of thing a real production environment can get wrong silently for months without a real traffic test ever being run against it. Next time, I would test a rule's actual behavior immediately after creating it, before moving on to the next configuration step, rather than testing everything at the end.

---
*Part of the AZ-104 lab series. See also: [Azure Container Apps](README-az104-container-apps.md)*
