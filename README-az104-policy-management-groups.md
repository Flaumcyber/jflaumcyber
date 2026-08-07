# Azure Policy and Management Groups

Assigning a built-in Azure Policy that requires an environment tag on every resource group in the subscription, then proving it actually enforces by deliberately failing a creation without the tag and succeeding once the tag is present. Governance at the subscription scope, not the resource group scope, since resource groups themselves can only be governed from above.

## What I Built

- A policy assignment of the built-in "Require a tag on resource groups" definition, scoped to the subscription
- Required tag name set to environment, with the default Deny effect
- A failed test: attempted to create TestGroup with no tags, confirmed by a RequestDisallowedByPolicy error
- A successful test: created rg-policy-test with the environment tag present, confirmed it succeeded
- Cleanup: deleted rg-policy-test and removed the policy assignment afterward

## Architecture

```
                    Azure subscription 1
                              |
                    Policy Assignment
                    "Require a tag on resource groups"
                    parameter: tagName = environment
                    effect: Deny
                              |
                +-------------+--------------+
                |                             |
          New RG, no tag              New RG, tag present
          -> TestGroup                 -> rg-policy-test
          RequestDisallowedByPolicy    created successfully
          (blocked)                    environment = test
```

| Resource | Role | Notes |
|----------|------|-------|
| Policy assignment | Governance | Scoped to the subscription, not a resource group |
| TestGroup | Failed test case | No tags, blocked by policy |
| rg-policy-test | Successful test case | environment tag present, allowed, later deleted |

## What I Learned (and Why It Matters)

- **Policy scope has to sit above the resource type it governs.** A policy targeting resource-group creation has to be assigned at the subscription or management group level, since resource groups aren't nested inside other resource groups the way most resources are.
- **Assignment is not the same as verified enforcement.** The policy showed as assigned within seconds, but I didn't actually trust it worked until I deliberately tried to violate it and watched Azure reject the request with a specific error code.
- **Typos in policy parameters are a real, silent risk.** My first attempt at the tag name was "enviorment." Azure does not validate tag names against a dictionary, so a misspelled policy would have been enforced exactly as configured, with no warning that it didn't match any naming convention anyone would actually use.

## Deployment Instructions

### Prerequisites
- Azure Free Account or pay-as-you-go subscription
- Azure CLI installed, or PowerShell with the Az module

### 1. Assign the policy
```
POLICY_ID=$(az policy definition list --query "[?displayName=='Require a tag on resource groups'].name" -o tsv)

az policy assignment create \
  --name require-tag-on-resource-groups \
  --display-name "Require a tag on resource groups" \
  --policy "$POLICY_ID" \
  --scope "/subscriptions/<subscription-id>" \
  --params '{ "tagName": { "value": "environment" } }'
```

### 2. Confirm it blocks a resource group without the tag
```
az group create --name TestGroup --location eastus
```
Expected output: an error containing RequestDisallowedByPolicy.

### 3. Confirm it allows a resource group with the tag
```
az group create --name rg-policy-test --location eastus --tags environment=test
```
Expected output: success, no policy error.

### 4. Clean up
```
az group delete --name rg-policy-test --yes --no-wait
az policy assignment delete --name require-tag-on-resource-groups --scope "/subscriptions/<subscription-id>"
```

## Lessons Learned

The typo in the tag parameter was the most useful mistake in this lab. It made concrete something that's easy to nod along to in the abstract: Azure Policy enforces exactly what you configured, not what you meant to configure. Next time, I would double check every parameter value on the review screen before clicking Create, rather than after, since catching it early would have saved a redo.

---
*Part of the AZ-104 lab series. See also: [RBAC built-in vs custom roles](README-az104-rbac-custom-roles.md)*
