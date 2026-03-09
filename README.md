# Cybersecurity Home Lab — SOC / Blue Team Detection Lab

A fully documented cybersecurity home lab focused on **SOC / Blue Team detection and investigation**, not just attacks. Built to demonstrate hands-on experience with Elastic SIEM, Windows Event Forwarding, Sysmon, Active Directory attack detection, and real-time alerting — designed to showcase practical experience to future employers.

---

## Table of Contents

- [Overview](#overview)
- [Lab Objectives](#lab-objectives)
- [Network Architecture](#network-architecture)
- [Lab Status](#lab-status)
- [Technologies & Tools](#technologies--tools)
- [Documentation](#documentation)
- [Projects & Labs](#projects--labs)
- [Skills Demonstrated](#skills-demonstrated)

---

## Overview

This home lab simulates a small enterprise environment with a dedicated attack platform, a Windows Active Directory domain, and a self-hosted Elastic SIEM stack. It provides a safe, isolated environment for practising real-world SOC scenarios including:

- Active Directory attacks (Kerberoasting, password spraying, lateral movement)
- Real-time log ingestion via Windows Event Forwarding and Elastic Agent
- Threat detection with Sysmon + Elastic Security detection rules
- Custom alert creation and investigation workflows
- SOC analyst skills mapped to real attacker TTPs (MITRE ATT&CK)

---

## Lab Objectives

| Objective | Description |
|-----------|-------------|
| **SOC / Blue Team Focus** | Build detection capabilities, not just attack tooling — tune rules, triage alerts, investigate incidents |
| **Elastic SIEM Proficiency** | Ingest Windows event logs, Sysmon telemetry, and forwarded events; build custom detection rules and dashboards |
| **Active Directory Security** | Deploy a realistic AD domain (corp.lab) and detect common attacks (Kerberoasting, Pass-the-Hash, spray) |
| **Windows Telemetry** | Implement Sysmon and Windows Event Forwarding for rich endpoint visibility |
| **Portfolio Building** | Document every lab exercise with commands, results, and detection evidence |

---

## Network Architecture

```
Home LAN — 192.168.0.0/24
Default Gateway: 192.168.0.1
─────────────────────────────────────────────────────
  ┌───────────────────────┐
  │   Attack Mac (Intel)  │
  │   Host OS: macOS      │
  │   └── Kali Linux VM   │  ← Kerberoasting, CME,
  │       (Attacker)      │     password spraying
  └───────────┬───────────┘
              │  192.168.0.0/24 (flat LAN)
  ┌───────────┴───────────┐
  │  DC01 — Windows Server│
  │  Domain: corp.lab     │  ← AD DS, DNS, Sysmon,
  │  Static IP            │     WEF (WEC receiver),
  │                       │     Elastic Agent
  └───────────┬───────────┘
              │
  ┌───────────┴───────────┐
  │  WS01 — Windows Client│
  │  Joined: corp.lab     │  ← Sysmon, WEF client,
  │                       │     domain user logins,
  │                       │     lateral movement target
  └───────────────────────┘

  ┌────────────────────────────────────────────────────┐
  │  SOC Mac (Apple Silicon M1 Pro)                    │
  │  └── Ubuntu Server VM (ARM64, CLI-only / headless) │
  │      └── Elastic Stack (self-hosted)               │
  │          ├── Elasticsearch                         │
  │          ├── Kibana + Elastic Security             │
  │          └── Fleet Server                          │
  └────────────────────────────────────────────────────┘
```

See [docs/architecture/network-topology.md](docs/architecture/network-topology.md) for full details.

---

## Lab Status

### ✅ Completed

| Component | Status | Notes |
|-----------|--------|-------|
| Active Directory domain (`corp.lab`) | ✅ Working | DNS, Kerberos, SMB all functional |
| Time sync | ✅ Fixed | No Kerberos skew issues |
| Sysmon on DC01 | ✅ Installed & logging | Capturing process creation, network, registry events |
| Sysmon on WS01 | ✅ Installed & logging | Same configuration as DC01 |
| Windows Event Forwarding (WEF) | ✅ Functional | WS01 → DC01; events visible in Forwarded Events log |
| WEF manual test (`eventcreate`) | ✅ Confirmed | Test event forwarded successfully |
| Elastic Stack on Ubuntu ARM64 | ✅ Running | Elasticsearch + Kibana + Fleet Server |
| Fleet Server | ✅ Healthy in Kibana | Visible in Fleet management UI |
| Windows agent policy | ✅ Created | Scoped to Windows endpoints |
| Windows + Sysmon integrations | ✅ Added | Both integrations configured in policy |
| Elastic Agent on DC01 | ✅ Installed | Agent enrolled and reporting |
| Logs hitting Kibana | ✅ Real-time | Windows Event Logs + Sysmon events visible |

### 🔜 Next Steps

| Step | Goal |
|------|------|
| [Validate Kerberoasting visibility](docs/projects/11-kerberoasting-detection.md) | Simulate Kerberoast from Kali; confirm Event ID 4769 in Kibana |
| [Build custom detection rules](docs/projects/12-custom-detection-rules.md) | Detect excessive 4625 failures and suspicious service ticket requests |
| [Add WS01 & simulate lateral movement](docs/projects/13-lateral-movement-lab.md) | Lateral movement from DC01 to WS01; detect with Elastic Security |
| [Tune Windows integration](docs/projects/14-powershell-logging-tuning.md) | Enable PowerShell logging (Script Block, Module, Transcription) |

---

## Technologies & Tools

### Infrastructure
| Category | Tool | Purpose |
|----------|------|---------|
| Hypervisor | macOS Virtualization / UTM | Hosting VMs on Intel Mac and Apple Silicon Mac |
| Domain Controller | Windows Server (DC01) | Active Directory domain `corp.lab`, DNS, WEF receiver |
| Workstation | Windows Client (WS01) | Domain-joined target for lateral movement simulation |

### Attack Platform
| Category | Tool | Purpose |
|----------|------|---------|
| Attacker OS | [Kali Linux](https://www.kali.org/) | Penetration testing and offensive toolset (VM on Intel Mac) |
| AD Attacks | [Impacket](https://github.com/fortra/impacket) | Kerberoasting (`GetUserSPNs.py`), AS-REP roasting |
| AD Attacks | [CrackMapExec (CME)](https://github.com/byt3bl33d3r/CrackMapExec) | Password spraying, SMB enumeration |
| Password Cracking | [Hashcat](https://hashcat.net/) | Offline cracking of Kerberos TGS tickets |

### Telemetry & Log Collection
| Category | Tool | Purpose |
|----------|------|---------|
| Endpoint Telemetry | [Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) | Rich process, network, and registry event logging on Windows |
| Log Forwarding | Windows Event Forwarding (WEF) | Forward events from WS01 to DC01 collector |
| SIEM Agent | [Elastic Agent](https://www.elastic.co/elastic-agent) | Ships Windows events + Sysmon logs to Elasticsearch |

### Defense & Monitoring
| Category | Tool | Purpose |
|----------|------|---------|
| SIEM | [Elastic Security](https://www.elastic.co/security) | Log aggregation, detection rules, alerting, investigations |
| Log Storage | [Elasticsearch](https://www.elastic.co/elasticsearch) | Indexed log storage (self-hosted, free Basic licence) |
| UI / Dashboards | [Kibana](https://www.elastic.co/kibana) | Visualisations, Discover, Security app |
| Agent Management | [Fleet Server](https://www.elastic.co/guide/en/fleet/current/fleet-server.html) | Centralised Elastic Agent policy and enrollment |
| Threat Intel | [MITRE ATT&CK](https://attack.mitre.org/) | Mapping adversary TTPs to detection rules |
| Packet Capture | [Wireshark](https://www.wireshark.org/) | Network traffic analysis |

---

## Documentation

| Section | Description |
|---------|-------------|
| [Architecture](docs/architecture/) | Network topology and system design |
| [Setup Guide](docs/setup/) | Step-by-step environment build instructions |
| [Tools](docs/tools/) | Configuration and usage notes for each tool |
| [Projects](docs/projects/) | Individual lab exercises and walkthroughs |

---

## Projects & Labs

### SOC / Blue Team Series (Current Lab)

| Project | Description | Status | Skills |
|---------|-------------|--------|--------|
| [09 — Elastic SIEM Setup](docs/projects/09-elastic-siem-setup.md) | Deploy Elastic Stack on Ubuntu ARM64; enroll Elastic Agent on DC01 | ✅ Complete | Elastic, Fleet, Ubuntu |
| [10 — WEF & Sysmon Setup](docs/projects/10-wef-sysmon-setup.md) | Install Sysmon on DC01/WS01; configure WEF pipeline | ✅ Complete | Sysmon, WEF, WinRM, GPO |
| [11 — Kerberoasting Detection](docs/projects/11-kerberoasting-detection.md) | Simulate Kerberoast from Kali; detect via Event ID 4769 in Kibana | 🔜 Next | Kerberos, Impacket, KQL |
| [12 — Custom Detection Rules](docs/projects/12-custom-detection-rules.md) | Build Elastic detection rules for 4625 brute-force and SPN scanning | 🔜 Next | Elastic Security, EQL, KQL |
| [13 — Lateral Movement Lab](docs/projects/13-lateral-movement-lab.md) | Simulate lateral movement to WS01; detect with Sysmon + Elastic | 🔜 Next | SMB, WMI, Pass-the-Hash |
| [14 — PowerShell Logging Tuning](docs/projects/14-powershell-logging-tuning.md) | Enable Script Block, Module, and Transcription logging; tune Elastic integration | 🔜 Next | PowerShell, GPO, Elastic |

### Foundation Series

| Project | Description | Skills |
|---------|-------------|--------|
| [01 — Environment Setup](docs/projects/01-environment-setup.md) | Provisioning VMs and network | Virtualization, networking |
| [02 — Active Directory Lab](docs/projects/02-active-directory-lab.md) | Deploy AD, simulate common AD attacks | Windows, AD, offense/defense |
| [03 — Network Scanning & Enumeration](docs/projects/03-network-scanning-enumeration.md) | Reconnaissance with Nmap, enum4linux, Nessus | Recon, vulnerability scanning |
| [04 — Exploitation with Metasploit](docs/projects/04-exploitation-metasploit.md) | Exploiting vulnerabilities on Metasploitable 3 | Exploitation, post-exploitation |
| [05 — Web Application Testing](docs/projects/05-web-app-testing.md) | Testing DVWA for OWASP Top 10 | Web security, Burp Suite |
| [06 — SIEM & Log Analysis](docs/projects/06-siem-log-analysis.md) | Log aggregation and detection rules | SIEM, log analysis, alerting |
| [07 — Intrusion Detection with Security Onion](docs/projects/07-intrusion-detection.md) | Tuning Suricata rules and analysing IDS alerts | IDS/NSM, Suricata |
| [08 — Incident Response Simulation](docs/projects/08-incident-response.md) | End-to-end IR exercise: detect, contain, eradicate, recover | Incident response, forensics |

---

## Skills Demonstrated

- ✅ **Active Directory Security** — Domain deployment, AD attack simulation (Kerberoasting, Pass-the-Hash, password spraying), and detection
- ✅ **Elastic SIEM** — Self-hosted deployment, Fleet management, agent enrollment, Windows/Sysmon integrations, KQL/EQL queries, custom detection rules
- ✅ **Windows Telemetry** — Sysmon configuration and tuning; Windows Event Forwarding (WEF) subscriber/collector setup
- ✅ **SOC Operations** — Alert triage, detection rule creation, investigation workflows, event correlation
- ✅ **Threat Intelligence** — MITRE ATT&CK TTP mapping (T1558 Kerberoasting, T1110 Brute Force, T1021 Lateral Movement)
- ✅ **Penetration Testing** — Reconnaissance, scanning, exploitation, post-exploitation
- ✅ **Web Application Security** — OWASP Top 10, Burp Suite, manual testing
- ✅ **Security Monitoring** — IDS rule tuning, NSM with Security Onion
- ✅ **Incident Response** — PICERL methodology (Preparation, Identification, Containment, Eradication, Recovery, Lessons Learned)
- ✅ **Documentation** — Detailed write-ups for all lab activities

---

> **Note:** This lab is entirely self-contained and built for educational and portfolio purposes. All offensive techniques are performed only within the controlled lab environment against intentionally vulnerable machines owned by the lab operator.

