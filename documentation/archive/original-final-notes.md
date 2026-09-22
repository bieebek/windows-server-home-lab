> **Historical draft — superseded.** See the [current walkthroughs](../README.md) and [corrections](README.md). Status claims and screenshot descriptions below are retained as historical notes, not verified conclusions. Password arguments have been removed from command examples.

# Active Directory Home Lab — Full Documentation

**Author:** Bibek Maharjan
**Environment:** Legion Go (handheld PC used as desktop), Pop!_OS Linux, 16GB RAM (12GiB usable due to iGPU memory reservation)
**Base project:** [alebov/AD-lab](https://github.com/alebov/AD-lab) (Vagrant + Ansible automated AD lab)
**Status:** Core domain complete, custom OUs/GPOs built, break/fix scenario investigated

---

## Reading this historical draft

Screenshot evidence is linked in the current walkthroughs.

---

## 1. Project Goal

Build a working Active Directory domain lab (Domain Controller + Windows 10 workstation) to demonstrate real, hands-on Windows Server / AD administration skills for IT support / help desk job applications, including an internal transfer pathway at Timmins and District Hospital (TADH).

---

## 2. Environment Setup

- **Hardware constraint discovered:** Legion Go reports 12GiB usable RAM (not the advertised 16GB) due to AMD Z1 Extreme iGPU memory reservation. Decision made to leave BIOS untouched and instead trim the lab's footprint to fit safely.
- **Tools installed:** VirtualBox, Vagrant (via HashiCorp's official APT repo), Ansible (via `ppa:ansible/ansible`), freerdp2-x11 for RDP access.
- **Vagrantfile modifications:** Trimmed the original 5-VM lab down to 2 VMs (`dc`, `win_workstation`), commenting out `win_server`, `ubuntu_domain`, `ubuntu_outside`. Memory tuned from default `1024MB/1CPU` to `1536MB/2CPU` per VM to safely fit available RAM.

**Screenshot 1.1 — Both VMs running**
`vagrant status` showing `dc` and `win_workstation` as `running`
See the [current screenshot index](../../screenshots/README.md).

**Screenshot 1.2 — RAM check**
`free -h` output
See the [current screenshot index](../../screenshots/README.md).

---

## 3. Domain Controller Provisioning

Ran `ansible-playbook -i hosts domain_controller.yml`. This repo was written for older Ansible/Chocolatey/PowerShell tooling versions, so provisioning surfaced a series of version-compatibility issues, each resolved in turn:

| # | Issue | Fix |
|---|---|---|
| 1 | `ansible.windows.win_domain` module removed | Pinned `ansible-galaxy collection install ansible.windows:2.8.0 --force` |
| 2 | `community.windows.win_domain_user` module removed | Pinned `ansible-galaxy collection install community.windows:2.4.0 --force` |
| 3 | `win_psmodule` `AcceptLicense` parameter error | Further pinned `community.windows:1.10.0` (known fix for this bug) |
| 4 | Chocolatey 2.0.0 requires .NET Framework 4.8 | Pinned Chocolatey install to version `1.4.0` in the playbook task |
| 5 | `sysinternals` package checksum mismatch | Removed `sysinternals` from the package list (stale upstream checksum, non-essential package) |
| 6 | DNS Forwarders `win_dsc` task — parameter errors (`Name`, then `DnsServer` missing) | Added `ignore_errors: yes` at correct task-level indentation; DNS forwarding is optional and not required for internal AD function |
| 7 | VMs going unresponsive after host sleep/idle (WinRM timeouts, RDP `LOGON_FAILED_OTHER`) | `vagrant reload <vm>`, or `vagrant halt -f <vm>` + `vagrant up <vm>` if reload hangs |

**Result:** Playbook completed with `failed=0`. Domain `cyberloop.local` created, DC promoted, domain admin `admin` created, users `bob` and `alice` created, domain groups created (AllTeams, DBAOracle, DBASQLServer, DBAMongo, DBARedis, DBAEnterprise, Test Group).

**Screenshot 3.1 — Successful DC playbook run**
Terminal output showing `PLAY RECAP` with `failed=0`
See the [current screenshot index](../../screenshots/README.md).

**Screenshot 3.2 — Server Manager on the DC**
Shows AD DS role installed, server recognized as Domain Controller
See the [current screenshot index](../../screenshots/README.md).

**Screenshot 3.3 — dcdiag health check**
Output of `dcdiag` run on the DC, confirming domain health
See the [current screenshot index](../../screenshots/README.md).

---

## 4. Workstation Domain Join

Ran `ansible-playbook -i hosts win_workstation.yml`. Completed with `failed=0` — hostname changed to `win-workstation-1`, packages installed, DNS pointed at the DC, workstation joined to `cyberloop.local`, `bob` added as local administrator.

**Screenshot 4.1 — Successful workstation playbook run**
See the [current screenshot index](../../screenshots/README.md).

**Screenshot 4.2 — System Properties showing domain membership**
`sysdm.cpl` on the workstation, showing `cyberloop.local` instead of "WORKGROUP"
See the [current screenshot index](../../screenshots/README.md).

**Screenshot 4.3 — Command-line proof of domain membership**
`echo %USERDOMAIN%` run in Command Prompt on the workstation
See the [current screenshot index](../../screenshots/README.md).

---

## 5. Users, Groups & OU Structure

Built a custom OU structure to reflect a realistic organizational layout:
- **IT Support**
- **Hospital Staff**
- **Contractors**

Created a test user (`IT Tech` / `ittech`) inside the IT Support OU.

**Screenshot 5.1 — Active Directory Users and Computers, users view**
Shows `admin`, `alice`, `bob` and built-in accounts
See the [current screenshot index](../../screenshots/README.md).

**Screenshot 5.2 — Custom groups (found via Action → Find)**
Shows AllTeams, DBAOracle, DBASQLServer, DBAMongo, DBARedis, DBAEnterprise, Test Group
See the [current screenshot index](../../screenshots/README.md).

**Screenshot 5.3 — OU structure**
Shows IT Support, Hospital Staff, Contractors OUs under `cyberloop.local`
See the [current screenshot index](../../screenshots/README.md).

**Screenshot 5.4 — PowerShell AD query (optional)**
`Get-ADUser -Filter * | Select Name,Enabled`
See the [current screenshot index](../../screenshots/README.md).

---

## 6. Group Policy Configuration

Created a GPO ("IT Support - Password Policy") linked to the IT Support OU, configuring:
- Minimum password length
- Account lockout threshold (set aggressively to 1 attempt, for a later break/fix test)

**Screenshot 6.1 — Group Policy Management Console**
Shows the GPO linked to the IT Support OU, status Enabled
See the [current screenshot index](../../screenshots/README.md).

**Screenshot 6.2 — GPO password/lockout settings**
The actual policy configuration screen
See the [current screenshot index](../../screenshots/README.md).

**Screenshot 6.3 — gpresult on the workstation**
`gpresult /r` output after `gpupdate /force`
See the [current screenshot index](../../screenshots/README.md).

---

## 7. Break / Fix Investigation — RDP Access for a New Domain User

This is the core troubleshooting story from this project.

**The problem:** After creating `ittech` in the IT Support OU, RDP login attempts to the workstation consistently failed with authentication errors (`STATUS_LOGON_FAILURE`, and at one point `STATUS_PASSWORD_MUST_CHANGE`), even though a different account (`vagrant`) connected to the same workstation without issue.

**Diagnostic steps taken, in order:**
1. Verified the DC itself was reachable and other accounts could RDP in successfully — ruled out a broader network/VM problem.
2. Reset `ittech`'s password to a simple known value, ruling out a typo or special-character encoding issue in the original password.
3. Checked the account's **Account** tab in Active Directory Users and Computers for lockout status and "must change password" flags.
4. Checked **System Properties → Remote → Select Users** on the workstation and found `ittech` was **not** in the list of users permitted to RDP in — added it.
5. RDP still failed after that fix, so connected via the **VirtualBox console directly** (bypassing RDP/NLA entirely) — confirmed `ittech`'s credentials and account status were completely valid, isolating the problem specifically to the RDP/NLA authentication layer rather than anything wrong with the account or Active Directory itself.

**Screenshot 7.1 — The misconfigured/tested policy (before)**
Account lockout policy set to threshold 1, from Section 6
See the [current screenshot index](../../screenshots/README.md).

**Screenshot 7.2 — The problem, shown clearly**
Windows login screen showing an authentication failure for `ittech`
See the [current screenshot index](../../screenshots/README.md).

**Screenshot 7.3 — Diagnosis: RDP permissions check**
System Properties → Remote → Select Users dialog
See the [current screenshot index](../../screenshots/README.md).

**Screenshot 7.4 — Diagnosis: account status check**
Active Directory Account tab for `ittech`, showing lockout/password flags
See the [current screenshot index](../../screenshots/README.md).

**Screenshot 7.5 — Confirmed account validity via console**
Successful direct console login as `ittech`, bypassing RDP entirely
See the [current screenshot index](../../screenshots/README.md).

**Outcome:** Root cause narrowed to the RDP/NLA authentication layer specifically — not the AD account, password, lockout state, or RDP group membership, all of which were individually verified and ruled out. A next step for further investigation would be checking Local Security Policy (`secpol.msc → Local Policies → User Rights Assignment → Allow log on through Remote Desktop Services`) on the workstation, which is a separate setting from the "Remote Desktop Users" group already checked.

---

## 8. Write-Up

**What I built:**
A working Active Directory domain (`cyberloop.local`) with a promoted Domain Controller and a domain-joined Windows 10 workstation, using Vagrant and Ansible for automated provisioning. Built a custom OU structure (IT Support, Hospital Staff, Contractors), domain users and groups, and a Group Policy Object enforcing password and account lockout policies.

**Biggest problem I ran into and how I fixed it:**
After creating a new domain user inside the IT Support OU, I couldn't get it to log in via RDP — it consistently failed with authentication errors, even after resetting the password and confirming the account wasn't locked or disabled. I methodically ruled out causes one by one: verified the password was correct, checked account lockout status, and confirmed the account's RDP permissions (found it was missing from the "Remote Desktop Users" list and added it). When RDP still failed, I logged into the VM directly through the VirtualBox console instead of over RDP — confirming the account and password were genuinely fine, and isolating the problem specifically to the RDP/NLA authentication layer rather than anything wrong with Active Directory itself.

**What I'd do differently next time:**
I'd screenshot each troubleshooting step as I go, rather than reconstructing them afterward — capturing state live is much easier than recreating it once an issue is understood. I'd also test new accounts against multiple login methods (console and RDP) earlier in the process, since it would have shown faster that the account itself was fine and narrowed the investigation sooner.

---

## 9. Key Learnings

- Older open-source Ansible/Vagrant lab repos frequently break against current tool versions — collection version pinning is the standard fix pattern.
- Chocolatey 2.x requires .NET Framework 4.8; older Windows images may need Chocolatey pinned to a 1.x release.
- `ignore_errors: yes` must sit at task level, not nested inside a module's parameter block.
- RDP sessions are independent of VM lifecycle — closing a viewer does not stop the VM.
- Once a server becomes a Domain Controller, local-account RDP behavior changes — domain-qualified credentials (`DOMAIN\user`) may be required.
- RDP's NLA (Network Level Authentication) can fail in ways that are hard to diagnose from the client side alone — connecting via the hypervisor's console directly is a valuable way to isolate whether a problem is with an account/credentials or with the RDP protocol layer itself.
- A new domain user needs explicit RDP permission (membership in "Remote Desktop Users" or equivalent) — domain membership alone does not grant remote login rights.
- Not every troubleshooting session ends in a fully confirmed fix — documenting a well-reasoned, methodical investigation is still valuable and realistic.
