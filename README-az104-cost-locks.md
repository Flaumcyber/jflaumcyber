# Subscriptions, Cost Management, and Resource Locks

Two governance controls that protect a subscription from two different kinds of mistakes: a budget alert that catches unexpected spend before it grows, and a resource lock that blocks accidental deletion of a resource group entirely. Verified both by proving the lock actually stops a real delete attempt, not just by assuming it would.

## What I Built

- Reused the budget-az104-labs budget from earlier setup: $180 amount, alerts at 50/75/90% of spend
- Applied a CanNotDelete lock named prevent-delete-identity-rg to rg-az104-identity
- Attempted to delete rg-az104-identity through the normal flow and confirmed Azure blocked it, naming the lock in the error
- Removed the lock afterward and confirmed the resource group had no locks remaining

## Architecture

```
                    Azure subscription 1
                              |
                +-------------+--------------+
                |                             |
          Budget: budget-az104-labs      rg-az104-identity
          amount: $180                          |
          alerts: 50% / 75% / 90%          Lock: prevent-delete-identity-rg
          recipient: jflaum.it@gmail.com   type: CanNotDelete
                                                  |
                                            Delete attempt
                                            -> blocked by lock
                                            -> lock removed afterward
```

| Resource | Role | Notes |
|----------|------|-------|
| budget-az104-labs | Cost control | Billing account scope, reused from earlier setup |
| prevent-delete-identity-rg | Resource lock | CanNotDelete, applied to rg-az104-identity |

## What I Learned (and Why It Matters)

- **CanNotDelete is not the same as read-only.** The lock blocked deletion of the resource group entirely, but I could still have edited tags, added resources, or changed IAM assignments inside it while the lock was active. A ReadOnly lock would have blocked that too.
- **Locks and budgets protect against different failure modes.** A budget catches spend drifting somewhere unexpected. A lock catches a person, script, or automated cleanup accidentally deleting something that's still in use. Neither substitutes for the other.
- **The delete error names the specific lock, not just a generic denial.** That specificity matters in a real environment with many locks across many resource groups, since it tells you exactly what to go remove rather than making you hunt for the cause.

## Deployment Instructions

### Prerequisites
- Azure Free Account or pay-as-you-go subscription
- rg-az104-identity resource group already created (Lab 1.1)
- Azure CLI installed, or PowerShell with the Az module

### 1. Apply the lock
```
az lock create \
  --name prevent-delete-identity-rg \
  --lock-type CanNotDelete \
  --notes "Cannot Delete Resource Group" \
  --resource-group rg-az104-identity
```

### 2. Confirm it blocks deletion
```
az group delete --name rg-az104-identity --yes
```
Expected output: an error stating the resource group is locked and cannot be deleted.

### 3. Remove the lock
```
az lock delete --name prevent-delete-identity-rg --resource-group rg-az104-identity
```

### 4. Verify
```
az lock list --resource-group rg-az104-identity -o table
```
Expected output: an empty table, confirming no locks remain.

## Lessons Learned

This lab was less about learning something new and more about confirming an assumption under actual pressure: that a lock genuinely stops a delete, not just that it looks configured correctly in the portal. That distinction, between something being set up and something being verified, has been the throughline across all four labs in this domain. Next time, I would test the ReadOnly lock level too, to see the difference in behavior directly rather than only reading about it.

---
*Part of the AZ-104 lab series. See also: [Azure Policy and management groups](README-az104-policy-management-groups.md)*
