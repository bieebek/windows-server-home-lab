# Lab architecture

[Project overview](../README.md)

```mermaid
flowchart LR
    Host["Pop!_OS host<br/>Legion Go"]
    subgraph VBox["VirtualBox: cyberloop.local"]
        DC["domain-controller<br/>Windows Server 2019<br/>AD DS + DNS"]
        WS["win-workstation-1<br/>Windows 10"]
        WS -->|"Domain membership / DNS"| DC
    end
    Host -->|"localhost:23389 RDP"| DC
    Host -->|"localhost:43389 RDP"| WS
    Host -.->|"VirtualBox console"| WS
```

The RDP host ports above appear in the supplied captures. Vagrant also reports dynamically adjusted forwarded ports during startup, so check the active mappings when restarting the lab. This logical diagram does not assert guest IP addresses or an unverified VirtualBox network mode.

Both VMs were provisioned through Vagrant/Ansible. The notes record 1536 MB RAM and 2 vCPUs for each VM. The workstation uses the DC for domain DNS. The custom OUs are IT Support, Hospital Staff and Contractors.
