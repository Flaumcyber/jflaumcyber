# Lifecycle Management and Backup

Automating what Lab 2.1 did manually, and proving what Lab 2.1's soft delete setting actually does. A lifecycle management rule watches blobs based on age and retiers them to Cool automatically. Soft delete turns an accidental deletion into a recoverable event instead of a permanent one. The final lab in the Storage domain.

## What I Built

- A lifecycle management rule, move-to-cool-after-30-days, on az104storagejf: any block blob not modified in 30 days automatically moves to the Cool tier
- Reviewed both the List View and the raw Code View JSON definition Azure generated from that rule
- A dedicated test blob, lifecycle-restore-test.txt, kept separate from blobs already referenced in Labs 2.1 and 2.2
- Deleted it, confirmed a soft-deleted state with a visible retention countdown (6 of 7 days remaining), then restored it
- Confirmed it was back in the active blob list, fully recovered

## Architecture

```
                    rg-az104-storage
                              |
                    az104storagejf
                              |
                +-------------+--------------+
                |                             |
          Lifecycle Rule                Soft Delete (7 days)
          move-to-cool-after-30-days    already enabled from Lab 2.1
          condition: not modified          |
          in 30 days                    lifecycle-restore-test.txt
          action: tier to Cool           uploaded -> deleted -> 
          evaluates once daily          restored within retention window
```

| Resource | Role | Notes |
|----------|------|-------|
| move-to-cool-after-30-days | Lifecycle rule | Enabled, applies to blockBlob type |
| lifecycle-restore-test.txt | Test blob | Uploaded, deleted, restored via soft delete |

## Why These Choices

**30 days, and Cool rather than jumping straight to Archive.** 30 days is a common baseline because access patterns tend to drop off sharply after the first month. Going straight to Archive from a rule like this would be too aggressive for a lot of real workloads, since Archive requires a multi-hour rehydration wait before the data is readable again at all. Cool is the middle ground: cheaper storage than Hot, but still instantly readable. Same Hot vs. Cool vs. Archive tradeoff from Lab 2.1, just automated instead of manual this time.

**Automating the tier change instead of doing it by hand like Lab 2.1.** Manually changing a blob's tier works fine for one file someone is actively paying attention to. It does not scale to a storage account with thousands of objects. A lifecycle rule turns a one-time decision into a standing policy that runs forever without anyone thinking about it again.

**Trusting the existing 7-day soft delete window instead of changing it.** Soft delete is not a real backup strategy, it is a short recovery window for exactly the mistake this lab simulated. 7 days balances catching most accidental deletions during active work against the ongoing cost of retaining every deleted object indefinitely. There is no universally correct number, only a deliberate choice between recovery confidence and cost.

## What I Learned (and Why It Matters)

- **Lifecycle rules operate on a delayed, daily schedule, not in real time.** There is no way to demonstrate a live before-and-after tier change in a single study session; the rule's definition is what gets documented and verified, not its live effect.
- **Soft delete has a visible, ticking countdown, not just an on/off state.** Seeing "Retention: 6 days" rather than a generic "recoverable" label made the tradeoff concrete: this protection has an expiration, and after day 7 the exact same delete becomes permanent.
- **Reusing a blob already referenced in earlier lab documentation would have created cross-lab entanglement.** A dedicated throwaway blob for the delete/restore demo kept this lab self-contained.

## Deployment Instructions

### Prerequisites
- Azure Free Account or pay-as-you-go subscription
- rg-az104-storage and az104storagejf already created (Lab 2.1)
- Azure CLI installed, or PowerShell with the Az module

### 1. Create the lifecycle rule
```
az storage account management-policy create \
  --account-name az104storagejf \
  --resource-group rg-az104-storage \
  --policy @lifecycle-policy.json
```

### 2. Upload, delete, and restore the test blob
```
az storage blob upload \
  --account-name az104storagejf \
  --container-name az104-storage-blob1 \
  --name lifecycle-restore-test.txt \
  --file ./lifecycle-restore-test.txt

az storage blob delete \
  --account-name az104storagejf \
  --container-name az104-storage-blob1 \
  --name lifecycle-restore-test.txt

az storage blob undelete \
  --account-name az104storagejf \
  --container-name az104-storage-blob1 \
  --name lifecycle-restore-test.txt
```

### 3. Verify
```
az storage blob show \
  --account-name az104storagejf \
  --container-name az104-storage-blob1 \
  --name lifecycle-restore-test.txt \
  --query deleted
```
Expected output: false

### 4. Clean up
```
az storage blob delete \
  --account-name az104storagejf \
  --container-name az104-storage-blob1 \
  --name lifecycle-restore-test.txt

az storage account management-policy delete \
  --account-name az104storagejf \
  --resource-group rg-az104-storage
```

## Lessons Learned

This lab was less about learning a new Azure feature and more about confirming a setting that had been quietly active since Lab 2.1 actually does what it claims. It's easy to enable soft delete once and never think about it again. Deliberately deleting something and watching the retention countdown tick down from 7 made the protection feel real rather than theoretical. Next time, I would also test what happens after the retention window fully expires, to see the permanent-deletion side of that same tradeoff, not just the recovery side.

---
*Part of the AZ-104 lab series. See also: [Access Control: SAS Tokens and Private Endpoints](README-az104-storage-access-control.md)*
