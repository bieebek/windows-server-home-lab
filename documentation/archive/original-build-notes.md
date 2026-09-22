> **Historical draft — superseded.** See the [current walkthroughs](../README.md) and [corrections](README.md). Status claims and screenshot descriptions below are retained as historical notes, not verified conclusions. Password arguments have been removed from command examples.

# Active Directory Home Lab — Build Documentation

**Author:** Bibek Maharjan
**Environment:** Legion Go (handheld PC used as desktop), Pop!_OS Linux, 16GB RAM (12GiB usable due to iGPU memory reservation)
**Base project:** [alebov/AD-lab](https://github.com/alebov/AD-lab) (Vagrant + Ansible automated AD lab)
**Status:** In progress — Domain Controller provisioning underway

---

## 1. Goal

Build a working Active Directory domain lab (Domain Controller + Windows 10 workstation) to demonstrate real, hands-on Windows Server / AD administration skills for IT support / help desk job applications, including internal transfer prospects at Timmins and District Hospital (TADH).

---

## 2. Environment Setup

### 2.1 Hardware constraint discovered
- Legion Go reports `free -h` total as **12Gi**, not the advertised 16GB.
- Root cause: AMD Z1 Extreme APU reserves a portion of system RAM for the integrated GPU (UMA frame buffer), which is invisible to the OS as usable system RAM.
- Decision: **left BIOS/UEFI settings untouched** (too time-risky to adjust on a handheld device mid-project) and instead trimmed the lab's resource footprint to fit comfortably within available RAM.

### 2.2 Tools installed
- **VirtualBox** — hypervisor (`sudo apt install virtualbox virtualbox-ext-pack`)
- **Vagrant** — VM automation, installed via HashiCorp's official APT repo (Pop!_OS repos don't carry the latest version)
- **Ansible** — configuration management, installed via `ppa:ansible/ansible`
- **freerdp2-x11** — for RDP access into the Windows VMs from Pop!_OS

### 2.3 Repo and Vagrantfile modifications
Cloned: `git clone https://github.com/alebov/AD-lab.git`

The original Vagrantfile defines **5 VMs**: `dc`, `win_server`, `win_workstation`, `ubuntu_domain`, `ubuntu_outside` — too heavy for available RAM.

**Trimmed to 2 VMs:** `dc` (Windows Server 2019) + `win_workstation` (Windows 10), commenting out the other three.

**Memory tuning:**
- Default was `1024MB` / `1 CPU` per VM
- First increased to `2048MB` / `2 CPUs` per VM (assumed 16GB available)
- After discovering real usable RAM was ~12GB, **reduced to `1536MB` / `2 CPUs` per VM** (3GB total for both VMs) to leave safe headroom for the host OS

---

## 3. Bringing the VMs Up

```bash
vagrant up dc win_workstation
```

- Both VMs downloaded (`StefanScherer/windows_2019`, `StefanScherer/windows_10` boxes), booted, and WinRM configured successfully.
- Confirmed both reachable via RDP:
```bash
xfreerdp /v:localhost:23389 /u:vagrant /cert:ignore   # dc
xfreerdp /v:localhost:43389 /u:vagrant /cert:ignore   # win_workstation
```
- **Result: both VMs confirmed up and visually accessible.**

**Note:** RDP sessions and the VMs themselves are independent — closing an RDP window does not shut down the VM. VMs run under VirtualBox regardless of terminal/RDP session state.

---

## 4. Ansible Provisioning — Issues Encountered & Fixes

Running `ansible-playbook -i hosts domain_controller.yml` surfaced a series of version-compatibility issues, since the AD-lab repo was written for older Ansible collection versions than what installs by default today. Each was resolved incrementally:

### 4.1 `ansible.windows.win_domain` module removed
**Error:** `The 'ansible.windows.win_domain' module has been removed. Use microsoft.ad.domain instead.`
**Cause:** Newer `ansible.windows` collection (3.0.0+) removed several `win_*` domain modules in favor of a new `microsoft.ad` collection.
**Fix:** Pinned to an older collection version that still includes the needed modules:
```bash
ansible-galaxy collection install ansible.windows:2.8.0 --force
```

### 4.2 `community.windows.win_domain_user` module removed
**Error:** Same pattern, different collection.
**Fix:**
```bash
ansible-galaxy collection install community.windows:2.4.0 --force
```

