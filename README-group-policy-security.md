# Group Policy & Security Hardening — Home Lab

Designing a realistic OU/security-group structure on top of the jflaum.cyber domain, then using Group Policy to enforce least privilege and harden every domain-joined machine — password policy, logon-hour restrictions, screen lock, Defender AV, the firewall, AppLocker, USB blocking, audit logging, and local security hardening.

## What I Built

- An **OU structure** mirroring a small organization: Exec, MiddleManagement, Sales, IT, Security, HR, Finance, Servers, Workstations
- Matching **security groups** (GRP_Exec, GRP_ITAdmins, GRP_StandardUsers, GRP_SecOps, etc.) for role-based access control, with IT admins added to Domain Admins
- **GPO_PasswordPolicy** — 8+ character minimum, 24-password history, 180-day max age, complexity enabled
- **Account lockout policy** — 5 failed attempts, 30-minute lockout/reset, admin lockout enabled
- **GPO_LogonHours** — Mon–Fri 7am–6pm logon window for standard users (applied via PowerShell), with forced logoff when hours expire
- **GPO_ScreenLock** — 15-minute idle timeout with password-protected screen saver
- **GPO_DefenderAV** — Defender locked on, cloud-delivered protection (MAPS) at Advanced, Block at First Sight, scheduled weekly scans, removable-drive and download scanning
- **Windows Defender Firewall** enforced "On" across Domain, Private, and Public profiles, with local rule merging disabled
- **GPO_AppLocker** for application allow-listing
- **GPO_NoUSB** blocking removable storage on the more sensitive OUs (Exec, Finance, HR, MiddleManagement, Sales, Security)
- **GPO_RestrictControlPanel** to stop standard users from reaching Control Panel/PC settings
- **GPO_AuditPolicy** — advanced audit logging for logon events, sensitive privilege use, and audit policy changes
- **GPO_SecurityHardening** — renamed built-in Administrator account, disabled the Guest account, set an "Authorized Users Only" logon banner
- Verified everything with gpupdate /force and gpresult /r / gpresult /h

## Architecture

```
                              jflaum.cyber
                                  |
        +---------+---------+---------+---------+---------+--------+
        |         |         |         |         |         |        |
       Exec   MiddleMgmt  Sales       IT      Security    HR    Finance
        |                                                              
   GRP_Exec  ...        ...      GRP_ITAdmins  GRP_SecOps   ...    ...
   (Tier 1)             (Tier 3) (Tier 2)      (Tier 2)    (Tier 3)(Tier 3)

  Linked GPOs (domain-wide unless noted):
   GPO_PasswordPolicy     -> password + account lockout
   GPO_LogonHours         -> Mon-Fri 7am-6pm, forced logoff
   GPO_ScreenLock         -> 15-min idle lock
   GPO_DefenderAV         -> AV + cloud protection
   GPO_AuditPolicy        -> logon / privilege / policy-change auditing
   GPO_SecurityHardening  -> renamed admin, disabled guest, logon banner
   GPO_AppLocker          -> application allow-listing
   GPO_NoUSB              -> Exec, Finance, HR, MiddleMgmt, Sales, Security only
   GPO_RestrictControlPanel -> non-IT OUs

  Least-privilege tiers:
   Tier 1 (Exec)            -> broad read, restricted write
   Tier 2 (IT / Security)   -> admin rights, scoped to role
   Tier 3 (everyone else)   -> standard user, time-restricted logon
```

## What I Learned (and Why It Matters)

- **Design before you build.** Mapping the OU tree, GPO links, security groups, and access tiers out as a diagram *before* touching GPMC meant every GPO had an obvious home (which OU it links to, which group it targets) instead of being bolted on after the fact and fought with later.
- **Per-user logon hours aren't a GUI feature.** Active Directory Users and Computers exposes a per-user logon-hours grid, but applying it consistently across an entire group has to be scripted — Set-ADUser -LogonHours with a 21-byte bitmask, one bit per hour of the week. This is a good example of where the GUI works for one-offs but PowerShell is the only practical option at scale.
- **GPO scope and link order decide who wins.** Linking GPO_NoUSB only to the sensitive OUs (instead of domain-wide) while leaving IT and Workstations untouched is what kept the policy from blocking legitimate admin work — scope is as much a security decision as the setting itself.
- **Windows doesn't audit much by default.** Logon events, privilege use, and policy changes don't show up in the Security event log unless an audit policy explicitly turns them on. GPO_AuditPolicy is what makes the Security log actually useful for incident response later — without it, there'd be nothing to investigate.
- **"Turn off Defender" set to Disabled is not a typo.** The double-negative phrasing in that policy (disabling the *setting that turns off Defender*) is a common real-world gotcha — get it backwards and you've just told every client it's allowed to disable its own antivirus.
- **Trust, then verify.** The GPMC console showing a GPO as "linked" doesn't guarantee it applied. gpresult /r and the HTML RSoP report were what actually confirmed each policy landed on DC01 with no errors — the same check I'd want to run against a real production rollout before assuming it worked.

## Deployment Instructions

