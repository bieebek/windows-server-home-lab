# Historical build notes

[Current walkthroughs](../README.md) · [Project overview](../../README.md)

These drafts preserve the original troubleshooting narrative and planning history. Their status labels and screenshot descriptions are historical; use the six current walkthroughs for the reconciled account of the lab.

- [Original build notes](original-build-notes.md): detailed provisioning issues, interim workarounds and planned next steps.
- [Original final-writeup notes](original-final-notes.md): later narrative used to assemble the current walkthroughs.

Publication edits remove literal passwords from Markdown, replace unfilled screenshot markers with a link to the current evidence index, and add this context. The source files remain unchanged in the local working folder.

## Corrections applied in the current walkthroughs

- The captured lockout threshold is three attempts, while the notes describe a one-attempt test.
- The files originally labeled playbook-success captures show RDP desktops, not recaps.
- The command-line USERDOMAIN and gpresult captures show a local account context.
- The supplied remote-settings and ADUC captures do not display RDP membership or account-status flags.
- OU-linked password settings are not proof of effective domain-user password policy.
- Console sign-in succeeded, but the RDP root cause and final fix remain unconfirmed.
- No dcdiag result, DHCP scope deployment, ESXi deployment or Server 2022 build is evidenced by these supplied files.
