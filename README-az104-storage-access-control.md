# Access Control: SAS Tokens and Private Endpoints

Two different ways to control access to storage that solve two different problems. A SAS token grants scoped, time-limited authorization without ever sharing the storage account key. A private endpoint removes the storage account from the public internet entirely for a given network path. Built and tested both against the same storage account from Lab 2.1.

## What I Built

- A SAS token on baby-yoda-2021.jpeg scoped to Read only, with a short expiry
- Confirmation that the SAS URL loads the blob, and the same URL without the token returns PublicAccessNotPermitted
- A minimal VNet, vnet-az104-storage, with subnet snet-private-endpoints (10.0.0.0/24), configured with no default outbound access
- A private endpoint targeting az104storagejf's blob sub-resource, linked to the privatelink.blob.core.windows.net private DNS zone
- Verification that the private endpoint's network interface received a real private IP, 10.0.0.4

## Architecture

```
                    rg-az104-storage
                              |
                +-------------+--------------+
                |                             |
          SAS Token (auth layer)      Private Endpoint (network layer)
          scope: baby-yoda-2021.jpeg   name: vnet-az104-storage (see note)
          permission: Read only        target: az104storagejf, blob
          expiry: ~8 hours              |
          -> works with token         vnet-az104-storage
          -> PublicAccessNotPermitted   snet-private-endpoints (10.0.0.0/24)
             without it                private IP: 10.0.0.4
                                        DNS: privatelink.blob.core.windows.net
```

| Resource | Role | Notes |
|----------|------|-------|
| SAS token | Authorization | Read only, short expiry, on baby-yoda-2021.jpeg |
| vnet-az104-storage | Virtual network | Address space 10.0.0.0/16 |
| snet-private-endpoints | Subnet | 10.0.0.0/24, no default outbound access |
| Private endpoint | Network isolation | Named vnet-az104-storage due to a wizard naming carry-over; targets az104storagejf blob sub-resource |

## Why These Choices

**SAS token, scoped to Read only, instead of the storage account key.** The account key grants full read, write, and delete access to everything in the account, indefinitely. A SAS token narrows that to exactly one permission, on exactly one resource, for exactly one short window of time. If a SAS token leaks, the exposure is bounded. If the account key leaks, the exposure is the entire storage account, forever, until someone notices and rotates it.

**Private endpoint instead of a firewall rule on the public endpoint.** A firewall rule still leaves the storage account reachable from the public internet, just restricted to approved source IPs. A private endpoint removes the public attack surface for that connection path entirely, since traffic from the VNet travels over Microsoft's private backbone network and never touches the public internet.

**Private subnet with no default outbound access.** A private endpoint doesn't need to initiate outbound internet connections to function; its job is letting resources inside the VNet reach the storage account privately. Leaving the subnet without default outbound access is tighter posture for a subnet whose only purpose is hosting private endpoints.

## What I Learned (and Why It Matters)

- **Authorization and network isolation are separate problems, often solved together.** A SAS token controls who can do what. A private endpoint controls where traffic is even allowed to originate from. A real production storage account would typically use both at once.
- **Private endpoints only show compatible VNets in the same region as the endpoint itself.** An empty Virtual network dropdown during creation wasn't a bug, it was Azure correctly filtering out a region mismatch on the Basics tab.
- **Not every mistake is worth fixing immediately.** The private endpoint ended up named vnet-az104-storage instead of the planned pe-az104storagejf-blob, but it functions identically either way. Deleting and recreating a resource that bills hourly, purely to fix a label, would have cost more than the naming inconsistency itself.

## Deployment Instructions

### Prerequisites
- Azure Free Account or pay-as-you-go subscription
- rg-az104-storage and az104storagejf already created (Lab 2.1)
- Azure CLI installed, or PowerShell with the Az module

### 1. Generate a SAS token
```
EXPIRY=$(date -u -d "8 hours" '+%Y-%m-%dT%H:%MZ')

az storage blob generate-sas \
  --account-name az104storagejf \
  --container-name az104-storage-blob1 \
  --name baby-yoda-2021.jpeg \
  --permissions r \
  --expiry $EXPIRY \
  --https-only
```

### 2. Build the VNet and private DNS zone
```
az network vnet create \
  --resource-group rg-az104-storage \
  --name vnet-az104-storage \
  --address-prefix 10.0.0.0/16 \
  --subnet-name snet-private-endpoints \
  --subnet-prefix 10.0.0.0/24

az network private-dns zone create \
  --resource-group rg-az104-storage \
  --name privatelink.blob.core.windows.net

az network private-dns link vnet create \
  --resource-group rg-az104-storage \
  --zone-name privatelink.blob.core.windows.net \
  --name vnet-az104-storage-link \
  --virtual-network vnet-az104-storage \
  --registration-enabled false
```

### 3. Create the private endpoint
```
az network private-endpoint create \
  --resource-group rg-az104-storage \
  --name pe-az104storagejf-blob \
  --vnet-name vnet-az104-storage \
  --subnet snet-private-endpoints \
  --private-connection-resource-id $(az storage account show --name az104storagejf --query id -o tsv) \
  --group-id blob \
  --connection-name psc-az104storagejf-blob
```

### 4. Verify
```
az network private-endpoint show \
  --resource-group rg-az104-storage \
  --name pe-az104storagejf-blob \
  --query "privateLinkServiceConnections[0].privateLinkServiceConnectionState.status"
```
Expected output: "Approved"

### 5. Clean up (this resource bills hourly)
```
az network private-endpoint delete --resource-group rg-az104-storage --name pe-az104storagejf-blob
az network private-dns link vnet delete --resource-group rg-az104-storage --zone-name privatelink.blob.core.windows.net --name vnet-az104-storage-link --yes
az network private-dns zone delete --resource-group rg-az104-storage --name privatelink.blob.core.windows.net --yes
az network vnet delete --resource-group rg-az104-storage --name vnet-az104-storage
```

## Lessons Learned

The empty VNet dropdown was the most instructive moment in this lab. It looked like a broken portal at first, but it was actually correct behavior enforcing a real constraint: private endpoints have to live in the same region as their VNet. Next time, I would check region alignment as a first troubleshooting step whenever a resource-selection dropdown comes up unexpectedly empty, before assuming something is actually broken.

---
*Part of the AZ-104 lab series. See also: [Storage Account Fundamentals](README-az104-storage-fundamentals.md)*
