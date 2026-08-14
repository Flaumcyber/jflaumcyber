# Azure Bastion and Network Watcher

Deployed Azure Bastion and connected to a VM that has no public IP at all, no RDP client, no exposed port anywhere. Then used Network Watcher's IP flow verify tool to confirm, directly, that public-internet-facing RDP traffic to that VM is denied, while the live Bastion session had just worked seconds earlier. Highest cost-risk lab in the project so far, since Bastion has no pause option and bills a flat hourly rate the moment it exists.

## What I Built

- rg-az104-bastion-netwatcher and vnet-az104-bastion (10.3.0.0/16) with AzureBastionSubnet (10.3.0.0/26, the mandatory exact name and minimum size Azure requires) and snet-workload (10.3.1.0/24)
- bastion-az104-demo, Basic SKU, deployed into AzureBastionSubnet
- vm-az104-bastiontest, deployed into snet-workload concurrently with Bastion to save time, explicitly with Public IP set to None
- Confirmed the VM's Overview page shows no Public IP address at all, only a private IP (10.3.1.4)
- Connected via Connect > Bastion > Use Bastion, hitting a transient "Log in failed" error on the first attempt, succeeding on retry, with a live desktop session opening inside the browser
- Ran Network Watcher's IP flow verify against the VM's public-facing RDP path, receiving Access denied via the default DenyAllInBound rule
- Deleted rg-az104-bastion-netwatcher in full immediately after testing

## Architecture

```
                    rg-az104-bastion-netwatcher
                    Region: East US 2
                              |
                    vnet-az104-bastion (10.3.0.0/16)
                              |
                +-------------+--------------+
                |                             |
      AzureBastionSubnet              snet-workload
      10.3.0.0/26                     10.3.1.0/24
                |                             |
      bastion-az104-demo              vm-az104-bastiontest
      Basic SKU, public IP            NO public IP (confirmed: -)
      pip-bastion-az104-demo          Private IP: 10.3.1.4

      Live session: browser -> Bastion -> private VNet path -> VM
      IP flow verify (public IP -> VM, port 3389): Access denied, DenyAllInBound
      Two different results, same VM, proving the real traffic path is private
```

| Resource | Role | Notes |
|----------|------|-------|
| bastion-az104-demo | Azure Bastion, Basic SKU | ~$0.19/hr, no pause option, deleted same session |
| pip-bastion-az104-demo | Public IP for Bastion itself | Required by Bastion, distinct from any VM's networking |
| vm-az104-bastiontest | Test VM | No public IP, connected to purely via Bastion |

## Why These Choices

**Basic SKU, not Standard.** Standard tier adds native client support, VNet peering scope, and IP-based connection, none of which this lab's objective needed. Basic is the cheaper tier and fully sufficient to demonstrate the core concept: browser-based access to a VM with zero public IP exposure.

**A dedicated resource group, deleted the moment testing finished.** Bastion has no auto-shutdown option, unlike every VM in this project so far. It either exists and bills continuously (roughly $0.19/hour) or it's deleted; there's no middle state. Isolating it in its own resource group made teardown a single, unambiguous action rather than something that could accidentally get left running inside a group with other, lower-urgency resources.

**Testing IP flow verify against the public IP path, expecting and getting a Deny.** The point of this check wasn't to find an Allow rule, it was to prove that the VM's only real protection isn't a permissive NSG, it's the complete absence of a public attack surface. A Deny result against Bastion's public IP on port 3389, immediately after a real Bastion session had already worked, is stronger evidence of Bastion's actual architecture than an Allow result would have been.

## What I Learned (and Why It Matters)

