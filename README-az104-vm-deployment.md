# VM Deployment: Linux and Windows

The first lab in the Compute domain. Deployed a Linux and a Windows VM into a shared resource group, hit a real subscription-level regional quota restriction along the way, and configured auto-shutdown on both immediately rather than as an afterthought.

## What I Built

- rg-az104-compute, the resource group for the entire Compute domain
- Attempted Standard_B1s in East US, hit NotAvailableForSubscription on every size tried across both Burstable and D-series families
- Switched the deployment region to East US 2, where Standard_D2as_v7 deployed cleanly
- vm-az104-linux (Ubuntu Server 24.04) and vm-az104-windows (Windows Server 2022 Datacenter), both Standard_D2as_v7
- Auto-shutdown set to 9:00 PM Central on both, immediately after each deployment

## Architecture

```
                    rg-az104-compute
                    Region: East US 2 (not East US)
                              |
                +-------------+--------------+
                |                             |
          vm-az104-linux              vm-az104-windows
          Ubuntu Server 24.04          Windows Server 2022
          Standard_D2as_v7             Standard_D2as_v7
          Zone 1                       Zone 1
          auto-shutdown: 9 PM CT       auto-shutdown: 9 PM CT
                                        (landed in an auto-created
                                        resource group at first,
                                        moved into rg-az104-compute)
```

| Resource | Role | Notes |
|----------|------|-------|
| vm-az104-linux | Linux VM | Ubuntu 24.04, D2as_v7, East US 2 |
| vm-az104-windows | Windows VM | Windows Server 2022 Datacenter, D2as_v7, corrected into rg-az104-compute after an auto-generated resource group mixup |

## Why These Choices

**Standard_D2as_v7, once B-series proved unavailable.** The original plan was Standard_B1s, a Burstable-tier size inside Azure's free 750-hour monthly allowance. When the entire B-series family returned NotAvailableForSubscription in East US, D2as_v7 became the practical choice: genuinely available, still small at 2 vCPUs, at a real but low cost of roughly $0.09/hour rather than free.

**East US 2 instead of continuing to troubleshoot East US.** Once both a Burstable size and a D-series size returned the identical error in East US, that pattern pointed to a regional restriction on the subscription itself, not a size-specific quota problem. Switching regions tested that theory directly and resolved every size issue at once.

**Auto-shutdown configured immediately, not after the fact.** Every VM here bills by the hour the moment it exists, unlike almost everything built in the Identities, Governance, and Storage domains. Setting auto-shutdown as the literal next action after deployment turns a habit into a default rather than something to remember later.

## What I Learned (and Why It Matters)

- **NotAvailableForSubscription can mean a regional restriction, not just a per-size quota gap.** Testing a second, unrelated VM family and getting the identical error in the same region was the actual diagnostic signal.
- **The VM creation form's Resource group field can silently default to an auto-generated name.** Same root cause as the storage account mixup in Lab 2.1. Azure's Move feature exists precisely because this is a common, recoverable mistake.
- **A VM's billing clock starts at deployment, not at first use.** Configuring auto-shutdown as the literal next click after Create is the only reliable way to guarantee it happens before an idle VM racks up unnecessary hours.

## Deployment Instructions

### Prerequisites
- Azure Free Account or pay-as-you-go subscription
- Azure CLI installed, or PowerShell with the Az module
- Note: if East US returns NotAvailableForSubscription on VM sizes, try East US 2 before troubleshooting individual sizes further

### 1. Create the resource group
```
az group create --name rg-az104-compute --location eastus2
```

### 2. Deploy both VMs
```
az vm create \
  --resource-group rg-az104-compute \
  --name vm-az104-linux \
  --image Canonical:ubuntu-24_04-lts:server:latest \
  --size Standard_D2as_v7 \
  --admin-username azureuser \
  --generate-ssh-keys

az vm create \
  --resource-group rg-az104-compute \
  --name vm-az104-windows \
  --image MicrosoftWindowsServer:WindowsServer:2022-datacenter-g2:latest \
  --size Standard_D2as_v7 \
  --admin-username azureadmin \
  --admin-password '<a-strong-password>'
```

### 3. Set auto-shutdown on both
```
az vm auto-shutdown --resource-group rg-az104-compute --name vm-az104-linux --time 2100
az vm auto-shutdown --resource-group rg-az104-compute --name vm-az104-windows --time 2100
```

### 4. Verify
```
az vm show --resource-group rg-az104-compute --name vm-az104-linux --query powerState
az vm show --resource-group rg-az104-compute --name vm-az104-windows --query powerState
```
Expected output: both show "VM running"

### 5. Clean up
```
az group delete --name rg-az104-compute --yes --no-wait
```

## Lessons Learned

This lab took considerably longer than expected, but not because of anything conceptually difficult, purely because of a real-world quota restriction that had nothing to do with the actual VM deployment skill being practiced. That turned out to be a useful lesson on its own: troubleshooting infrastructure availability issues, reading error codes precisely, and knowing when to switch regions rather than keep negotiating with the same blocked resource, is as much a real administrator skill as the deployment steps themselves. Next time, I would check regional availability before starting a Compute lab rather than discovering it mid-deployment.

---
*Part of the AZ-104 lab series. See also: [Lifecycle Management and Backup](README-az104-storage-lifecycle-backup.md)*