### 4.3 `xRemoteDesktopAdmin` PowerShell module install failure
**Error:** `A parameter cannot be found that matches parameter name 'AcceptLicense'`
**Cause:** Known bug in `win_psmodule` (community.windows) passing an `-AcceptLicense` flag incompatible with the box's older PowerShellGet version.
**Fix (interim):** Commented out the "Enable Remote Desktop via DSC" tasks in `roles/common/tasks/main.yml`, since RDP was already confirmed working without it.

### 4.4 `xNetworking` PowerShell module — same `AcceptLicense` error
**Cause:** Same root cause as 4.3, but this task was required (not skippable).
**Real fix:** Downgraded `community.windows` further, to the version confirmed working by the community for this exact bug:
```bash
ansible-galaxy collection install community.windows:1.10.0 --force
```
This resolved the `AcceptLicense` error across **all** `win_psmodule` tasks in the playbook, not just this one.

### 4.5 Chocolatey install failure — .NET Framework requirement
**Error:** `Chocolatey 2.0.0 requires .NET Framework 4.8 or higher... specify a 1.x version of Chocolatey to install.`
**Cause:** Default Chocolatey install pulls the latest version (2.0.0+), which needs .NET 4.8 — not present on this box by default.
**Fix:** Pinned Chocolatey itself to a 1.x version in `roles/domain_controller/tasks/main.yml`:
```yaml
- name: Ensure chocolatey is installed
  win_chocolatey:
    name: chocolatey
    version: '1.4.0'
    state: present

- name: Ensure chocolatey-core.extension is installed
  win_chocolatey:
    name: chocolatey-core.extension
    state: present
```

### 4.6 `sysinternals` package checksum mismatch
**Error:** `Checksum for '...SysinternalsSuite.zip' did not meet '...' for checksum type 'sha256'`
**Cause:** Upstream Chocolatey package definition's expected checksum doesn't match the file Microsoft is currently serving — a stale/broken package definition, unrelated to lab configuration.
**Fix:** Removed `sysinternals` from the package list entirely (not required for AD/domain functionality):
```yaml
with_items:
- notepadplusplus
- putty
- python
- git
- 7zip
# - sysinternals   <-- removed, checksum mismatch upstream
- wget
- pstools
```

### 4.7 DNS Forwarders `win_dsc` task — multiple parameter errors
**Errors (in sequence):**
1. `Unsupported parameters for (win_dsc) module: Name` — removed the `Name: DNSServerProperties` line
2. `missing required arguments: DnsServer` — this DSC resource version required a `DnsServer` parameter not provided in the original task
3. Attempted `ignore_errors: yes` but initially misplaced it *inside* the `win_dsc:` parameter block instead of at task level, causing `Unsupported parameters for (win_dsc) module: ignore_errors`
**Cause:** The `xDnsServerSetting` DSC resource version installed on this box has a different parameter set than the original playbook author's version — a version-drift issue similar to the earlier Ansible collection mismatches, but at the underlying PowerShell DSC module level.
**Fix:** Correctly placed `ignore_errors: yes` at task level (same indentation as `name:`), so the play continues even when this optional DNS-forwarding config fails:
```yaml
- name: Configure DNS Forwarders
  win_dsc:
    resource_name: xDnsServerSetting
    NoRecursion: false
    Forwarders:
      - "8.8.8.8"
      - "8.8.4.4"
  ignore_errors: yes
```
**Result:** Task fails and is reported as `...ignoring` in playbook output, but does not block subsequent tasks (domain admin/user/group creation). This is acceptable since DNS forwarding to public resolvers is a nice-to-have, not required for internal AD functionality.

### 4.8 VM going unresponsive / WinRM timeouts after idle periods
**Symptom:** `vagrant status` still shows `running`, but Ansible fails with `winrm connection error: ... Read timed out` or `Connection ... timed out`, and RDP may show `LOGON_FAILED_OTHER`.
**Cause:** Likely the host machine (Legion Go) sleeping/suspending while VMs were running, leaving VirtualBox VMs in an unresponsive state rather than a clean pause.
**Fix pattern:**
```bash
vagrant reload <vm_name>       # graceful restart; if it hangs on shutdown:
vagrant halt -f <vm_name>      # force power-off
vagrant up <vm_name>           # boot fresh
```
Then retry the Ansible playbook — a fresh boot resolves WinRM responsiveness reliably.
**Prevention going forward:** disable host sleep/suspend during lab sessions, or run `vagrant halt` cleanly before stepping away for extended periods.

