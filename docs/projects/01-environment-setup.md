# Project 01 — Attack Machine Setup

**Skills:** Virtualization, VirtualBox, Kali Linux, Network Verification

---

## Objective

Set up the attack machine: install VirtualBox on the Intel Mac, deploy a Kali Linux VM, and confirm the VM can reach the rest of the lab network. This provides the offensive platform used in all subsequent attack-simulation exercises.

---

## Environment

| Machine | Role | OS |
|---------|------|----|
| Attack Mac (Intel Core i7) | Attack host | macOS |
| Kali Linux VM | Attacker | Kali Linux (latest) |

**Network:** flat home LAN — `192.168.0.0/24`, gateway `192.168.0.1`

---

## Steps Completed

### 1. Installed VirtualBox on the Attack Mac

- Downloaded VirtualBox from [virtualbox.org](https://www.virtualbox.org/)
- Ran the macOS installer package and completed the installation
- Verified VirtualBox launched and the Extension Pack was installed for USB 3.0 / RDP support

### 2. Downloaded and Deployed the Kali Linux VM

- Downloaded the pre-built **Kali Linux VirtualBox image** (64-bit) from [kali.org/get-kali](https://www.kali.org/get-kali/#kali-virtual-machines)
- Imported the `.ova` file into VirtualBox:
  - **File → Import Appliance** → selected the Kali `.ova`
  - Accepted default settings (2 CPU, 2 GB RAM) then adjusted to 2 CPU / 4 GB RAM for lab use
- Set the network adapter to **Bridged Adapter** (attached to the Mac's active NIC) so the VM receives an IP on the home LAN

### 3. Started the Kali VM and Verified Network

Booted the VM, logged in with default credentials (`kali` / `kali`), then confirmed the VM obtained a LAN IP:

```bash
ip a
# eth0 showed an IP in the 192.168.0.0/24 range
```

### 4. Confirmed Reach to Windows Systems

Before any Windows VMs existed, tested internet connectivity to confirm bridged networking was working:

```bash
ping -c 3 192.168.0.1        # gateway
curl -s ifconfig.me           # public IP — confirms internet routing
```

Once the Windows VMs were up (later phases), connectivity was re-verified:

```bash
ping -c 3 192.168.0.10        # DC01
ping -c 3 192.168.0.20        # WS01
```

---

## Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| VirtualBox network adapter showing as NAT after import | Changed adapter type to **Bridged Adapter** in VM settings → Network |
| Kali VM only showing a 192.168.56.x IP (VirtualBox host-only network) | Detached the host-only adapter; set adapter 1 to Bridged Adapter on the Mac's Wi-Fi or Ethernet interface |

---

## Key Takeaways

- Using the pre-built Kali VirtualBox OVA saves significant setup time compared to a manual install
- Bridged Adapter mode places the VM directly on the home LAN, making it reachable to all other lab systems without any NAT or port-forwarding configuration
- Verifying network connectivity at each phase avoids troubleshooting surprises later

---

## Next Steps

→ [Project 02 — Active Directory Domain Setup](02-active-directory-lab.md)
