# Network Topology

This document describes the full network architecture of the home lab, including all systems, IP addressing, and traffic flows.

---

## Overview

The lab uses a **flat home LAN (192.168.0.0/24)** — no VLANs or dedicated hypervisor. Each physical machine (Attack Mac, SOC Mac, and a dedicated Windows host) runs VMs or acts as the host directly. All systems communicate over the flat home network.

```
Home LAN — 192.168.0.0/24
Default Gateway: 192.168.0.1

┌───────────────────────────────────────────────────────────────────────┐
│                         192.168.0.0/24                                │
│                                                                       │
│  ┌─────────────────────┐   ┌─────────────────────┐                   │
│  │  Attack Mac         │   │  DC01 (Windows Svr) │                   │
│  │  Intel Core i7      │   │  corp.lab DC         │                   │
│  │  └── Kali Linux VM  │   │  Static IP           │                   │
│  │      (Attacker)     │   │  AD DS + DNS         │                   │
│  └─────────────────────┘   │  Sysmon + WEF (WEC)  │                   │
│                            │  Elastic Agent       │                   │
│                            └─────────────────────┘                   │
│                                       ↑ WEF                           │
│                            ┌─────────────────────┐                   │
│                            │  WS01 (Windows 10)  │                   │
│                            │  Joined: corp.lab    │                   │
│                            │  Sysmon + WEF client │                   │
│                            └─────────────────────┘                   │
│                                                                       │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │  SOC Mac (Apple Silicon M1 Pro)                                │   │
│  │  └── Ubuntu Server VM (ARM64 — headless / CLI-only)           │   │
│  │      └── Elastic Stack (self-hosted, free Basic licence)      │   │
│  │          ├── Elasticsearch  (port 9200)                       │   │
│  │          ├── Kibana         (port 5601)                       │   │
│  │          └── Fleet Server   (port 8220)                       │   │
│  └────────────────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────────────┘
```

---

## Systems

| Hostname | OS | Role | IP | Notes |
|----------|----|------|----|-------|
| `Attack Mac` | macOS (Intel i7) | Attack host | 192.168.0.x (DHCP) | Runs Kali Linux VM |
| `kali` | Kali Linux (VM) | Attacker | 192.168.0.x (DHCP) | Impacket, CrackMapExec, Hashcat |
| `DC01` | Windows Server | Domain Controller | Static (192.168.0.x) | corp.lab, AD DS, DNS, WEF WEC, Sysmon, Elastic Agent |
| `WS01` | Windows 10 / 11 | Workstation | DHCP (corp.lab) | Domain-joined, Sysmon, WEF client |
| `SOC Mac` | macOS (Apple Silicon M1 Pro) | SOC / SIEM host | 192.168.0.x | Runs Ubuntu Server VM |
| `ubuntu-siem` | Ubuntu Server (ARM64 VM) | Elastic Stack host | Static (192.168.0.x) | Elasticsearch, Kibana, Fleet Server |

---

## Data Flows

### Log Collection Pipeline

```
WS01 (Sysmon + Windows Event Log)
  │
  │  Windows Event Forwarding (WEF / WinRM)
  ▼
DC01 (WEC Collector — Forwarded Events log)
  │
  │  Elastic Agent (Windows + Sysmon integrations)
  ▼
Fleet Server (ubuntu-siem)
  │
  ▼
Elasticsearch → Kibana (Elastic Security)
```

### Attack Traffic Flow

```
Kali Linux VM
  │  Kerberoasting, password spray, lateral movement
  ▼
DC01 / WS01
  │  Events generated (4769, 4625, 4624, 7045, etc.)
  ▼
Elastic Security (alerts + investigation timeline)
```

---

## Domain Configuration

| Setting | Value |
|---------|-------|
| Domain Name | `corp.lab` |
| NetBIOS Name | `CORP` |
| Domain Controller | `DC01` |
| DNS Server | DC01 static IP |
| Kerberos | Configured and functional (no time skew) |
| WEF Subscription | Push (DC01 as WEC; WS01 as source) |

---

## Elastic Stack Configuration

| Component | Port | Notes |
|-----------|------|-------|
| Elasticsearch | 9200 | Single-node cluster, Basic licence |
| Kibana | 5601 | Elastic Security app enabled |
| Fleet Server | 8220 | Manages Elastic Agent on DC01 |

---

## Physical Hardware

| Machine | Hardware |
|---------|---------|
| Attack Mac | Intel Core i7, macOS |
| SOC Mac | Apple Silicon M1 Pro, macOS |
| DC01 / WS01 | Dedicated Windows machines on home LAN |

---

## References

- [Elastic Fleet Server Documentation](https://www.elastic.co/guide/en/fleet/current/fleet-server.html)
- [Sysmon Documentation](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)
- [Windows Event Forwarding (WEF) Documentation](https://learn.microsoft.com/en-us/windows/security/threat-protection/use-windows-event-forwarding-to-assist-in-intrusion-detection)
- [Elastic Agent Documentation](https://www.elastic.co/guide/en/fleet/current/elastic-agent-installation.html)

