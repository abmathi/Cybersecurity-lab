# Setup Guide

This guide walks through the complete process of building the cybersecurity home lab from scratch, from installing the hypervisor through to deploying all virtual machines and monitoring tools.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Install Proxmox VE](#2-install-proxmox-ve)
3. [Configure Networking & VLANs](#3-configure-networking--vlans)
4. [Deploy pfSense Firewall](#4-deploy-pfsense-firewall)
5. [Deploy Attack Segment VMs](#5-deploy-attack-segment-vms)
6. [Deploy Corporate Segment VMs](#6-deploy-corporate-segment-vms)
7. [Deploy Monitoring Stack](#7-deploy-monitoring-stack)
8. [Configure Log Forwarding](#8-configure-log-forwarding)
9. [Verify Lab Connectivity](#9-verify-lab-connectivity)

---

## 1. Prerequisites

Before starting, ensure you have:

- A physical machine meeting the [hardware requirements](../architecture/network-topology.md#physical-hardware)
- ISO images downloaded:
  - [Proxmox VE 8.x](https://www.proxmox.com/en/downloads)
  - [pfSense 2.7](https://www.pfsense.org/download/)
  - [Kali Linux 2024.x](https://www.kali.org/get-kali/)
  - [Windows Server 2019 Evaluation](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2019)
  - [Windows 10 Enterprise Evaluation](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-10-enterprise)
  - [Security Onion 2.4](https://github.com/Security-Onion-Solutions/securityonion/releases)
  - [Ubuntu Server 22.04 LTS](https://ubuntu.com/download/server) (x2 — for Splunk and Wazuh)
- A USB drive (≥ 8 GB) for the Proxmox installer
- Basic familiarity with Linux command line

---

## 2. Install Proxmox VE

### 2.1 Create Bootable USB

```bash
# On Linux
dd if=proxmox-ve_8.x.iso of=/dev/sdX bs=4M status=progress
```

For Windows, use [Rufus](https://rufus.ie/) or [balenaEtcher](https://etcher.balena.io/).

### 2.2 Boot and Install

1. Boot from USB and follow the Proxmox installer wizard
2. Select the target disk for installation (the NVMe SSD)
3. Configure:
   - Country and timezone
   - Root password (strong, unique)
   - Management IP: `10.0.0.2/24`, gateway `10.0.0.1`
   - Hostname: `proxmox.lab.local`
4. Complete installation and reboot

### 2.3 Post-Install Configuration

Access the web UI at `https://10.0.0.2:8006`.

```bash
# Disable enterprise repo (no subscription)
nano /etc/apt/sources.list.d/pve-enterprise.list
# Comment out the enterprise line, then add:
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  >> /etc/apt/sources.list

# Update system
apt-get update && apt-get upgrade -y
```

---

## 3. Configure Networking & VLANs

### 3.1 Create Linux Bridge with VLAN Support

In the Proxmox web UI:

1. Go to **Node → Network → Create → Linux Bridge**
2. Name: `vmbr0`
3. Bridge port: your lab NIC (e.g., `enp2s0`)
4. Enable **VLAN aware**
5. Apply configuration

### 3.2 VLAN Reference

| VLAN ID | Name | Subnet |
|---------|------|--------|
| 10 | attack | 10.10.10.0/24 |
| 20 | corporate | 10.20.20.0/24 |
| 30 | dmz | 10.30.30.0/24 |
| 99 | management | 10.0.0.0/24 |

---

## 4. Deploy pfSense Firewall

### 4.1 Create the VM

1. In Proxmox: **Create VM**
   - Name: `pfsense`
   - OS: Other (upload the pfSense ISO)
   - Disk: 20 GB
   - CPU: 2 cores
   - RAM: 2 GB
2. Add **multiple network interfaces** — one per network segment:
   - `vmbr0` (no VLAN tag) → WAN interface (uplink to home router)
   - `vmbr0` tag `10` (Attack VLAN)
   - `vmbr0` tag `20` (Corporate VLAN)
   - `vmbr0` tag `30` (DMZ VLAN)
   - `vmbr0` tag `99` (Management VLAN)

### 4.2 Install pfSense

1. Boot the VM from the ISO
2. Follow installation wizard (accept defaults for ZFS or UFS)
3. On first boot, assign interfaces when prompted:
   - WAN → `vtnet0`
   - LAN interfaces → remaining adapters

### 4.3 Configure Interfaces and Firewall Rules

Access the pfSense web UI at `http://10.0.0.1` after assigning the management interface.

Configure each interface with the IP addresses from the [network topology](../architecture/network-topology.md#ip-address-plan).

Add firewall rules as described in the [Firewall Rules Summary](../architecture/network-topology.md#firewall-rules-summary).

---

## 5. Deploy Attack Segment VMs

### 5.1 Kali Linux

1. **Create VM** in Proxmox:
   - Name: `kali`, 2 CPU cores, 4 GB RAM, 60 GB disk
   - Network: `vmbr0`, VLAN tag `10`
2. Install Kali from ISO (choose full install with all tools)
3. Post-install network config:
   ```bash
   # /etc/network/interfaces
   auto eth0
   iface eth0 inet static
     address 10.10.10.10
     netmask 255.255.255.0
     gateway 10.10.10.1
   ```

---

## 6. Deploy Corporate Segment VMs

### 6.1 Windows Server 2019 (Active Directory)

1. **Create VM**: 2 CPU cores, 4 GB RAM, 60 GB disk, VLAN 20
2. Install Windows Server 2019 (Desktop Experience)
3. Set static IP: `10.20.20.10/24`, gateway `10.20.20.1`
4. Promote to Domain Controller:
   ```powershell
   Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
   Install-ADDSForest -DomainName "lab.local" -DomainNetbiosName "LAB" -InstallDns
   ```

### 6.2 Windows 10 Clients

1. Create two Windows 10 VMs (2 CPU, 2 GB RAM, 40 GB disk, VLAN 20)
2. Static IPs: `10.20.20.20` and `10.20.20.21`
3. Join to `lab.local` domain:
   ```powershell
   Add-Computer -DomainName "lab.local" -Credential (Get-Credential) -Restart
   ```

### 6.3 Metasploitable 3

```bash
# Clone and build with Vagrant + VirtualBox, then export OVA, import to Proxmox
git clone https://github.com/rapid7/metasploitable3
cd metasploitable3
vagrant up ub1404
```

After importing: set IP `10.20.20.30/24`, gateway `10.20.20.1`.

### 6.4 DVWA

```bash
# Deploy via Docker on an Ubuntu 20.04 VM
docker run --rm -it -p 80:80 vulnerables/web-dvwa
```

Access at `http://10.20.20.40`.

---

## 7. Deploy Monitoring Stack

### 7.1 Security Onion

1. Create VM: 4 CPU cores, 8 GB RAM, 100 GB disk, VLAN 30
2. Add a second NIC on VLAN 20 (for traffic monitoring — this is the tap/span interface)
3. Install Security Onion following the [official setup wizard](https://docs.securityonion.net/en/2.4/installation.html)
4. Set static IP: `10.30.30.10/24`
5. During setup, designate the VLAN 20 NIC as the **sniffing interface**

### 7.2 Splunk

```bash
# On Ubuntu 22.04 VM (10.30.30.20)
wget -O splunk.deb 'https://download.splunk.com/products/splunk/releases/9.x/linux/splunk-9.x-linux-amd64.deb'
dpkg -i splunk.deb
/opt/splunk/bin/splunk start --accept-license
/opt/splunk/bin/splunk enable boot-start
```

Access at `http://10.30.30.20:8000`.

### 7.3 Wazuh

```bash
# On Ubuntu 22.04 VM (10.30.30.30) — Single-node deployment
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh
bash wazuh-install.sh -a
```

Access the Wazuh dashboard at `https://10.30.30.30`.

---

## 8. Configure Log Forwarding

### 8.1 Windows → Splunk (via Splunk Universal Forwarder)

On each Windows VM:

```powershell
# Download and install Splunk Universal Forwarder
Start-Process msiexec.exe -Wait -ArgumentList `
  '/i splunkforwarder.msi SPLUNKUSERNAME=admin SPLUNKPASSWORD=<password> ' +
  'RECEIVING_INDEXER="10.30.30.20:9997" WINEVENTLOG_SEC_ENABLE=1 ' +
  'WINEVENTLOG_SYS_ENABLE=1 WINEVENTLOG_APP_ENABLE=1 AGREETOLICENSE=Yes /quiet'
```

### 8.2 Linux → Splunk (via rsyslog)

```bash
# On each Linux VM
echo "*.* @10.30.30.20:514" >> /etc/rsyslog.conf
systemctl restart rsyslog
```

### 8.3 Wazuh Agents (Windows & Linux)

```powershell
# Windows (PowerShell)
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.x.msi -OutFile wazuh-agent.msi
msiexec.exe /i wazuh-agent.msi WAZUH_MANAGER="10.30.30.30" /q
```

```bash
# Linux
curl -so wazuh-agent.deb https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.x_amd64.deb
WAZUH_MANAGER="10.30.30.30" dpkg -i wazuh-agent.deb
systemctl start wazuh-agent
```

---

## 9. Verify Lab Connectivity

Run these checks to confirm the lab is working end-to-end:

```bash
# From Kali (attack segment), verify access to corporate segment
ping 10.20.20.10   # Windows DC — should succeed
ping 10.30.30.10   # Security Onion — should FAIL (blocked by firewall)

# Confirm Splunk is receiving logs
# Browse to http://10.30.30.20:8000 → Search → index=* | stats count by host

# Confirm Security Onion is capturing traffic
# Browse to https://10.30.30.10 → Dashboards → check for events

# Run a basic Nmap scan from Kali to confirm visibility
nmap -sV 10.20.20.0/24
# Verify the scan appears as alerts in Security Onion and Splunk
```

---

## Troubleshooting

| Issue | Resolution |
|-------|-----------|
| VMs cannot reach gateway | Check VLAN tag on VM network interface in Proxmox |
| pfSense not routing between VLANs | Verify firewall rules allow the traffic; check interface assignments |
| Security Onion not seeing traffic | Confirm span/tap NIC is set as the sniffing interface and not assigned an IP |
| Splunk not receiving logs | Check Universal Forwarder status; verify port 9997 is open on Splunk host |
| Wazuh agent not connecting | Confirm agent pointing to correct manager IP; check firewall allows port 1514 |

---

## Next Steps

With the lab running, proceed to the [Projects](../projects/) to start performing and documenting security exercises.
