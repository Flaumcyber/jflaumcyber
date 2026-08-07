# RBAC: Built-In vs. Custom Roles

Assigning a built-in Reader role directly to a user, then building and assigning a custom role scoped down to exactly three permissions: read, start, and restart on virtual machines, with no delete or write access anywhere. This builds on Lab 1.1's group-based Reader assignment by adding the direct-assignment and custom-role patterns to the same resource group.

## What I Built

- Built-in Reader role assigned directly to asmith at the rg-az104-identity scope, contrasted against the group-based assignment from Lab 1.1
- A custom role, VM Restart Operator, with exactly three actions: read, start, and restart on virtual machines
- The custom role assigned directly to jdoe at the same resource group scope
- Verification that jdoe now holds two separate role assignments at once: Reader (inherited through the group) and VM Restart Operator (direct)

## Architecture

```
                    rg-az104-identity
                              |
                +-------------+--------------+
                |                             |
          Reader (built-in)          VM Restart Operator (custom)
          direct assignment           direct assignment
          -> asmith                    -> jdoe
                                        actions:
                                        - Microsoft.Compute/virtualMachines/read
                                        - Microsoft.Compute/virtualMachines/start/action
                                        - Microsoft.Compute/virtualMachines/restart/action
                                        (no delete, no write)

                                        note: jdoe also still has Reader
                                        via sg-az104-readers from Lab 1.1
```

| Resource | Role | Notes |
|----------|------|-------|
| asmith | Reader (built-in, direct) | Contrast case against Lab 1.1's group-based Reader |
| jdoe | VM Restart Operator (custom, direct) | Only read, start, restart on VMs, no delete |
| jdoe | Reader (inherited) | Still active from Lab 1.1's group membership |

## What I Learned (and Why It Matters)

- **Custom roles use an allow-list model, not a deny-list one.** Leaving delete out of the actions array was enough to block it. notActions exists for carving exceptions out of a wildcard grant, not something that had to be populated for a role this narrow.
- **assignableScopes is what actually limits where a custom role can be used.** Scoping it to just rg-az104-identity means this role cannot be assigned anywhere else in the subscription, even by someone with Owner access, without editing the role definition itself.
- **A single user can hold a group-inherited role and a direct assignment at the same time.** jdoe's Check access result showed Reader (via sg-az104-readers) and VM Restart Operator (direct) side by side. Being able to read and reason about layered access like this is exactly what the exam expects.

## Deployment Instructions

### Prerequisites
- Azure Free Account or pay-as-you-go subscription
- Azure CLI installed, or PowerShell with the Az module
- rg-az104-identity resource group and the jdoe/asmith users already created (Lab 1.1)

### 1. Assign built-in Reader directly to asmith
```
az role assignment create --assignee asmith@jflaum.onmicrosoft.com --role "Reader" --resource-group rg-az104-identity
```

### 2. Create the custom role
```
az role definition create --role-definition '{
  "Name": "VM Restart Operator",
  "Description": "Can read, start, and restart virtual machines. Cannot delete.",
  "Actions": [
    "Microsoft.Compute/virtualMachines/read",
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/restart/action"
  ],
  "NotActions": [],
  "AssignableScopes": ["/subscriptions/<subscription-id>/resourceGroups/rg-az104-identity"]
}'
```

### 3. Assign the custom role to jdoe
```
az role assignment create --assignee jdoe@jflaum.onmicrosoft.com --role "VM Restart Operator" --resource-group rg-az104-identity
```

### 4. Verify
```
az role assignment list --resource-group rg-az104-identity --assignee jdoe@jflaum.onmicrosoft.com --output table
```
Expected output: two rows, one for Reader and one for VM Restart Operator.

### 5. Clean up
```
az role assignment delete --assignee jdoe@jflaum.onmicrosoft.com --role "VM Restart Operator" --resource-group rg-az104-identity
az role definition delete --name "VM Restart Operator"
az role assignment delete --assignee asmith@jflaum.onmicrosoft.com --role "Reader" --resource-group rg-az104-identity
```

## Lessons Learned

The biggest shift in understanding from this lab was realizing custom roles are additive, not restrictive by default. There is no need to explicitly deny delete access. Simply not granting it is enough, since Azure RBAC denies anything not explicitly listed in actions. Next time, I would test the boundary more directly by actually deploying a small VM and confirming jdoe truly cannot delete it through the portal, rather than relying on the Check access panel alone.

---
*Part of the AZ-104 lab series. See also: [Entra ID users and groups](README-az104-identity-governance.md)*
