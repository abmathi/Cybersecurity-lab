# Project 07 — SOC Infrastructure Deployment

**Skills:** Ubuntu Server, Linux Administration, VirtualBox, ARM64, Headless Server, Infrastructure Setup

---

## Objective

Set up the dedicated SOC server: install VirtualBox on the Apple Silicon Mac (M1 Pro), deploy an Ubuntu Server ARM64 VM in headless / CLI-only mode, and verify the server is reachable from the rest of the lab. This VM will host the full Elastic Stack in the next phase.

---

## Environment

| Machine | Role | OS |
|---------|------|----|
| SOC Mac (Apple Silicon M1 Pro) | SOC host | macOS (Apple Silicon) |
| Ubuntu Server VM | Elastic Stack host | Ubuntu Server 22.04 LTS (ARM64) |

**Network:** flat home LAN — `192.168.0.0/24`, gateway `192.168.0.1`

The Ubuntu VM is intended to run headless (no GUI) — all management is done via SSH from the Mac terminal.

---

## Steps Completed

### 1. Installed VirtualBox on the SOC Mac

- Downloaded VirtualBox 7.x for macOS / Apple Silicon from [virtualbox.org](https://www.virtualbox.org/)
- Ran the macOS installer package
- Verified VirtualBox launched successfully on the M1 Pro Mac

> **Note:** VirtualBox 7.x added support for Apple Silicon (ARM64). Earlier versions did not support ARM Macs. Ensure VirtualBox 7.0 or later is installed.

### 2. Downloaded Ubuntu Server ARM64 ISO

- Downloaded **Ubuntu Server 22.04 LTS (ARM64)** from [ubuntu.com/download/server/arm](https://ubuntu.com/download/server/arm)
- Confirmed the `.iso` filename ends in `arm64` (not `amd64`)

### 3. Created the Ubuntu Server VM in VirtualBox

- Created a new VM in VirtualBox:
  - **Name:** `ubuntu-siem`
  - **Type:** Linux | **Version:** Ubuntu (64-bit)
  - **RAM:** 8 GB (minimum for Elastic Stack; 16 GB recommended if available)
  - **CPU:** 4 cores
  - **Disk:** 100 GB (dynamically allocated)
  - **Network adapter 1:** Bridged Adapter (attached to the Mac's active NIC — Wi-Fi or Ethernet)

- Mounted the Ubuntu Server ARM64 ISO as a virtual optical drive and started the VM

### 4. Installed Ubuntu Server

During the Ubuntu Server installer:

- Selected **English** language and keyboard layout
- Network configuration: left on **DHCP** initially (static IP set post-install)
- Storage: used the full disk (no LVM required for a single-purpose SIEM VM)
- Set hostname: **`ubuntu-siem`**
- Created a local admin user (e.g., `socadmin`)
- Selected **Install OpenSSH server** ✅ (essential for headless management)
- Did not install any snaps
- Completed the installation and rebooted

### 5. Assigned a Static IP

After the first boot, logged in at the VM console and assigned a static IP using Netplan:

```bash
# Find the network interface name
ip a
# Note the interface name (commonly 'enp0s3' or 'ens3')
```

Edited the Netplan configuration:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 192.168.0.50/24
      routes:
        - to: default
          via: 192.168.0.1
      nameservers:
        addresses:
          - 192.168.0.1
          - 8.8.8.8
```

```bash
sudo netplan apply
ip a   # confirm 192.168.0.50 is shown
```

### 6. Updated the System

```bash
sudo apt update && sudo apt upgrade -y
```

### 7. Verified SSH Access from the SOC Mac

From the Mac terminal:

```bash
ssh socadmin@192.168.0.50
```

**Result:** SSH session established. All subsequent management of the Ubuntu VM is done over SSH — no VirtualBox console required.

### 8. Verified Connectivity to the Rest of the Lab

```bash
ping -c 3 192.168.0.10    # DC01
ping -c 3 192.168.0.1     # gateway
```

---

## Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| VirtualBox showing only `amd64` VM types | Ensured VirtualBox 7.x was installed (supports ARM64 guests on Apple Silicon) |
| Ubuntu installer network showed no interface | Confirmed bridged adapter was attached in VirtualBox settings before booting the ISO |
| Static IP not applying after `netplan apply` | YAML indentation error; fixed spacing — Netplan is whitespace-sensitive |
| VM clock out of sync with lab | Installed and enabled `chrony` for NTP time synchronisation |

---

## Key Takeaways

- VirtualBox 7.x on Apple Silicon supports ARM64 guests natively — no Rosetta translation needed
- Installing OpenSSH during the Ubuntu setup wizard makes the server immediately accessible headlessly without touching the VirtualBox console again
- Allocating at least 8 GB RAM and 100 GB disk upfront avoids having to resize the VM later when Elasticsearch indices grow

---

## Next Steps

→ [Project 08 — Elastic Stack Installation](08-elastic-stack-setup.md)