- **Bastion eliminates the public attack surface entirely, it doesn't just add a layer in front of it.** Every prior VM lab in this project has had a public IP, protected by NSG rules or auto-shutdown timing. This VM has no public IP at all, confirmed both by the portal and by a direct Network Watcher check. There is no public port to scan, brute-force, or accidentally misconfigure open, because there is no public inbound path to the VM in the first place.
- **Azure Bastion has no pause state, unlike every VM in this project.** A VM can be stopped/deallocated to halt billing while keeping the resource for later. Bastion bills continuously from the moment it finishes deploying until the moment it's deleted, with no equivalent middle option. This changes the planning approach for any future lab involving Bastion: it needs a dedicated, uninterrupted session, not something to leave running between other tasks.
- **A denied result from a diagnostic tool can be the correct, expected outcome, not a sign something is broken.** The IP flow verify check testing the public IP path returned Access denied, and that was exactly right, since Bastion's real traffic never uses that path. Reading a Deny result in context, against what the traffic path actually is, mattered more than treating any Deny as automatically a problem to fix.
- **A first connection attempt failing doesn't necessarily indicate a real problem.** The initial "Log in failed" Bastion connection error resolved on a simple retry, likely a transient timing issue related to how recently the VM had finished provisioning. Worth remembering as a genuine, low-stakes possibility before assuming a deeper misconfiguration on the next unexpected error.

## Deployment Instructions

### Prerequisites
- Azure Free Account or pay-as-you-go subscription
- Azure CLI installed, or PowerShell with the Az module, if repeating this via CLI/PowerShell rather than the portal
- Budget for roughly $0.19/hour for the full duration Bastion exists, uninterruptible, with no pause option

### 1. Create the resource group and VNet with both subnets
```
az group create --resource-group rg-az104-bastion-netwatcher --location eastus2

az network vnet create \
  --resource-group rg-az104-bastion-netwatcher \
  --name vnet-az104-bastion \
  --address-prefix 10.3.0.0/16 \
  --subnet-name AzureBastionSubnet \
  --subnet-prefix 10.3.0.0/26

az network vnet subnet create \
  --resource-group rg-az104-bastion-netwatcher \
  --vnet-name vnet-az104-bastion \
  --name snet-workload \
  --address-prefix 10.3.1.0/24
```
Note: the AzureBastionSubnet name is mandatory and exact; Azure will not deploy Bastion into a subnet with any other name, and the minimum size is /26.

### 2. Deploy Bastion
```
az network public-ip create \
  --resource-group rg-az104-bastion-netwatcher \
  --name pip-bastion-az104-demo \
  --sku Standard

az network bastion create \
  --resource-group rg-az104-bastion-netwatcher \
  --name bastion-az104-demo \
  --public-ip-address pip-bastion-az104-demo \
  --vnet-name vnet-az104-bastion \
  --sku Basic
```
Expect this step to take 8-10+ minutes.

### 3. Deploy a test VM with no public IP
Deploy a VM into snet-workload with the public IP explicitly set to None. This can run concurrently with Step 2's Bastion deployment to save time.

### 4. Connect via Bastion
Portal: VM > Connect > Bastion > Use Bastion, enter credentials, Connect. This opens a live desktop session directly inside the browser tab.

### 5. Verify the public attack surface is genuinely closed
```
az network watcher test-ip-flow \
  --resource-group rg-az104-bastion-netwatcher \
  --vm vm-az104-bastiontest \
  --direction Inbound \
  --protocol TCP \
  --local <VM-private-IP>:3389 \
  --remote <Bastion-public-IP>:443
```
Expected output: Access denied via DenyAllInBound. This is correct; Bastion's real traffic path to the VM is private and never uses this route.

### 6. Clean up immediately
```
az group delete --name rg-az104-bastion-netwatcher --yes --no-wait
```
Do this the moment testing is complete. Bastion has no auto-shutdown and bills continuously until deleted.

## Lessons Learned

This lab's cost profile forced a different working style than every prior lab: deploying Bastion and the test VM concurrently rather than sequentially, moving directly from deployment confirmation into testing without pausing, and treating teardown as the immediate next step rather than something to circle back to. That urgency turned out to be a good forcing function, not just a constraint, since it meant every step had to be deliberate rather than exploratory. The most valuable single result was the IP flow verify check returning Access denied at exactly the moment it should have: proof that the live Bastion session worked specifically because it never touches the public, denied path at all. Next time working with Bastion, I would confirm the deletion has actually started before considering any Bastion-involving lab done, given how much more it costs per hour idle compared to a VM.

---
*Part of the AZ-104 lab series. See also: [VNet Peering](README-az104-vnet-peering.md)*
