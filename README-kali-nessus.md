# Vulnerability Assessment Using Nessus Via Kali Linux

Standing up Kali Linux as the attacker-perspective box for the lab, then using Nessus to run a vulnerability scan against the domain-joined Windows 11 client.

## What I Built

- A **Kali Linux VM** in VirtualBox with two network adapters:
  - Adapter 1 (NAT) for internet access and updates
  - Adapter 2 (Internal Network) on the same 192.168.10.0/24 segment as DC01 and the Windows 11 client
- **Nessus Essentials** installed on Kali, accessed locally via https://localhost:8834
- A **Basic Network Scan** policy targeting the Windows 11 client (192.168.10.20)
- A completed unauthenticated scan returning 15 unique findings (30 total) across the host
- A review of each finding family (Windows, port scanners, service detection) and a drill-down into individual plugin output (e.g. SMB service detection)

## Architecture

```
                         Internet
                            |
                      NAT (Adapter 1)
                            |
                       Kali Linux
                 (Nessus, Nmap, Metasploit)
                            |
                Internal Network (Adapter 2)
                     192.168.10.0/24
                            |
              +-------------+-------------+
              |                           |
        Windows Server               Windows 11
        192.168.10.10               192.168.10.20
        (DC01 — jflaum.cyber)        (domain-joined client,
                                       scan target)
```

| Host | Role | IP |
|------|------|-----|
| Kali Linux | Attacker-perspective / scanning box | NAT + 192.168.10.x (internal) |
| Windows 11 Client | Scan target | 192.168.10.20 |
| DC01 | Domain controller (in scope for future scans) | 192.168.10.10 |

## What I Learned (and Why It Matters)

- **A dual-NIC attacker box mirrors how real engagements are scoped.** Giving Kali a NAT adapter for tooling/updates and a separate internal adapter for the target network is the same separation you'd want on a real assessment — your scanning traffic stays on the engagement network, and your own outbound traffic doesn't leak into it.
- **Unauthenticated scans only see what's exposed externally.** Every finding from the Basic Network Scan came back at **Info** severity — mostly service/port detection (SMB, general OS fingerprinting) rather than exploitable vulnerabilities. That's expected for a default, unpatched-but-current Windows 11 install scanned without credentials: Nessus can fingerprint what's listening, but can't see patch levels, installed software, or local misconfigurations without authenticating to the host.
- **Severity context matters more than the finding count.** "30 findings" sounds alarming until you look at the severity breakdown — going straight to the dashboard's vulnerability count without checking the severity distribution is how vulnerability data gets over- or under-reacted to in practice.
- **Drilling into individual plugin output is where the real information is.** The summary view groups findings by family; the actual plugin detail (description, affected port/host, risk factor) is what you'd hand to someone doing remediation. Treating the summary as the whole report misses the actionable part.
- **Authenticated scanning is the natural next step.** To see missing patches, weak local configurations, and installed-software issues — the things that actually matter for a domain-joined production-style machine — the scan needs to run with credentials. This is the clearest gap between what I did and what a real assessment would require.

## Deployment Instructions

> Prerequisite: a working jflaum.cyber domain with a Windows 11 client at 192.168.10.20. See [README-active-directory.md](README-active-directory.md).

### 1. Create the Kali Linux VM
1. VirtualBox → **New**, Type: Linux, Version: Debian (64-bit)
2. Allocate at least **2048 MB RAM** (more improves tool performance)
3. Create an **80 GB** virtual disk (VDI, dynamically allocated)
4. Mount the Kali ISO and proceed through the install wizard as a standard bare-metal install
5. **Settings → Network:**
   - Adapter 1: **NAT**
   - Adapter 2: **Internal Network**, same network name as the one DC01/Client01 use, with a static IP on 192.168.10.0/24 (e.g. 192.168.10.30)

### 2. Install and Launch Nessus Essentials
1. Register for a free Nessus Essentials activation code at [tenable.com/products/nessus/nessus-essentials](https://www.tenable.com/products/nessus/nessus-essentials)
2. Download the Kali/Debian .deb package and install:
   ```bash
   sudo dpkg -i Nessus-<version>-debian10_amd64.deb
   sudo systemctl start nessusd
   ```
3. Open https://localhost:8834 in a browser on Kali, complete setup with the activation code, and create a login

### 3. Create and Run a Basic Network Scan
1. **Scans → New Scan → Basic Network Scan**
2. Name it (e.g. "My Basic Network Scan"), set the target to 192.168.10.20
3. Save, then **Launch**
4. Scan typically completes in ~10 minutes for a single host

### 4. Review Findings
1. Open the completed scan → **Hosts** tab to see total vulnerability count per host
2. **Vulnerabilities** tab to see findings grouped by severity and family
3. Click into individual findings for plugin ID, description, affected ports/hosts, and risk factor
4. Note that unauthenticated scans will skew toward Info-severity service/port detection — for deeper findings, configure scan credentials under **Configure → Credentials** and re-run

### 5. (Next Step) Run an Authenticated Scan
1. **Configure → Credentials → Windows**, supply a domain account with local admin rights on the target
2. Re-launch the scan and compare findings — authenticated scans surface missing patches, weak configurations, and installed-software issues that unauthenticated scans can't see

---
*Part of a larger home lab — see also: [Setting up Windows 2025 Domain Controller + Joining a Windows 11 Client + Verifying the Domain is Live](README-active-directory.md), [Group Policy & Security Hardening](README-group-policy-security.md)*
