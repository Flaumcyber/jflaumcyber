# Storage Account Fundamentals

Building a storage account from scratch with LRS redundancy, creating a blob container, uploading a real file into it, and changing that file's access tier from Hot to Cool. The first lab in the Storage domain, and the foundation the SAS token and lifecycle labs build on top of.

## What I Built

- rg-az104-storage resource group
- A Standard performance, LRS-redundant storage account, az104storagejf
- A private blob container, az104-storage-blob1
- An uploaded file, baby-yoda-2021.jpeg, which landed in the Hot tier by default
- The blob's access tier changed from Hot to Cool

## Architecture

```
                    rg-az104-storage
                              |
                    Storage account: az104storagejf
                    Performance: Standard
                    Replication: LRS
                              |
                    Container: az104-storage-blob1
                    access level: private
                              |
                    Blob: baby-yoda-2021.jpeg
                    7.32 KiB, Block blob
                    Hot -> Cool tier change
```

| Resource | Role | Notes |
|----------|------|-------|
| az104storagejf | Storage account | Standard tier, LRS redundancy, StorageV2 |
| az104-storage-blob1 | Blob container | Private access level |
| baby-yoda-2021.jpeg | Blob | 7.32 KiB, moved from Hot to Cool tier |

## Why These Choices

**Redundancy: why LRS.** Azure offers four main redundancy tiers. LRS copies data three times within a single datacenter, protecting against hardware failure but not a datacenter or regional outage. ZRS spreads those copies across availability zones in the same region. GRS adds asynchronous replication to a paired secondary region. GZRS combines both for the highest durability and cost. LRS was the right choice here because this is disposable lab data rebuilt from Terraform and Bicep definitions checked into this repo. If the datacenter hosting it disappeared, the fix is redeploying the code, not restoring from a geo-replicated copy. Paying for ZRS or GRS on data that costs nothing to regenerate would be optimizing for a failure mode that doesn't threaten anything important here.

**Access tier: why Hot to Cool, not Archive.** Hot has the highest storage cost and lowest access cost, for data touched frequently. Cool lowers storage cost in exchange for higher access cost and an expected minimum retention around 30 days, for data accessed occasionally but not urgently. Archive drops storage cost to the lowest point but requires an explicit rehydration step, taking hours, before the data can be read again at all. Cool was the right comparison point for this lab because the goal was to demonstrate the tier mechanism and its immediate effect. Cool still allows instant reads, so the change was visible and verifiable in the same session, unlike Archive's rehydration wait.

## What I Learned (and Why It Matters)

- **Always verify the resource group during creation, not after.** The portal's create-resource flow lets the resource group field default to whatever was last used, which is exactly how this storage account initially ended up in rg-az104-identity instead of rg-az104-storage. A quick glance at that field before clicking Create would have caught it immediately.
- **Resources can be moved between resource groups without being rebuilt.** Azure's Move feature handled the correction cleanly, with no data loss, no downtime, and no need to delete and recreate the storage account from scratch.
- **New blobs default to whatever access tier the storage account has set,** which was Hot here. Cool and Archive are opt-in per blob, or set as the account default, not something Azure assumes you want for older or less-accessed data.

## Deployment Instructions

### Prerequisites
- Azure Free Account or pay-as-you-go subscription
- Azure CLI installed, or PowerShell with the Az module

### 1. Create the resource group and storage account
```
az group create --name rg-az104-storage --location eastus

az storage account create \
  --name az104storagejf \
  --resource-group rg-az104-storage \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2
```

### 2. Create a container and upload a file
```
az storage container create \
  --name az104-storage-blob1 \
  --account-name az104storagejf \
  --public-access off

az storage blob upload \
  --account-name az104storagejf \
  --container-name az104-storage-blob1 \
  --name baby-yoda-2021.jpeg \
  --file ./baby-yoda-2021.jpeg
```

### 3. Change the access tier
```
az storage blob set-tier \
  --account-name az104storagejf \
  --container-name az104-storage-blob1 \
  --name baby-yoda-2021.jpeg \
  --tier Cool
```

### 4. Verify
```
az storage blob show \
  --account-name az104storagejf \
  --container-name az104-storage-blob1 \
  --name baby-yoda-2021.jpeg \
  --query properties.blobTier
```
Expected output: "Cool"

### 5. Clean up
```
az group delete --name rg-az104-storage --yes --no-wait
```

## Lessons Learned

The resource group mixup was the most useful part of this lab, even though it wasn't planned. It's a very easy mistake to make, since the create-resource dialog doesn't force you to actively confirm the resource group the way it does for riskier fields like deletion. Next time, I would make a habit of checking the resource group field first, before filling in anything else, rather than last.

This mistake happened in the same resource group, rg-az104-identity, that I nearly deleted by accident earlier in this same study session. That coincidence was the push to stop treating the lock from Lab 1.4 as just a demonstration. It's now permanently reapplied as CanNotDelete, this time as actual protection for a resource group holding four completed labs worth of work, not something removed again once the lab was done.

---
*Part of the AZ-104 lab series. See also: [Entra ID users and groups](README-az104-identity-governance.md)*
