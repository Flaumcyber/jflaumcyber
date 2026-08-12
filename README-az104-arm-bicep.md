# ARM Templates vs. Bicep

Deployed the same storage account resource two ways in the same resource group, once with a raw ARM JSON template and once with the equivalent Bicep file, to compare the authoring experience directly and confirm that Bicep transpiles to the same underlying ARM deployment engine rather than being a separate one.

## What I Built

- rg-az104-iac-demo, a dedicated resource group for this authoring-tooling lab
- azuredeploy-storage.json (50 lines), an ARM JSON template deploying a StorageV2 account with TLS 1.2 minimum and public blob access disabled
- Deployed the ARM template via Azure CLI, creating storage account az104iacdemojf
- main.bicep (24 lines), the equivalent Bicep file for the identical resource, targeting a second storage account name (az104iacdemojf2) so both deployments could coexist
- Deployed the Bicep file via Azure CLI
- Ran az bicep decompile against the original ARM JSON and compared the auto-generated Bicep file against the hand-written one

## Architecture

```
                    rg-az104-iac-demo
                    Region: East US 2
                              |
                +-------------+--------------+
                |                             |
        az104iacdemojf                az104iacdemojf2
        deployed via ARM JSON         deployed via Bicep
        StorageV2, Standard_LRS       StorageV2, Standard_LRS
        TLS 1.2 min, no public blob   TLS 1.2 min, no public blob

                Same ARM deployment engine underneath both
```

| Resource | Role | Notes |
|----------|------|-------|
| az104iacdemojf | Storage account | Deployed from azuredeploy-storage.json, StorageV2, Standard_LRS |
| az104iacdemojf2 | Storage account | Deployed from main.bicep, identical properties to the ARM-deployed account |

## Why These Choices

**A storage account instead of a VM.** The previous Compute lab (3.1) already fought through a real regional VM quota issue, and this lab isn't about the deployed resource, it's about the authoring experience of ARM JSON versus Bicep. A storage account is free-tier, avoids re-triggering that East US 2 VM sizing story, and keeps this lab's cost at zero while still deploying a real, non-trivial resource with several configurable properties worth comparing across both syntaxes.

**Writing the ARM JSON by hand first, not starting in Bicep.** The whole point of this lab is to feel the pain before being told it exists. Starting in Bicep would have meant taking "Bicep is easier" on faith rather than experiencing the verbose parameter blocks, the resourceId() function-call syntax, and the schema boilerplate that Bicep exists specifically to eliminate. Writing ARM JSON first, then Bicep, made the size and readability difference concrete rather than something to just accept as received wisdom.

**Distinct storage account names instead of reusing one.** Giving the ARM-deployed and Bicep-deployed storage accounts different names (az104iacdemojf and az104iacdemojf2) let both deployments coexist in the same resource group at the same time, which made the resource-group-overview screenshot a direct side-by-side comparison instead of a before-and-after shot that required deleting one deployment before creating the other.

## What I Learned (and Why It Matters)

- **Bicep is an authoring layer over ARM, not a second deployment engine.** The Azure CLI transpiles the Bicep file to ARM JSON in memory before sending it to the same ARM deployment engine that processed the hand-written JSON. Both templates produced identical deployed infrastructure through the same API.
- **Roughly half the lines isn't a marketing claim, it's what actually happened.** The hand-written ARM JSON came to 50 lines; the equivalent Bicep file came to 24. The difference was almost entirely syntactic overhead: schema declarations, type wrappers, and function-call syntax that Bicep doesn't require.
- **A clean ARM source decompiles into clean Bicep.** Decompiling the original JSON produced a file nearly identical to the hand-written Bicep, differing only in the one property that was deliberately changed. That's a stronger case for the tooling's reliability than "decompiled code is always messy" would have been, and it shows the real value of Bicep is in avoiding the verbose JSON at authoring time, not in cleaning up bad output after the fact.
- **PowerShell line-continuation and path syntax differ from bash, and pwsh on Linux isn't Windows PowerShell.** Backtick continuation instead of backslash, forward-slash paths instead of C:\, and mkdir -p to create parent directories were all small but real friction points working in this environment.

## Deployment Instructions

### Prerequisites
- Azure Free Account or pay-as-you-go subscription
- Azure CLI installed, or PowerShell with the Az module
- Note: this lab is written for pwsh on Linux; adjust path syntax if using native Windows PowerShell

### 1. Create the resource group
```
az group create --resource-group rg-az104-iac-demo --location eastus2
```

### 2. Deploy the ARM JSON template
```
az deployment group create \
  --resource-group rg-az104-iac-demo \
  --name deploy-arm-storage \
  --template-file azuredeploy-storage.json
```

### 3. Deploy the equivalent Bicep file
```
az deployment group create \
  --resource-group rg-az104-iac-demo \
  --name deploy-bicep-storage \
  --template-file main.bicep
```

### 4. Verify both deployments
```
az resource list --resource-group rg-az104-iac-demo --output table
```
Expected output: both az104iacdemojf and az104iacdemojf2 listed, type Microsoft.Storage/storageAccounts, status Succeeded

### 5. Optional: decompile the ARM JSON and compare
```
az bicep decompile --file azuredeploy-storage.json
cat azuredeploy-storage.bicep
```

### 6. Clean up
```
az group delete --name rg-az104-iac-demo --yes --no-wait
```

## Lessons Learned

This lab was lower-stakes than the Compute labs that came before it since the deployed resource was free-tier, but the authoring comparison itself was the actual point and it delivered. Seeing 50 lines of ARM JSON compress into 24 lines of equivalent Bicep, and then watching a decompile of the original JSON land almost exactly back on the hand-written Bicep file, made the case for Bicep concretely rather than as something to accept on faith. The environment friction (PowerShell on Linux behaving differently than expected at several points) was also a useful, if minor, real-world reminder that tooling assumptions are worth verifying early rather than mid-deployment.

---
*Part of the AZ-104 lab series. See also: [VM Deployment: Linux and Windows](README-az104-vm-deployment.md)*
