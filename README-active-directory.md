# Setting up Windows 2025 Domain Controller + Joining a Windows 11 Client + Verifying the Domain is Live

Standing up a Windows Server 2025 domain controller from scratch in VirtualBox, joining a Windows 11 client to it, and verifying the domain is live — the foundation the rest of the home lab (Group Policy, security hardening, Kali/Nessus) builds on top of.

## What I Built

- A VirtualBox VM running **Windows Server 2025 Standard (Desktop Experience)**, installed via an unattended setup
- A static IP (192.168.10.10/24) and hostname (DC01), with DC01 pointing DNS at itself
- **Active Directory Domain Services** promoted on DC01, creating a new forest and domain: jflaum.cyber
- DNS running alongside AD DS on the domain controller
- A **Windows 11 client** (192.168.10.20) joined to jflaum.cyber
- A first domain user account (jdoe) used to confirm the client could authenticate against the domain

## Architecture

```
                    jflaum.cyber (forest / domain root)
                              |
                +-------------+--------------+
                |                             |
        Windows Server 2025            Windows 11 Pro
        DC01 — 192.168.10.10           192.168.10.20
        - Active Directory DS          - Domain-joined
        - DNS Server                   - Authenticates via DC01
        - Domain Controller            - DNS -> 192.168.10.10
                |                             |
                +-------------+--------------+
                              |
                   VirtualBox Internal Network
                        192.168.10.0/24
```

| Host | Role | IP | OS |
|------|------|----|----|
| DC01 | Domain Controller, DNS | 192.168.10.10 | Windows Server 2025 Standard (Eval) |
| Client01 | Domain-joined workstation | 192.168.10.20 | Windows 11 Pro |

## What I Learned (and Why It Matters)

- **Unattended installs save time but hide steps.** Walking through the VirtualBox unattended-install flow (hostname, credentials, product key) up front meant I didn't have to babysit the Windows setup wizard — but it also meant I had to know exactly what I wanted *before* kicking it off, since there's no "next, next, next" to course-correct along the way.
- **A domain controller has to be its own DNS server.** AD relies on DNS to locate domain controllers, services, and other clients (via SRV records). Pointing DC01's own DNS settings at itself — and only after that, setting a static IP — is what actually let promotion succeed without errors. Get the order wrong and the AD DS Configuration Wizard throws prerequisite failures.
- **Promotion is more than a checkbox.** "Promote to Domain Controller" actually involves real decisions: forest functional level, whether to also install DNS, and a separate DSRM (Directory Services Restore Mode) password that's distinct from the domain admin password and exists specifically for offline AD recovery. Skipping past these without understanding them is how labs (and production environments) end up misconfigured.
- **Client-side DNS is the #1 domain-join failure point.** The Windows 11 client wouldn't find jflaum.cyber until its DNS was explicitly pointed at DC01 — public/ISP DNS can't resolve a private AD domain. This is the most common real-world reason a domain join fails, so reproducing (and fixing) it here was a useful failure to actually see.
- **Verification beats assumption.** Using systeminfo | findstr /B /C:"Domain" on the server and a successful jdoe login on the client gave concrete proof the domain was actually live, rather than just trusting that the wizard said "success."

## Deployment Instructions

### Prerequisites
- VirtualBox 7+
- Windows Server 2025 evaluation ISO ([microsoft.com](https://www.microsoft.com/evalcenter))
- Windows 11 ISO
- ~10 GB free RAM and 160 GB free disk across both VMs

### 1. Create the Domain Controller VM
1. VirtualBox → **New** → name it, point the ISO image field at the Server 2025 ISO
2. Allocate **2048 MB RAM**, **1 CPU**
3. Create an **80 GB** virtual disk (VDI, dynamically allocated)
4. During unattended setup, set the hostname and local admin credentials
5. Select **Windows Server 2025 Standard Evaluation (Desktop Experience)** as the image
6. Let setup complete and log in

### 2. Configure Networking on DC01
1. Rename the computer to DC01 (System Properties → Change)
2. Set a static IP:
   - IP address: 192.168.10.10
   - Subnet mask: 255.255.255.0
   - Preferred DNS server: 192.168.10.10 (itself)
3. Verify with:
   ```
   ipconfig /all
   ```

### 3. Install AD DS and Promote to a Domain Controller
1. Server Manager → **Add roles and features** → select **Active Directory Domain Services**
2. Accept the additional required features (GPMC, AD DS/AD LDS Tools, AD Administrative Center)
3. Once installed, click the notification flag → **Promote this server to a domain controller**
4. Choose **Add a new forest**, root domain name: jflaum.cyber
5. Set forest/domain functional level to **Windows Server 2025**, keep **DNS Server** checked
6. Set a DSRM password (separate from the domain admin password — write it down)
7. Let the server restart to complete promotion

### 4. Verify the Domain
```
systeminfo | findstr /B /C:"Domain"
```
Expected output: Domain: jflaum.cyber

### 5. Create a Domain User
1. **Active Directory Users and Computers** → right-click jflaum.cyber → **New → User**
2. Fill in name and logon name (e.g. jdoe)
3. Set a password and finish

### 6. Build and Join the Windows 11 Client
1. Create a Windows 11 VM on the same VirtualBox internal network as DC01
2. Set a static IP 192.168.10.20/24, DNS pointed at 192.168.10.10
3. **System Properties → Change → Member of: Domain** → enter jflaum.cyber
4. Authenticate with domain admin credentials when prompted, then restart
5. At the lock screen, switch user and log in as JFLAUM\jdoe to confirm domain authentication works

---
*Part of a larger home lab — see also: [Group Policy & Security Hardening](README-group-policy-security.md), [Vulnerability Assessment Using Nessus Via Kali Linux](README-kali-nessus.md)*
