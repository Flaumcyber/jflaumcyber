# Availability Sets and VM Scale Sets

Deployed an Availability Set with two VMs to confirm real fault-domain separation, then a small VM Scale Set to compare the manual, fixed-membership high-availability model against a managed, elastic one. Hit three distinct real quota limits along the way and torn down the same session per cost discipline, since every resource here bills by the hour.

## What I Built

- rg-az104-ha-demo, a dedicated resource group for this high-availability lab, deleted in full at the end of the session
- avset-az104-demo, an Availability Set with 2 fault domains and 5 update domains
- Attempted Standard_B1s in East US 2, hit a capacity restriction (SkuNotAvailable, distinct from the NotAvailableForSubscription error in Lab 3.1), fell back to Standard_D2as_v7
- vm-az104-ha1 and vm-az104-ha2, deployed into the availability set, confirmed placed in different fault domains (0 and 1)
- vmss-az104-demo, a 2-instance Flexible-orchestration VM Scale Set, deployed without a public IP after hitting a public IP quota limit
- Worked through a regional vCPU quota limit by deleting the two Availability Set VMs once their screenshots were captured, freeing the quota the scale set needed

## Architecture

```
                    rg-az104-ha-demo
                    Region: East US 2
                              |
                +-------------+--------------+
                |                             |
        avset-az104-demo             vmss-az104-demo
        (Availability Set)            (VM Scale Set, Flexible)
                |                             |
        +-------+-------+             2 instances, no public IP
        |               |             sku.capacity: 2
   vm-az104-ha1    vm-az104-ha2
   Fault Domain 0  Fault Domain 1
   Standard_D2as_v7  Standard_D2as_v7
   (torn down after screenshots,
   freed vCPU quota for the scale set)
```

| Resource | Role | Notes |
|----------|------|-------|
| avset-az104-demo | Availability Set | 2 fault domains, 5 update domains |
| vm-az104-ha1 | HA VM | Fault domain 0, deleted after confirming placement |
| vm-az104-ha2 | HA VM | Fault domain 1, deleted after confirming placement |
| vmss-az104-demo | VM Scale Set | Flexible orchestration, 2 instances, no public IP |

## Why These Choices

**2 fault domains and 5 update domains for the Availability Set.** 2 platform fault domains is the maximum East US 2 supports for standard availability sets, and 5 update domains is the Azure default ceiling. Splitting exactly 2 VMs across 2 fault domains was the cleanest way to get a real, verifiable fault-domain split rather than a set that happened to have unused capacity.

**Deleting the HA VMs before deploying the Scale Set, not at the very end.** The Scale Set deployment failed on a regional vCPU quota (4 total, already fully used by the two Availability Set VMs) before it had provisioned anything billable. Rather than requesting a quota increase for a lab resource that would be deleted within the hour anyway, deleting the two VMs immediately, once their fault-domain screenshots were already captured, freed the quota cleanly and kept the lab moving without waiting on Microsoft support.

**No public IP on the Scale Set's load balancer.** A separate subscription-level quota, 3 public IPs total, was already exhausted by an orphaned IP left over from Lab 3.1's resource-group mixup. Rather than wait on that leftover group's slow deletion to free a slot, deploying the Scale Set with `--public-ip-address ""` sidestepped the quota entirely. The lab's actual goal, comparing instance distribution across the two HA models, doesn't require external reachability, so this cost nothing in terms of what the lab needed to demonstrate.

## What I Learned (and Why It Matters)

- **Not every quota error is the same kind of limit.** This lab alone hit three distinct types in one session: a per-size capacity restriction (SkuNotAvailable), a resource-type count quota (3 public IPs total), and a regional vCPU quota (4 cores total across the whole subscription). Each required a different diagnosis and a different fix, and treating them as interchangeable "quota errors" would have wasted time on the wrong solution.
- **Availability Sets guarantee fault-domain separation and expose it cleanly; Flexible-mode Scale Sets don't expose it the same way.** Querying fault domain per instance worked immediately and predictably for the Availability Set VMs via `az vm get-instance-view`. The equivalent data for Flexible-orchestration Scale Set instances wasn't reliably available through several CLI commands or the portal's Instances blade in this session, a real, documented tooling gap rather than a configuration mistake.
- **Orphaned resources from earlier labs can silently block unrelated later labs.** A leftover public IP from Lab 3.1's resource-group mixup consumed the exact quota slot this lab's scale set needed. Cost hygiene between labs isn't just about avoiding unnecessary billing, it also prevents quota exhaustion from stacking up unnoticed.
- **Deleting a VM to solve a quota problem, rather than requesting a quota increase, was the right call for a lab environment.** The two Availability Set VMs had already served their purpose (the fault-domain screenshots were captured); keeping them running only to avoid a delete-and-redeploy step would have meant either waiting on a support ticket or paying for idle compute that added nothing further to the lab.

