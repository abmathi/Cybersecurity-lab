# Project 01 — Environment Setup

**Skills:** Virtualization, Networking, VLAN Configuration, Firewall Policy

---

## Objective

Build the complete lab infrastructure from scratch: install the hypervisor, configure network segmentation, deploy pfSense, and verify that all VLANs are routed and isolated correctly before any security exercises begin.

---

## Steps Completed

### 1. Installed Proxmox VE 8.x

- Downloaded Proxmox VE 8.x ISO from the official site
- Created a bootable USB using `dd`
- Installed Proxmox on the lab host's NVMe SSD
- Post-install: disabled enterprise repository, enabled free repository, updated all packages

### 2. Configured VLAN-Aware Bridge

In the Proxmox web UI, created a Linux bridge `vmbr0` attached to the lab NIC with **VLAN-aware** mode enabled. This allows VMs to use specific VLAN tags without requiring separate physical interfaces per segment.

### 3. Deployed pfSense VM

- Created a VM with 2 vCPUs, 2 GB RAM, 20 GB disk
- Attached 5 virtual NICs, one per VLAN (10, 20, 30, 99 + WAN)
- Installed pfSense 2.7 and configured each interface:

| Interface | VLAN | IP |
|-----------|------|----|
| WAN | — | DHCP from home router |
| OPT1 (attack) | 10 | 10.10.10.1/24 |
| OPT2 (corporate) | 20 | 10.20.20.1/24 |
| OPT3 (dmz) | 30 | 10.30.30.1/24 |
| OPT4 (mgmt) | 99 | 10.0.0.1/24 |

### 4. Configured Firewall Rules

Defined firewall rules in pfSense to:
- Allow full traffic from VLAN 10 (attack) → VLAN 20 (corporate)
- Allow log forwarding from VLAN 20 → VLAN 30 (monitoring)
- Block VLAN 10 from reaching VLAN 30 and VLAN 99
- Allow VLAN 99 to reach everything (management access)
- Allow all VLANs outbound internet via NAT

### 5. Deployed Initial VMs

Deployed and configured network settings on:
- **Kali Linux** (VLAN 10, 10.10.10.10)
- **Windows Server 2019** (VLAN 20, 10.20.20.10)
- **Ubuntu Server** placeholder VMs for monitoring (VLAN 30)

### 6. Verified Connectivity

```
Kali → Windows DC:        PASS  (ping 10.20.20.10)
Kali → Security Onion:    FAIL  (blocked — expected)
DC → Splunk port 9997:    PASS  (log forwarder test)
Kali → Internet:          PASS  (curl ifconfig.me)
Management → All VMs:     PASS  (SSH/WinRM reachable)
```

---

## Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| pfSense interfaces not appearing after boot | Fixed by reseating virtual NIC order in Proxmox — pfSense interface assignment is boot-order sensitive |
| VLAN traffic not passing between VMs | Enabled VLAN-aware mode on `vmbr0` — initially missed this setting |
| Windows Server couldn't resolve `lab.local` | Configured pfSense DNS resolver to forward `lab.local` to the Windows DC (`10.20.20.10`) |

---

## Key Takeaways

- VLAN-aware bridging in Proxmox is essential for multi-tenant network segmentation on a single physical NIC
- pfSense interface assignment order matters and should be documented carefully
- Establishing and verifying firewall rules before starting offensive exercises ensures the lab boundary is maintained

---

## Next Steps

→ [Project 02 — Active Directory Lab](02-active-directory-lab.md)
