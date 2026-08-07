# Entra ID Users, Groups, and Group-Based RBAC

Creating Microsoft Entra ID users, organizing them into a security group, and assigning a role to the group rather than to individuals. This is the identity foundation the rest of the AZ-104 labs build on.

## What I Built

- Two Entra ID users: jdoe and asmith
- A security group, sg-az104-readers, with both users as members
- A Reader role assignment scoped to the rg-az104-identity resource group, assigned to the group instead of the individual users
- Verification of inherited access using Check access on one of the member users

## Architecture

```
              jflaum.onmicrosoft.com (tenant)
                              |
                    +---------+---------+
                    |                   |
              Entra ID Users     Security Group
              - jdoe               sg-az104-readers
              - asmith                    |
                    |               +-----+-----+
                    +---------------+           |
                    (group members)      Role Assignment
                                          Reader at
                                          rg-az104-identity scope
```

| Resource | Role | Notes |
|----------|------|-------|
| jdoe, asmith | Entra ID users | Test accounts, no licenses assigned |
| sg-az104-readers | Security group | Mail-disabled, security-enabled |
| rg-az104-identity | Resource group | Scope for the Reader assignment |

## What I Learned (and Why It Matters)

- **Group-based RBAC scales better than per-user assignment.** Adding or removing someone from the group changes their access automatically, and it keeps the role assignment list on the resource itself short and auditable. This is the pattern Microsoft expects on the AZ-104 exam and in real environments.
- **Object IDs, not display names, are what role assignments actually reference.** The portal shows friendly names, but every assignment under the hood is tied to an immutable object ID. The CLI and PowerShell commands resolve the group and users to their IDs before assigning anything, which is worth understanding before troubleshooting a broken assignment.
- **Entra ID resources sit outside standard ARM and Bicep resource types.** Bicep can assign a role to a group, but it cannot create the group itself without the Microsoft Graph Bicep extension, so the group has to exist first through the CLI, PowerShell, or the portal, then get referenced by object ID in the Bicep deployment.

## Deployment Instructions

### Prerequisites
- Azure Free Account or pay-as-you-go subscription
- Azure CLI installed, or PowerShell with the Az and Microsoft.Graph modules
- Permissions in Entra ID to create users and groups (Global Administrator or User Administrator role)

### 1. Create the users and group
Using the Azure CLI:
1. Run `az login`
2. Create both users with `az ad user create`
3. Create the security group with `az ad group create`
4. Add both users to the group with `az ad group member add`

### 2. Create the resource group and assign the role
1. Create rg-az104-identity in your target region
2. Resolve the group's object ID with `az ad group show`
3. Assign the Reader role to the group at the resource group scope with `az role assignment create`

### 3. Verify
```
az role assignment list --resource-group rg-az104-identity --output table
```
Expected output: one row showing Reader assigned to sg-az104-readers, not to jdoe or asmith individually.

### 4. Clean up
```
az group delete --name rg-az104-identity --yes --no-wait
az ad group delete --group "sg-az104-readers"
az ad user delete --id jdoe@jflaum.onmicrosoft.com
az ad user delete --id asmith@jflaum.onmicrosoft.com
```

## Lessons Learned

The main thing this lab confirmed is that identity and governance labs are less about clicking through the portal and more about understanding how object IDs and scopes connect the pieces together. Next time, I would script the object ID lookups into a single variable file so the CLI and Terraform versions of the lab could share the same values instead of hardcoding user principal names in multiple places.

---
*Part of the AZ-104 lab series. See also: [RBAC custom roles](README-az104-rbac-custom-roles.md)*
