# Lab documentation

[Project overview](../README.md)

Read the walkthroughs in order, or go directly to the troubleshooting case study.

1. [Environment setup](01-environment-setup.md)
2. [Domain controller and provisioning fixes](02-domain-controller.md)
3. [Workstation domain join](03-workstation-domain-join.md)
4. [Users, groups and OUs](04-users-groups-and-ous.md)
5. [Group Policy configuration and verification](05-group-policy.md)
6. [RDP access investigation](06-rdp-troubleshooting.md)

Supporting material: [screenshot index](../screenshots/README.md), [original 23-page screenshot collection](ad-lab-screenshot-collection.pdf), [architecture](../diagrams/README.md), and [historical notes](archive/README.md).

## Evidence conventions

The walkthroughs distinguish visible screenshot evidence, results recorded only in the build notes, and future work. Screenshots have descriptive filenames and captions based on their actual content. Their original names and new paths are mapped in the screenshot index.

The PDF is the supplied screenshot collection, formerly `1.pdf`, renamed without changing its contents. It is not a separate written report. The PNG images remain unchanged, including historical lab defaults visible in some terminal captures; passwords have been removed from the archived Markdown command examples.

## Remaining work

- [ ] Capture the DC and workstation Ansible play recaps, including ignored-task counts.
- [ ] Run and capture `dcdiag`; investigate any reported failures.
- [ ] Repair the DNS forwarder task and test external resolution.
- [ ] Verify domain password/lockout policy and capture resultant settings for `ittech`.
- [ ] Resolve the RDP issue and capture a successful remote sign-in as `ittech`.
- [ ] Add the actual modified Vagrantfile and playbooks after removing credentials.

The original notes also mention PowerShell automation, osTicket and pfSense as future projects. They are not presented as completed work in this repository.
