# Azure Container Apps

Deployed a public quickstart container image to Azure Container Apps, the last lab in the Compute domain and the point where the domain's story finally closes: VMs, to Availability Sets and Scale Sets, to a fully serverless container platform that scales to zero. Built entirely through the portal wizard, since Container Apps genuinely walks environment and app creation through together in one flow.

## What I Built

- Confirmed the Microsoft.App resource provider was registered on the subscription
- rg-az104-containerapps, a dedicated resource group in East US 2
- cae-az104-demo (Container Apps Environment) and ca-az104-demo (Container App), created together via the portal's combined creation wizard
- Used the built-in quickstart hello-world image (mcr.microsoft.com/azuredocs/containerapps-helloworld:latest) with ingress enabled on port 80
- An auto-created Log Analytics workspace (workspacergaz104containerapps8d2a), required by the environment for logging
- Confirmed the app's Running status and loaded its live public Application Url in a browser, seeing the hello-world page render
- Reviewed the Scale blade (min replicas 0, max replicas 10, an http-scaler rule) and the Revisions and replicas page (one active revision)
- Deleted rg-az104-containerapps in full at the end of the session

## Architecture

```
                    rg-az104-containerapps
                    Region: East US 2
                              |
                +-------------+--------------+
                |                             |
        cae-az104-demo               workspacergaz104...8d2a
        (Container Apps Environment)  (Log Analytics, auto-created)
                |
        ca-az104-demo
        Quickstart hello-world image
        Ingress: enabled, port 80
        Scale: min 0, max 10 replicas
        Scale rule: http-scaler (HTTP scaling)
        Application Url: public, live
```

| Resource | Role | Notes |
|----------|------|-------|
| cae-az104-demo | Container Apps Environment | Hosts the app, tied to the Log Analytics workspace |
| ca-az104-demo | Container App | Quickstart hello-world image, min 0 / max 10 replicas |
| workspacergaz104containerapps8d2a | Log Analytics workspace | Auto-created, required dependency of the environment |

## Why These Choices

**The portal instead of CLI, for this lab specifically.** Container Apps' portal wizard combines environment creation and app deployment into a single guided flow, which made it a genuinely better first-exposure path than the CLI equivalent, which requires several separate commands and more upfront knowledge of the resource model. The CLI and PowerShell equivalents are documented below and are the intended path for revisiting this lab after the full AZ-104 series is complete.

**Microsoft's quickstart image instead of a custom container.** Using mcr.microsoft.com/azuredocs/containerapps-helloworld:latest kept this lab focused on Container Apps as a compute and scaling platform, rather than turning it into a container registry and image-build lab as well. Azure Container Registry and custom image builds are a natural extension for a later, dedicated lab.

**Min replicas of 0, left at its default rather than raised.** This is the single clearest technical distinction from Lab 3.3's VM Scale Set, which always keeps at least its minimum instance count running and billing. Leaving min replicas at 0 here demonstrates the actual value proposition of serverless containers directly, rather than describing it abstractly: this app can scale down to genuinely zero running compute when idle, something no VM-based option in this domain can do.

## What I Learned (and Why It Matters)

- **Container Apps environments require a Log Analytics workspace, and it's created automatically.** This isn't optional infrastructure the way an NSG or public IP can sometimes be skipped, it's a hard dependency for logging. Worth remembering it exists for teardown purposes even though deleting the resource group handles it automatically.
- **Scaling to zero is a genuinely different cost model, not just a smaller number.** A VM Scale Set's minimum replica count is still whole VMs, each billing continuously. A Container App with min replicas set to 0 can have zero running compute and zero compute cost during idle periods, only scaling up (and only billing) when traffic actually arrives. That's the real distinction between "elastic" and "serverless."
- **The portal's combined environment-and-app wizard is a genuinely good learning path for a first exposure to a new Azure service.** Seeing environment creation and app deployment happen together in one guided flow, rather than as separate CLI commands run in sequence, made the relationship between the two resource types clearer than reading documentation about them would have.
- **Revisions exist from the very first deployment, not just after an update.** Even a single, never-updated Container App already has a named revision (ca-az104-demo--lpewvfn in this case). Understanding that a revision is Container Apps' fundamental unit of deployment, not an optional versioning feature you opt into later, matters for how blue-green deployments and traffic splitting work in future labs.

## Deployment Instructions

### Prerequisites
- Azure Free Account or pay-as-you-go subscription
- Microsoft.App resource provider registered (Portal: Subscriptions > Resource providers > Microsoft.App > Register)
- Azure CLI installed, or PowerShell with the Az.App module, if repeating this via CLI/PowerShell rather than the portal

### 1. Register the resource provider (if not already)
```
az provider register --namespace Microsoft.App
az provider show --namespace Microsoft.App --query "registrationState" -o tsv
```
Expected output: Registered

### 2. Create the resource group
```
az group create --resource-group rg-az104-containerapps --location eastus2
```

### 3. Create the Container Apps environment
```
az containerapp env create \
  --name cae-az104-demo \
  --resource-group rg-az104-containerapps \
  --location eastus2
```

### 4. Deploy the Container App
```
az containerapp create \
  --name ca-az104-demo \
  --resource-group rg-az104-containerapps \
  --environment cae-az104-demo \
  --image mcr.microsoft.com/azuredocs/containerapps-helloworld:latest \
  --target-port 80 \
  --ingress external \
  --min-replicas 0 \
  --max-replicas 10
```

### 5. Confirm it's live
```
az containerapp show --name ca-az104-demo --resource-group rg-az104-containerapps --query "properties.configuration.ingress.fqdn" -o tsv
```
Expected output: a URL ending in .azurecontainerapps.io. Open it in a browser to confirm the hello-world page renders.

### 6. Clean up
```
az group delete --name rg-az104-containerapps --yes --no-wait
```
This removes the Container App, the environment, and the auto-created Log Analytics workspace together.

## Lessons Learned

This was the most conceptually different lab in the Compute domain, and it landed that way on purpose: after three labs building on VMs (a single VM, an availability set of VMs, a scale set of VMs), Container Apps was the first resource that isn't a VM at all under the hood. Seeing min replicas actually reach 0, and understanding that this means genuinely zero running compute rather than just one small VM, was the concrete moment the abstraction became real rather than theoretical. The portal wizard also turned out to be a legitimately good teaching tool here, not just a shortcut, since it made the environment-to-app relationship visible in a way reading about it beforehand hadn't. Next time, deploying a second revision and watching traffic split between them would be the natural extension.

---
*Part of the AZ-104 lab series. See also: [Availability Sets and VM Scale Sets](README-az104-availability-scale-sets.md), [ARM Templates vs. Bicep](README-az104-arm-bicep.md)*
