# Screenshot index

[Project overview](../README.md) · [Walkthroughs](../documentation/README.md)

All 23 supplied PNGs are preserved without image edits, grouped by milestone and renamed to match their visible contents. Click a filename for the full-size image.

| Original filename | Organized image | What it shows |
| --- | --- | --- |
| `1.1-vagrant-status.png` | [01-environment/vagrant-status.png](01-environment/vagrant-status.png) | Both VMs running |
| `1.2-ram-check.png` | [01-environment/host-memory.png](01-environment/host-memory.png) | Host memory check |
| `Screenshot_1.1.png` | [01-environment/workstation-boot.png](01-environment/workstation-boot.png) | Workstation boot and forwarded ports |
| `2.1-dc-playbook-success.png` | [02-domain-controller/dc-rdp-session.png](02-domain-controller/dc-rdp-session.png) | DC RDP session; no Ansible recap visible |
| `2.2-server-manager.png` | [02-domain-controller/server-manager.png](02-domain-controller/server-manager.png) | Windows Server 2019, AD DS and DNS |
| `3.1-workstation-playbook-success.png` | [03-domain-join/workstation-rdp-session.png](03-domain-join/workstation-rdp-session.png) | Workstation desktop; no Ansible recap visible |
| `3.2-sysdm-domain.png` | [03-domain-join/system-properties-domain.png](03-domain-join/system-properties-domain.png) | System Properties confirms cyberloop.local |
| `3.3-domain-cmdline.png` | [03-domain-join/local-user-context.png](03-domain-join/local-user-context.png) | USERDOMAIN reports local account context |
| `4.1-aduc-users.png` | [04-users-groups-and-ous/aduc-users.png](04-users-groups-and-ous/aduc-users.png) | AD Users and Computers |
| `4.2-aduc-groups.png` | [04-users-groups-and-ous/aduc-group-search.png](04-users-groups-and-ous/aduc-group-search.png) | Search for DBA groups |
| `4.3-get-aduser-powershell.png` | [04-users-groups-and-ous/powershell-user-query.png](04-users-groups-and-ous/powershell-user-query.png) | Get-ADUser query |
| `5 Users and OU.png` | [04-users-groups-and-ous/ou-overview.png](04-users-groups-and-ous/ou-overview.png) | Custom OU structure |
| `5.1-ou-itsupport.png` | [04-users-groups-and-ous/it-support.png](04-users-groups-and-ous/it-support.png) | IT Tech in IT Support |
| `5.2-ou-hospitalstaff.png` | [04-users-groups-and-ous/hospital-staff.png](04-users-groups-and-ous/hospital-staff.png) | Hospital Staff OU |
| `5.3-ou-contractors.png` | [04-users-groups-and-ous/contractors.png](04-users-groups-and-ous/contractors.png) | Contractors OU |
| `5.1-gpmc.png` | [05-group-policy/gpo-link.png](05-group-policy/gpo-link.png) | GPO associated with IT Support |
| `5.2-gpo-settings.png` | [05-group-policy/password-length.png](05-group-policy/password-length.png) | Minimum password length: 10 |
| `5.3-gpresult.png` | [05-group-policy/gpresult-local-user.png](05-group-policy/gpresult-local-user.png) | Local user policy report |
| `6.1-gpo-broken.png` | [05-group-policy/account-lockout-settings.png](05-group-policy/account-lockout-settings.png) | Threshold: 3 attempts; duration and reset: 30 minutes |
| `6.1 problem.png` | [06-troubleshooting/freerdp-transport-error.png](06-troubleshooting/freerdp-transport-error.png) | Separate vagrant/DC transport failure |
| `6.3-diagnosis-rdp-permissions.png` | [06-troubleshooting/remote-desktop-settings.png](06-troubleshooting/remote-desktop-settings.png) | Remote settings, not the user membership dialog |
| `6.4-diagnosis-account-status.png` | [06-troubleshooting/ittech-directory-entry.png](06-troubleshooting/ittech-directory-entry.png) | IT Tech entry, not account-status flags |
| `6.5-console-login-success.png` | [06-troubleshooting/ittech-console-login.png](06-troubleshooting/ittech-console-login.png) | Console session as CYBERLOOP\ittech |