## Deployment Instructions

### Prerequisites
- Azure Free Account or pay-as-you-go subscription
- Azure CLI installed, or PowerShell with the Az module
- Note: check your subscription's regional vCPU quota and public IP quota before starting if you plan to run an Availability Set and a Scale Set concurrently, since a small default subscription can hit both limits in a single lab like this one did

### 1. Create the resource group and Availability Set
```
az group create --resource-group rg-az104-ha-demo --location eastus2

az vm availability-set create \
  --resource-group rg-az104-ha-demo \
  --name avset-az104-demo \
  --platform-fault-domain-count 2 \
  --platform-update-domain-count 5
```

### 2. Deploy two VMs into the Availability Set
```
az vm create \
  --resource-group rg-az104-ha-demo \
  --name vm-az104-ha1 \
  --availability-set avset-az104-demo \
  --image Canonical:ubuntu-24_04-lts:server:latest \
  --size Standard_D2as_v7 \
  --admin-username azureuser \
  --generate-ssh-keys

az vm create \
  --resource-group rg-az104-ha-demo \
  --name vm-az104-ha2 \
  --availability-set avset-az104-demo \
  --image Canonical:ubuntu-24_04-lts:server:latest \
  --size Standard_D2as_v7 \
  --admin-username azureuser \
  --generate-ssh-keys
```

### 3. Confirm fault domain placement
```
az vm get-instance-view --resource-group rg-az104-ha-demo --name vm-az104-ha1 --query "instanceView.platformFaultDomain"
az vm get-instance-view --resource-group rg-az104-ha-demo --name vm-az104-ha2 --query "instanceView.platformFaultDomain"
```
Expected output: different numbers (0 and 1) for each VM

### 4. Delete the Availability Set VMs to free regional vCPU quota
```
az vm delete --resource-group rg-az104-ha-demo --name vm-az104-ha1 --yes
az vm delete --resource-group rg-az104-ha-demo --name vm-az104-ha2 --yes
```

### 5. Deploy the Scale Set
```
az vmss create \
  --resource-group rg-az104-ha-demo \
  --name vmss-az104-demo \
  --image Canonical:ubuntu-24_04-lts:server:latest \
  --instance-count 2 \
  --vm-sku Standard_D2as_v7 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --upgrade-policy-mode manual \
  --public-ip-address ""
```
Note: omit `--public-ip-address ""` if your subscription has public IP quota available and you want external reachability.

### 6. Confirm instance count
```
az vmss show --resource-group rg-az104-ha-demo --name vmss-az104-demo --query "sku.capacity"
```
Expected output: 2

### 7. Clean up
```
az group delete --name rg-az104-ha-demo --yes --no-wait
```

## Lessons Learned

This lab ran into more friction than any Compute lab so far, but almost none of it was about Availability Sets or Scale Sets conceptually, it was about a small Azure Free Account subscription bumping against three separate quota ceilings in quick succession, plus a genuine tooling gap around Flexible-orchestration Scale Set instances. None of that was a mistake in the lab design; it's exactly the kind of real-world troubleshooting an administrator actually does. The most transferable lesson is diagnostic: reading the specific error code and quota name (SkuNotAvailable vs. NotAvailableForSubscription vs. ResourceCountExceedsLimitDueToTemplate vs. a regional core quota) mattered more than any single fix, since each pointed to a genuinely different constraint requiring a different response.

---
*Part of the AZ-104 lab series. See also: [VM Deployment: Linux and Windows](README-az104-vm-deployment.md), [ARM Templates vs. Bicep](README-az104-arm-bicep.md)*
