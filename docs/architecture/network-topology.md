# Network Topology

This document describes the full network architecture of the home lab, including all VLANs, IP addressing, firewall zones, and VM placement.

---

## Overview

The lab uses a single physical host running **Proxmox VE** as the hypervisor. A **pfSense** VM acts as the central router and firewall, providing VLAN-based network segmentation to isolate each security zone.

```
Physical Host (Proxmox VE)
└── Virtual Switch (vmbr0)
    └── pfSense (Router/Firewall)
        ├── WAN — uplink to home router (192.168.1.0/24)
        ├── VLAN 10 — Attack Segment    (10.10.10.0/24)
        ├── VLAN 20 — Corporate Segment (10.20.20.0/24)
        ├── VLAN 30 — DMZ / Monitoring  (10.30.30.0/24)
        └── VLAN 99 — Management        (10.0.0.0/24)
```

---

## IP Address Plan

| VLAN | Segment Name | Subnet | Gateway | Purpose |
|------|-------------|--------|---------|---------|
| 10 | Attack | 10.10.10.0/24 | 10.10.10.1 | Offensive tools (Kali Linux) |
| 20 | Corporate | 10.20.20.0/24 | 10.20.20.1 | Windows AD domain, targets |
| 30 | DMZ / Monitoring | 10.30.30.0/24 | 10.30.30.1 | Security Onion, Splunk, SIEM |
| 99 | Management | 10.0.0.0/24 | 10.0.0.1 | Proxmox, out-of-band access |

---

## Virtual Machines

| VM Name | OS | VLAN | IP Address | Role |
|---------|----|------|------------|------|
| pfsense | pfSense 2.7 | All (router) | 10.x.x.1 per VLAN | Firewall / Router |
| kali | Kali Linux 2024 | VLAN 10 | 10.10.10.10 | Attacker machine |
| winserver2019 | Windows Server 2019 | VLAN 20 | 10.20.20.10 | Active Directory / Domain Controller |
| win10-client1 | Windows 10 | VLAN 20 | 10.20.20.20 | Domain-joined workstation |
| win10-client2 | Windows 10 | VLAN 20 | 10.20.20.21 | Domain-joined workstation |
| metasploitable3 | Ubuntu 14.04 | VLAN 20 | 10.20.20.30 | Intentionally vulnerable Linux target |
| dvwa | Ubuntu 20.04 | VLAN 20 | 10.20.20.40 | Deliberately vulnerable web app |
| security-onion | Security Onion 2.4 | VLAN 30 | 10.30.30.10 | IDS / NSM (tap on VLAN 20) |
| splunk | Ubuntu 22.04 | VLAN 30 | 10.30.30.20 | SIEM log aggregation |
| wazuh | Ubuntu 22.04 | VLAN 30 | 10.30.30.30 | EDR / HIDS manager |

---

## Firewall Rules Summary

### Attack Segment → Corporate Segment (VLAN 10 → VLAN 20)
| Action | Protocol | Source | Destination | Notes |
|--------|----------|--------|-------------|-------|
| Allow | Any | 10.10.10.0/24 | 10.20.20.0/24 | Simulated attack traffic |
| Block | Any | 10.10.10.0/24 | 10.0.0.0/24 | Prevent management access from attacker |

### Corporate Segment → DMZ (VLAN 20 → VLAN 30)
| Action | Protocol | Source | Destination | Notes |
|--------|----------|--------|-------------|-------|
| Allow | Syslog/TCP 514 | 10.20.20.0/24 | 10.30.30.20 | Log forwarding to Splunk |
| Allow | Wazuh/TCP 1514 | 10.20.20.0/24 | 10.30.30.30 | Agent communication to Wazuh |
| Block | Any | 10.20.20.0/24 | 10.30.30.0/24 | No direct corp → monitoring access |

### DMZ → Everywhere
| Action | Protocol | Source | Destination | Notes |
|--------|----------|--------|-------------|-------|
| Block | Any | 10.30.30.0/24 | 10.10.10.0/24 | Monitoring never reaches attack segment |
| Block | Any | 10.30.30.0/24 | 10.20.20.0/24 | Monitoring is read-only |

### Management (VLAN 99)
| Action | Protocol | Source | Destination | Notes |
|--------|----------|--------|-------------|-------|
| Allow | Any | 10.0.0.0/24 | Any | Full management access |
| Block | Any | Any (other) | 10.0.0.0/24 | Only management VLAN can reach management net |

---

## Monitoring / Span Port

Security Onion monitors traffic on **VLAN 20** via a SPAN (mirrored) port configured on the virtual switch in Proxmox. This ensures all inter-VM traffic in the Corporate segment is captured for IDS analysis without altering the network path.

---

## Physical Hardware

| Component | Specification |
|-----------|--------------|
| CPU | Intel Core i7-12700 (12 cores / 20 threads) |
| RAM | 64 GB DDR4 |
| Storage | 1 TB NVMe SSD (VMs) + 2 TB HDD (snapshots/archives) |
| NIC | Intel dual-port gigabit NIC (one for WAN, one for lab VLANs) |
| Hypervisor | Proxmox VE 8.x |

---

## References

- [pfSense Documentation](https://docs.netgate.com/pfsense/en/latest/)
- [Proxmox VE Documentation](https://pve.proxmox.com/pve-docs/)
- [Security Onion Documentation](https://docs.securityonion.net/)
