# SIEM Deployment (Wazuh)

[← Back to main README](../README.md)

## Overview

Deployed [Wazuh](https://wazuh.com/) as a centralized SIEM/XDR platform to monitor the existing Active Directory homelab. This phase extends the AD + osTicket environment with real-time log collection, detection rules, and a dashboard for investigating security events across both a Windows Server domain controller and a Linux server.

**Why Wazuh:** Chosen over a raw ELK stack build because it ships as a purpose-built SIEM with agents, detection rules, and a dashboard already integrated — letting the project focus on deployment, agent management, and detection engineering rather than assembling infrastructure from scratch.

## Architecture

| Host | Role | OS | Network |
|---|---|---|---|
| DC01 | Active Directory domain controller, monitored endpoint | Windows Server 2022 | 192.168.1.10 (VMnet1) / NAT |
| CLIENT02 | Domain-joined workstation | Windows 11 | 192.168.1.20 (VMnet1) / NAT |
| TICKET01 | osTicket host, monitored endpoint | Ubuntu Server 26.04 LTS | 192.168.1.40 (VMnet1) / NAT |
| WAZUH01 | Wazuh manager, indexer, and dashboard (all-in-one) | Ubuntu Server 26.04 LTS | 192.168.1.30 (VMnet1) / NAT |

All VMs sit on a shared host-only network (VMnet1, 192.168.1.0/24) for internal communication, with a second NAT-connected adapter on each host providing outbound internet access for package installs and updates — keeping the internal network isolated while still letting each VM reach the internet independently.

## Build Steps

1. **Provisioned WAZUH01** — Ubuntu Server 26.04 LTS, 40GB disk (LVM), dual network adapters (VMnet1 static + NAT DHCP), matching the network pattern already established on DC01 and CLIENT02.
2. **Installed the Wazuh all-in-one stack** (manager, indexer, dashboard) via the official quickstart script:
```bash
   curl -O https://packages.wazuh.com/4.9/wazuh-install.sh
   sudo bash ./wazuh-install.sh -a -i
```
   (`-i` bypasses the recommended minimum hardware check, acceptable for a homelab-scale deployment.)
3. **Deployed the Windows agent to DC01** using the dashboard's agent-deployment wizard, which generates a ready-to-run PowerShell install command targeting the manager at 192.168.1.30.
4. **Deployed the Linux agent to TICKET01** the same way, using the generated `.deb`-based install command.
5. **Verified both agents connected** via `agent_control -l` on the manager and confirmed "Active" status in the dashboard's Endpoints view.
6. **Generated a real test event** — deliberately triggered failed logon attempts on DC01 and confirmed detection end-to-end in the dashboard's Threat Hunting module.

## Troubleshooting Log

Documenting the real issues hit during deployment, since working through them was most of the actual learning:

### 1. Disk-full error during dashboard install
The Wazuh dashboard component (~935MB installed) failed mid-install with a disk-full error, even though the VM had a 40GB virtual disk assigned. Ubuntu's guided LVM installer had only allocated half the disk (~19GB) to the root logical volume, leaving the rest as unallocated free space in the volume group.

**Fix:**
```bash
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
```
This extended the logical volume to claim the full disk, after which the install completed cleanly.

### 2. Fake default route breaking internet access
After manually configuring a static IP on the VMnet1 adapter during the OS install, the resulting netplan config included a default route via the VMnet1 gateway (`192.168.1.1`) — an address with no actual path to the internet, since VMnet1 is a host-only virtual network. This route had a lower metric than the real NAT adapter's default route, so Ubuntu preferred the dead-end route and all outbound traffic failed.

**Fix:** Removed the erroneous `routes:` block from the VMnet1 interface's netplan config, leaving only the NAT adapter's default route active:
```yaml
network:
  ethernets:
    ens33:            # VMnet1 — internal only, no default route
      addresses: [192.168.1.30/24]
      nameservers:
        addresses: [192.168.1.10]
    ens37:             # NAT — internet access
      dhcp4: true
  version: 2
```

### 3. TICKET01 agent silently failing to register
The Wazuh agent installed and started successfully on TICKET01, but never appeared in the manager's agent list. Investigation showed TICKET01 had only ever been configured with a NAT adapter — it had no interface on the 192.168.1.0/24 network at all, so it had no route to reach the manager at 192.168.1.30, despite the agent's own config file correctly pointing at that address.

**Fix:** Added a second network adapter to TICKET01 (matching the dual-adapter pattern used elsewhere), configured it with a static IP on VMnet1 (192.168.1.40), and the agent registered automatically on the next connection attempt — no reinstall needed.

## Detection Proof

To confirm the pipeline works end-to-end rather than just "looking connected," a real detection scenario was run:

1. Triggered multiple failed logon attempts on DC01 against a nonexistent user account.
2. Within seconds, the events appeared in Wazuh's Threat Hunting module, correctly parsed from the Windows Security Event Log (Event ID 4625) and classified under rule `60122` — *"Logon Failure - Unknown user"* — at rule level 5.
3. Confirmed the event count, timestamps, and source agent (DC01) all matched the test exactly.

*(Screenshot: failed logon events in Threat Hunting, filtered by agent DC01)*

*(Screenshot: expanded raw event detail showing parsed Windows Event ID 4625 data)*

## Skills Demonstrated

- SIEM deployment and administration (Wazuh manager, indexer, dashboard)
- Cross-platform agent deployment (Windows and Linux)
- Log-based detection and event triage
- Linux disk management (LVM volume extension, filesystem resizing)
- Linux network configuration and troubleshooting (netplan, routing tables, default route metrics)
- Root-cause diagnosis using systematic verification at each layer (network config → routing table → raw connectivity → DNS → application-level registration) rather than guessing

## Next Steps

- Add Microsoft Entra ID hybrid identity sync (DC01 → Entra Connect) as the next homelab phase.
- Feed Entra ID sign-in and audit logs into this same Wazuh instance.
- Layer in Conditional Access policies and a risk-based detection scenario tying hybrid identity and SIEM monitoring together.