### 4.9 Domain Controller RDP login failure after domain promotion
**Symptom:** `xfreerdp ... /u:vagrant` returns `LOGON_FAILED_OTHER` when connecting to the DC.
**Cause:** Once a server is promoted to a Domain Controller, local account authentication behaves differently — the `vagrant` local account context shifts once the domain exists.
**Fix:** Use domain-qualified credentials instead, e.g.:
```bash
xfreerdp /v:localhost:23389 /u:CYBERLOOP\\Administrator /cert:ignore
```
(Password as set by the "Ensure that Administrator is present with a valid password" task in the playbook.)

---

## 5. Current Status (updated — Domain fully operational)

- ✅ VirtualBox, Vagrant, Ansible installed and working
- ✅ Vagrantfile trimmed to 2 VMs, memory tuned to fit hardware (1536MB × 2)
- ✅ Both `dc` and `win_workstation` VMs up, booted, WinRM configured, RDP-accessible
- ✅ Ansible collection version conflicts resolved (`ansible.windows:2.8.0`, `community.windows:1.10.0`)
- ✅ Chocolatey install fixed (pinned to 1.4.0)
- ✅ Package install loop completing (notepadplusplus, putty, python, git, 7zip, wget, pstools installed; sysinternals removed)
- ✅ **Hostname changed to `domain-controller`, rebooted successfully**
- ✅ **Local Administrator password set**
- ✅ **`cyberloop.local` domain created** (via `win_domain`, after fixing DNS Forwarders `win_dsc` errors — see 4.7 below)
- ✅ **Server promoted to Domain Controller** (`win_domain_controller` task — `ok`, confirmed via full playbook run with `failed=0`)
- ✅ **Domain admin `admin@cyberloop.local` created**
- ✅ **Users `bob@cyberloop.local` and `alice@cyberloop.local` created** in `cn=Users,dc=CYBERLOOP,dc=local`
- ✅ **Domain groups created:** AllTeams, DBAOracle, DBASQLServer, DBAMongo, DBARedis, DBAEnterprise, Test Group
- ✅ **`win_workstation.yml` playbook completed with `failed=0`:**
  - Hostname changed to `win-workstation-1`
  - Chocolatey installed, packages installed (notepadplusplus, git, pstools)
  - DNS configured to point at the DC
  - Public network share created
  - **Workstation successfully joined to `cyberloop.local` domain**
  - `bob@cyberloop.local` added as local administrator on the workstation
- 🔲 **Next:** Verify domain join visually (RDP into workstation as `CYBERLOOP\bob`, confirm domain shows in System Properties)
- 🔲 **Next:** Explore Active Directory Users and Computers on the DC to confirm all users/groups are visible
- 🔲 **Not yet started:** Custom OU structure, additional GPOs, intentional break/fix troubleshooting scenario, PowerShell automation scripts, osTicket deployment, network security lab (pfSense), documentation diagrams

**Milestone: the core AD domain infrastructure (DC + domain-joined client, real users, real groups) is fully functional.** Remaining work is hands-on AD administration and the other separate lab projects (osTicket, pfSense, documentation), not further Vagrant/Ansible troubleshooting.

---

## 6. Key Learnings So Far

- Older open-source Ansible/Vagrant lab repos frequently break against current tool versions — collection version pinning (`ansible-galaxy collection install <name>:<version> --force`) is the standard fix pattern.
- `win_psmodule`'s `AcceptLicense` bug is a known, documented issue in `community.windows` versions 1.11.0+; version 1.10.0 avoids it.
- Chocolatey 2.x requires .NET Framework 4.8; older Windows Server images may need Chocolatey pinned to a 1.x release instead.
- PowerShell DSC resource modules (like `xDnsServerSetting`) can have differing parameter sets across versions, independent of the Ansible collection issues — same underlying "tooling has moved on since this repo was written" pattern.
- `ignore_errors: yes` must be placed at task level (same indentation as `name:`), not nested inside a module's parameter block.
- Hardware-reported RAM (`free -h`) can differ meaningfully from advertised specs on handheld/APU devices due to iGPU memory reservation — worth checking before sizing VMs.
- RDP sessions are independent of VM lifecycle — closing a viewer does not stop the VM.
- Once a server becomes a Domain Controller, local-account RDP behavior changes — use domain-qualified credentials (`DOMAIN\user`) afterward.
- VMs can become unresponsive (WinRM timeouts, failed RDP) after a host sleep/suspend event — `vagrant reload` (or `halt -f` + `up` if reload hangs) reliably recovers them.
- **Overall pattern:** nearly every error in this build was a version-compatibility issue between the (older) lab repo and (newer) currently-installed tooling — not a conceptual misunderstanding of AD itself. This is a normal, expected part of working with open-source infrastructure-as-code projects, and each fix followed the same diagnostic approach: read the exact error, identify what changed upstream, pin or patch accordingly.

