# Kyle's Home Lab (Actively Updating)

## Overview

This project demonstrates my hands-on experience building and expanding an enterprise-style IT environment from scratch inside VMware Workstation — starting with a Windows Server Active Directory domain and growing into a full identity, service desk, and security monitoring stack. Each phase below builds directly on the last, on the same set of VMs.

## Video Walkthrough

[Watch the full walkthrough on YouTube](https://www.youtube.com/)

## Environment

| Host | Role | OS |
|---|---|---|
| DC01 | Active Directory domain controller | Windows Server 2022 |
| CLIENT02 | Domain-joined workstation | Windows 11 |
| TICKET01 | osTicket host | Ubuntu Server |
| WAZUH01 | SIEM (Wazuh manager, indexer, dashboard) | Ubuntu Server |

VMware Workstation, with a host-only internal network (VMnet1) for VM-to-VM traffic and a separate NAT adapter on each VM for internet access.

## Phases

### Phase 1: Active Directory — Domain Setup, OUs, GPOs
Built the foundational Windows Server Active Directory environment: promoted a domain controller, created Organizational Units and security groups, managed users, and built and troubleshot Group Policy Objects.

**Skills demonstrated:**
- Installed Active Directory Domain Services (AD DS) and promoted the server to a Domain Controller
- Created Organizational Units (OUs), security groups, and user accounts
- Assigned users to groups and managed them via Active Directory Users and Computers
- Created and troubleshot Group Policy Objects (GPOs), including security filtering and Group Policy Client service issues

📄 [Full write-up: RBAC Structure](Documentation/RBAC-Structure.md)

### Phase 2: osTicket Integration
Deployed osTicket as a help desk ticketing system and integrated it with Active Directory via LDAP, so staff log in with their domain credentials. Ran a full ticket lifecycle end-to-end.

**Skills demonstrated:**
- Deployed osTicket in Docker on Ubuntu Server
- Configured LDAP Authentication and Lookup to bind osTicket to Active Directory
- Ran a complete ticket lifecycle — creation, assignment, triage, resolution, and closure

### Phase 3: SIEM Deployment (Wazuh)
Deployed Wazuh as a centralized SIEM to monitor the Active Directory and osTicket environment in real time, with agents on both the Windows domain controller and the Linux ticketing server, and a proven detection scenario.

**Skills demonstrated:**
- SIEM deployment and administration (Wazuh manager, indexer, dashboard)
- Cross-platform agent deployment (Windows and Linux)
- Log-based detection and event triage
- Linux disk management and network troubleshooting (LVM, netplan, routing)

📄 [Full write-up: SIEM Deployment (Wazuh)](Documentation/SIEM-Wazuh.md)

### Phase 4: Hybrid Identity (Microsoft Entra ID) — *planned*
Next up: connecting the on-premises Active Directory domain to Microsoft Entra ID via hybrid sync, layering in Conditional Access policies, and feeding Entra sign-in/audit logs into the existing Wazuh SIEM for a full identity + detection scenario.

## Screenshots

See the `Screenshots/` folder in this repository.
