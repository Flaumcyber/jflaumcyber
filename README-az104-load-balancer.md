# Load Balancer

Deployed a Standard Load Balancer distributing HTTP traffic across two Windows VMs running IIS. The health probe reported both backend instances healthy from the moment the load balancer went live, but the public IP itself returned ERR_EMPTY_RESPONSE in the browser. Traced this to the NSG layer: the default AllowAzureLoadBalancerInBound rule covers the probe's traffic, but real internet-sourced client traffic on port 80 had no explicit Allow rule and was being silently dropped by the default DenyAllInBound rule.

## What I Built

- rg-az104-loadbalancer with vm-az104-lb1 and vm-az104-lb2, both Windows, both with IIS installed via Run Command
- Confirmed IIS worked locally on each VM independently before introducing the load balancer
- lb-az104-demo, Standard SKU, with backend pool backend-lb-demo (both VMs), health probe probe-http-80 (HTTP, port 80), and load balancing rule rule-http-80 (TCP, port 80)
- Confirmed via the portal's health status view that both backend instances showed Up, 100% of instances healthy
- Hit ERR_EMPTY_RESPONSE loading the load balancer's public IP directly, despite the healthy probe status, and ruled out Windows Firewall on both VMs
- Verified via Run Command on vm-az104-lb1 that W3SVC was Running, port 80 had an active listener, the site's HTTP binding was correctly `*:80`, and localhost returned a clean HTTP 200
- Used Azure CLI in Cloud Shell to list the NSGs on each VM's NIC (vm-az104-lb1-nsg, vm-az104-lb2-nsg) and found zero custom rules on either, only Azure's defaults: AllowVnetInBound, AllowAzureLoadBalancerInBound, DenyAllInBound
- Added an explicit allow-http-80 rule (priority 100, Allow, TCP, port 80, source Internet) to both NSGs via Azure CLI
- Reloaded the load balancer's public IP and confirmed a successful response, correctly identifying vm-az104-lb1 as the serving backend

## Architecture

```
                    rg-az104-loadbalancer
                    Region: East US 2
                              |
                    lb-az104-demo (Standard SKU)
                    Frontend IP: 172.175.19.131
                              |
                    rule-http-80 (TCP/80)
                    probe-http-80 (HTTP/80)
                              |
                +-------------+--------------+
                |                             |
      vm-az104-lb1                    vm-az104-lb2
      IIS, Windows                    IIS, Windows
      vm-az104-lb1-nsg                vm-az104-lb2-nsg
                |                             |
      backend-lb-demo (backend pool, both VMs)

      Health probe (from 168.63.129.16): both VMs Up, 100% healthy
      Real client request (public IP -> port 80): ERR_EMPTY_RESPONSE

      Root cause: default NSG rules only cover AllowAzureLoadBalancerInBound
      (the probe source) and DenyAllInBound. No explicit Allow rule existed
      for real internet traffic on port 80, so the probe passed while every
      actual browser request was silently dropped.

      Fix: added allow-http-80 (priority 100, Allow, TCP/80, source Internet)
      to both vm-az104-lb1-nsg and vm-az104-lb2-nsg
```

| Resource | Role | Notes |
|----------|------|-------|
| lb-az104-demo | Standard Load Balancer | Frontend IP 172.175.19.131, backend pool of 2 VMs |
| vm-az104-lb1 / vm-az104-lb2 | IIS backend instances | Each with its own per-VM NSG |
| probe-http-80 | Health probe | HTTP, port 80, validates the application layer, not just the socket |
| rule-http-80 | Load balancing rule | TCP, frontend port 80 to backend port 80 |
| allow-http-80 | NSG inbound rule (added mid-lab) | Priority 100, Allow, TCP/80, source Internet; the actual fix |

## Why These Choices

**Standard SKU, not Basic.** Basic Load Balancer is on Microsoft's own retirement roadmap and lacks availability-zone support and a few security defaults. Standard also does not implicitly allow broad inbound traffic the way Basic's looser posture does, which is exactly what surfaced the NSG gap in this lab rather than masking it. Practicing against Standard's stricter defaults is more representative of both the exam and real production environments.

**Two VMs in the backend pool, not one.** A load balancer distributing across a single instance proves nothing about actual distribution, only that the rule and probe exist. Two VMs made it possible to confirm the fix needed to happen at the NSG layer on a per-VM basis, since both VMs independently showed the identical misconfiguration.

**An HTTP health probe on port 80, not TCP.** A TCP probe only confirms a port accepts connections; an HTTP probe confirms the web application behind it is actually returning a successful response. Since the entire point of this lab was distributing real web traffic, the probe needed to validate the same layer the real traffic depends on.