---

## 7. Milestone Reached: Working AD Domain

As of this session, `cyberloop.local` is a fully functional Active Directory domain with:
- One Domain Controller (`domain-controller`, Windows Server 2019)
- One domain-joined client (`win-workstation-1`, Windows 10)
- Domain admin account (`admin`) and two standard users (`bob`, `alice`)
- Seven domain groups (AllTeams, DBAOracle, DBASQLServer, DBAMongo, DBARedis, DBAEnterprise, Test Group)
- `bob` granted local administrator rights on the workstation

## 8. How to Resume This Session

1. Ensure VirtualBox VMs are still running: `vagrant status` (from `~/AD-lab`)
2. If stopped or unresponsive: `vagrant reload dc` / `vagrant reload win_workstation` (or `vagrant halt -f <vm>` then `vagrant up <vm>` if reload hangs)
3. Verify domain join: RDP into the workstation as `CYBERLOOP\bob` and confirm the domain shows under System Properties
4. Explore Active Directory Users and Computers on the DC (RDP as `CYBERLOOP\Administrator`) to see the existing users/groups
5. **Begin hands-on AD administration work:**
   - Design and create a custom OU structure (e.g., "IT Support," "Hospital Staff," "Contractors")
   - Create additional users/groups reflecting that structure
   - Build 2-3 GPOs (password policy, drive mapping, restricted access)
   - Intentionally misconfigure a GPO, then diagnose and fix it using `gpupdate /force`, `gpresult /r`, and Event Viewer — document this as the core "break/fix" interview story
6. Once AD work is documented, move to the next project in sequence: PowerShell automation scripts, then osTicket, then the pfSense network security lab, then draw.io documentation diagrams.

---

## 9. Screenshot Guide — What to Capture and Where

Screenshots are what turn this document from "a log of commands" into real proof for your GitHub repo and interviews. Take them **as you go**, not at the end — it's easy to forget what a broken state looked like once it's fixed.

### Folder structure (create this now)
```
AD-lab/
  screenshots/
    01-vm-setup/
    02-dc-promotion/
    03-domain-join/
    04-ou-users-groups/
    05-gpo-config/
    06-breakfix/
```
Name files descriptively, e.g. `03-domain-join_workstation-sysdm-cpl.png` — future-you (and recruiters) shouldn't have to guess what `Screenshot_2026-08-15_142233.png` shows.

### What to capture, section by section

**01-vm-setup**
- `vagrant status` showing both VMs `running`
- `free -h` output (shows you understood your hardware constraints — good talking point)

**02-dc-promotion**
- Terminal output of the final successful `ansible-playbook -i hosts domain_controller.yml` run showing `failed=0`
- Inside the DC (RDP): Server Manager showing the AD DS role installed and the server listed as a Domain Controller
- `dcdiag` output from an elevated PowerShell/cmd prompt on the DC (run this now — it's a standard AD health-check command and a great screenshot to have: `dcdiag /v`)

**03-domain-join**
- Terminal output of the successful `win_workstation.yml` run (`failed=0`)
- On the workstation (RDP): System Properties (`sysdm.cpl`) showing `cyberloop.local` as the domain instead of "WORKGROUP"
- `whoami /fqdn` or `echo %USERDOMAIN%` run in cmd on the workstation, proving domain membership from the command line

**04-ou-users-groups**
- Active Directory Users and Computers (`dsa.msc`) on the DC, showing your custom OU structure once built
- The same tool showing your user accounts inside their OUs with relevant attributes filled in (department, title)
- PowerShell: `Get-ADUser -Filter * | Select Name,Enabled` output — shows you can verify AD state via command line, not just GUI

**05-gpo-config**
- Group Policy Management Console (`gpmc.msc`) showing your created GPOs linked to the right OUs
- The actual GPO settings screen for one policy (e.g., password complexity settings)
- On the workstation, proof the policy applied — e.g. `gpresult /r` output, or the Control Panel restriction actually working/blocking as configured

**06-breakfix — your most valuable set**
- Screenshot of the GPO misconfigured (before state)
- Screenshot of the resulting failure/error the user would see
- Event Viewer entry showing the relevant error/warning
- `gpresult /r` or `gpupdate /force` output during diagnosis
- Screenshot after the fix, showing it now works correctly
- This sequence — break, diagnose, fix — is the single best thing you can show an interviewer, so don't skip any step in the middle even if it feels repetitive