> Prerequisite: a working jflaum.cyber domain controller. See [README-active-directory.md](README-active-directory.md).

### 1. Plan the OU / Group Structure
Sketch the OU tree, the GPOs that will link to each OU, the security groups behind RBAC, and your access tiers before creating anything — this becomes your build checklist.

### 2. Build OUs, Users, and Groups
In **Active Directory Users and Computers**:
1. Right-click jflaum.cyber → **New → Organizational Unit** for each top-level OU (Exec, MiddleManagement, Sales, IT, Security, HR, Finance, Servers), each with its own Computers sub-OU
2. Inside each OU, **New → User** for representative accounts (e.g. CEO, VP of Sales, CFO in Exec)
3. Inside each OU, **New → Group** (Global, Security) named GRP_<OUName> (e.g. GRP_Exec, GRP_ITAdmins)
4. Add users to their group, and add IT admin accounts to **Domain Admins**

### 3. Password & Account Lockout Policy
In **Group Policy Management**:
1. Right-click jflaum.cyber → **Create a GPO in this domain, and Link it here** → name it GPO_PasswordPolicy
2. Edit → **Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies**
3. **Password Policy:** minimum length 8, max age 180 days, min age 1 day, history 24, complexity enabled
4. **Account Lockout Policy:** threshold 5 invalid attempts, duration/reset 30 minutes, enable Administrator account lockout

### 4. Logon Hours & Screen Lock
1. Create GPO_LogonHours, link to the domain. Under **Computer Configuration → Security Settings → Local Policies → Security Options**, enable **Network security: Force logoff when logon hours expire**
2. Apply per-user logon hours via PowerShell on DC01:
   ```powershell
   $group = Get-ADGroupMember -Identity "GRP_StandardUsers" -Recursive
   # Hours array: 21 bytes representing each hour of the week
   # Denied = bit 0, Allowed = bit 1 per hour slot
   $hours = [byte[]](0,0,0,0,0,224,255,3,224,255,3,224,255,3,224,255,3,224,0)
   foreach ($user in $group) {
       Set-ADUser -Identity $user.SamAccountName -LogonHours $hours
   }
   ```
3. Create GPO_ScreenLock, link to the domain. Under **Computer Configuration → Policies → Administrative Templates → Control Panel → Personalization**:
   - **Enable screen saver:** Enabled
   - **Password protect the screen saver:** Enabled
   - **Screen saver timeout:** 900 seconds

### 5. Endpoint Security
1. Create GPO_DefenderAV, link to the domain. Under **Computer Configuration → Administrative Templates → Windows Components → Microsoft Defender Antivirus**:
   - **Turn off Microsoft Defender Antivirus:** Disabled (i.e. keep Defender on)
   - **MAPS → Join Microsoft MAPS:** Advanced membership
   - **MAPS → Configure the 'Block at First Sight' feature:** Enabled
   - **Scan → Specify the day of the week to run a scheduled scan:** Saturday
   - **Real-time Protection / Scan:** enable scan of removable drives and downloaded files/attachments
2. Edit the same or a dedicated GPO under **Computer Configuration → Policies → Windows Settings → Security Settings → Windows Defender Firewall with Advanced Security**:
   - For Domain, Private, and Public profiles: Firewall state **On**, inbound **Block (default)**, outbound **Allow (default)**, disable local rule merging
3. Create GPO_AppLocker, configure rule enforcement under **Application Control Policies → AppLocker**
4. Create GPO_NoUSB, link **only** to Exec, Finance, HR, MiddleManagement, Sales, and Security OUs — block removable storage device classes under **Removable Storage Access**
5. Create GPO_RestrictControlPanel, link to non-IT OUs. Under **Administrative Templates → Control Panel**: **Prohibit access to Control Panel and PC settings:** Enabled

### 6. Audit Policy
1. Create GPO_AuditPolicy, link to the domain. Under **Computer Configuration → Policies → Windows Settings → Security Settings → Advanced Audit Policy Configuration → Audit Policies**:
   - **Logon/Logoff → Audit Logon:** Success and Failure
   - **Privilege Use → Audit Sensitive Privilege Use:** Failure
   - **Policy Change → Audit Audit Policy Change:** Success and Failure

### 7. Local Security Hardening
1. Create GPO_SecurityHardening, link to the domain. Under **Computer Configuration → Policies → Windows Settings → Security Settings → Local Policies → Security Options**:
   - **Accounts: Rename administrator account:** set to a non-default name
   - **Accounts: Guest account status:** Disabled
   - **Interactive logon: Message text for users attempting to log on:** "Authorized Users Only"

### 8. Verify
On DC01 or any domain-joined client:
```
gpupdate /force
gpresult /r
gpresult /h report.html
```
Confirm the expected GPOs appear under **Applied Group Policy Objects** with no errors reported.

---
*Part of a larger home lab — see also: [Setting up Windows 2025 Domain Controller + Joining a Windows 11 Client + Verifying the Domain is Live](README-active-directory.md), [Vulnerability Assessment Using Nessus Via Kali Linux](README-kali-nessus.md)*