**Fixing this at the NSG layer with an explicit Allow rule, not by loosening Windows Firewall further.** Windows Firewall was already confirmed clear on both VMs. Azure's NSG sits in front of the OS firewall entirely and enforces independently of it; the explicit allow-http-80 rule fixed the actual point of failure and is the architecturally correct place to manage this kind of traffic decision.

## What I Learned (and Why It Matters)

- **A healthy load balancer probe is not proof that real traffic can reach the backend.** Azure's default AllowAzureLoadBalancerInBound NSG rule exists specifically for the probe's own source address (168.63.129.16), completely independent of whether any rule exists for real internet-sourced client traffic. A 100% healthy probe and a totally unreachable public endpoint are two different traffic paths governed by two different rules, not a contradiction.
- **Standard Load Balancer's stricter default posture is a deliberate security property, not an inconvenience.** Basic SKU's looser defaults would have masked this exact lesson entirely. Standard forced an explicit, auditable decision about what traffic is allowed in, which is exactly the behavior worth practicing before encountering it in a real environment.
- **Ruling things out in the right order saves real troubleshooting time.** Windows Firewall was checked and cleared first as the fastest, most familiar check. The application layer was then confirmed fully healthy directly on the VM (service running, port listening, correct binding, working localhost response) before moving to the network layer. Confirming the VM itself was innocent first made the eventual NSG discovery unambiguous.
- **Windows Firewall and NSGs are two independent layers; clearing one says nothing about the other.** It's easy to treat "firewall checked" as covering network security generally. Azure's NSG enforces its own rule set entirely independent of the guest OS firewall, and any future unreachable-service troubleshooting needs to check both layers explicitly.

## Deployment Instructions

### Prerequisites
- Azure Free Account or pay-as-you-go subscription
- Azure CLI installed, or PowerShell with the Az module, if repeating this via CLI/PowerShell rather than the portal
- Two Windows VMs with IIS already installed and confirmed working locally before introducing the load balancer

### 1. Create the resource group and public IP
```
az group create --resource-group rg-az104-loadbalancer --location eastus2

az network public-ip create \
  --resource-group rg-az104-loadbalancer \
  --name pip-lb-az104-demo \
  --sku Standard
```

### 2. Create the load balancer, backend pool, probe, and rule
```
az network lb create \
  --resource-group rg-az104-loadbalancer \
  --name lb-az104-demo \
  --sku Standard \
  --public-ip-address pip-lb-az104-demo \
  --frontend-ip-name frontend-lb-demo \
  --backend-pool-name backend-lb-demo

az network lb probe create \
  --resource-group rg-az104-loadbalancer \
  --lb-name lb-az104-demo \
  --name probe-http-80 \
  --protocol http \
  --port 80 \
  --path /

az network lb rule create \
  --resource-group rg-az104-loadbalancer \
  --lb-name lb-az104-demo \
  --name rule-http-80 \
  --protocol tcp \
  --frontend-port 80 \
  --backend-port 80 \
  --frontend-ip-name frontend-lb-demo \
  --backend-pool-name backend-lb-demo \
  --probe-name probe-http-80
```

### 3. Add both VMs' NICs to the backend pool
Portal: Load balancer > Backend pools > backend-lb-demo > Add both VMs' NICs. Confirm the health status view shows both as Up before proceeding.

### 4. If the public IP is unreachable despite a healthy probe, check the NSG layer
```
az network nsg rule list \
  --resource-group rg-az104-loadbalancer \
  --nsg-name <vm-nsg-name> \
  -o table
```
If this returns no custom rules, only Azure's defaults, add an explicit Allow rule for the real traffic port:
```
az network nsg rule create \
  --resource-group rg-az104-loadbalancer \
  --nsg-name <vm-nsg-name> \
  --name allow-http-80 \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes Internet \
  --source-port-ranges '*' \
  --destination-address-prefixes '*' \
  --destination-port-ranges 80
```
Repeat for every VM in the backend pool; each VM's NSG needs this rule independently.

### 5. Verify
Load the load balancer's public IP in a browser. Refresh a few times to confirm requests are being distributed across both backend VMs.

## Lessons Learned

The most valuable part of this lab wasn't the load balancer configuration itself, which was straightforward, it was the gap between "the probe says healthy" and "a real user can reach the site." That gap turned out to be a single missing NSG rule, invisible from the load balancer's own health status view because the probe and real client traffic take the same path but are governed by different default rules. Ruling out the application layer methodically before moving to the network layer, rather than guessing at both simultaneously, is what made the eventual NSG discovery fast and unambiguous once the investigation got there. Next time a load balancer setup shows a healthy probe but an unreachable endpoint, checking the NSG's actual rule list, not just Windows Firewall, will be the first move rather than a later one.

---
*Part of the AZ-104 lab series. See also: [Azure Bastion and Network Watcher](README-az104-bastion-network-watcher.md)*
