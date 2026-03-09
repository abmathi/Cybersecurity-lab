# Cybersecurity Home Lab

A fully documented cybersecurity home lab built to develop and demonstrate hands-on skills in network security, penetration testing, incident response, and security operations — designed to showcase practical experience to future employers.

---

## Table of Contents

- [Overview](#overview)
- [Lab Objectives](#lab-objectives)
- [Network Architecture](#network-architecture)
- [Technologies & Tools](#technologies--tools)
- [Documentation](#documentation)
- [Projects & Labs](#projects--labs)
- [Skills Demonstrated](#skills-demonstrated)

---

## Overview

This home lab simulates a small enterprise network environment with dedicated attack, defense, and monitoring segments. It provides a safe, isolated environment for practicing real-world cybersecurity scenarios including:

- Simulated attacks and penetration testing
- Threat detection and incident response
- Log aggregation and SIEM analysis
- Vulnerability assessment and remediation
- Active Directory management and exploitation

---

## Lab Objectives

| Objective | Description |
|-----------|-------------|
| **Hands-on Practice** | Apply cybersecurity concepts learned through coursework and certifications in a real environment |
| **Tool Proficiency** | Gain experience with industry-standard security tools (Kali Linux, Splunk, Suricata, pfSense, etc.) |
| **Attack Simulation** | Safely simulate offensive techniques to understand attacker TTPs (Tactics, Techniques, and Procedures) |
| **Defensive Operations** | Build and tune detection rules, analyze logs, and practice incident response |
| **Portfolio Building** | Document all lab activities to demonstrate skills to prospective employers |

---

## Network Architecture

```
                          ┌─────────────────────────────────┐
                          │         Home Network             │
                          │         192.168.1.0/24           │
                          └──────────────┬──────────────────┘
                                         │
                                  ┌──────┴──────┐
                                  │  Hypervisor │
                                  │  (Proxmox)  │
                                  └──────┬──────┘
                                         │
              ┌──────────────────────────┼──────────────────────────┐
              │                          │                           │
   ┌──────────┴──────────┐  ┌───────────┴───────────┐  ┌──────────┴──────────┐
   │   pfSense Firewall  │  │   pfSense Firewall    │  │  Management VLAN    │
   │   (WAN Interface)   │  │   (VLAN Routing)      │  │  10.0.0.0/24        │
   └──────────┬──────────┘  └───────────┬───────────┘  └─────────────────────┘
              │                          │
   ┌──────────┴──────────┐               │
   │                     │    ┌──────────┴──────────────────────────────┐
   │   Attack Segment    │    │                                          │
   │   VLAN 10           │    │         Internal Segments                │
   │   10.10.10.0/24     │    │                                          │
   │                     │    │  ┌─────────────┐   ┌─────────────────┐  │
   │  ┌───────────────┐  │    │  │ Corp VLAN20 │   │ DMZ  VLAN 30    │  │
   │  │  Kali Linux   │  │    │  │10.20.20.0/24│   │10.30.30.0/24    │  │
   │  │  (Attacker)   │  │    │  │             │   │                 │  │
   │  └───────────────┘  │    │  │ Win Server  │   │  Security Onion │  │
   │                     │    │  │ (AD/DC)     │   │  (IDS/NSM)      │  │
   └─────────────────────┘    │  │ Win 10 x2   │   │  Splunk SIEM    │  │
                              │  │ Metasploit. │   │                 │  │
                              │  │ Ubuntu Svr  │   └─────────────────┘  │
                              │  └─────────────┘                        │
                              └─────────────────────────────────────────┘
```

See [docs/architecture/network-topology.md](docs/architecture/network-topology.md) for full details.

---

## Technologies & Tools

### Infrastructure
| Category | Tool | Purpose |
|----------|------|---------|
| Hypervisor | [Proxmox VE](https://www.proxmox.com/) | Virtualization platform hosting all VMs |
| Firewall/Router | [pfSense](https://www.pfsense.org/) | Network segmentation, firewall rules, VPN |
| Networking | VLANs (802.1Q) | Isolating lab segments |

### Attack Platform
| Category | Tool | Purpose |
|----------|------|---------|
| Attacker OS | [Kali Linux](https://www.kali.org/) | Penetration testing and offensive toolset |
| Framework | [Metasploit](https://www.metasploit.com/) | Exploitation framework |
| Scanning | [Nmap](https://nmap.org/), [Nessus](https://www.tenable.com/products/nessus) | Network and vulnerability scanning |
| Web App Testing | [Burp Suite](https://portswigger.net/burp) | Web application security testing |
| Password Attacks | [Hashcat](https://hashcat.net/), [John the Ripper](https://www.openwall.com/john/) | Password cracking |

### Vulnerable Targets
| Category | Tool | Purpose |
|----------|------|---------|
| Windows AD | Windows Server 2019 + Windows 10 | Active Directory attack surface |
| Linux Target | [Metasploitable 3](https://github.com/rapid7/metasploitable3) | Intentionally vulnerable Linux VM |
| Web App | [DVWA](https://github.com/digininja/DVWA) | Deliberately vulnerable web application |
| Web App | [VulnHub machines](https://www.vulnhub.com/) | Additional CTF-style targets |

### Defense & Monitoring
| Category | Tool | Purpose |
|----------|------|---------|
| IDS/NSM | [Security Onion](https://securityonionsolutions.com/) | Network security monitoring, Suricata IDS |
| SIEM | [Splunk](https://www.splunk.com/) | Log aggregation, correlation, alerting |
| EDR/HIDS | [Wazuh](https://wazuh.com/) | Host-based intrusion detection |
| Packet Capture | [Wireshark](https://www.wireshark.org/) | Network traffic analysis |
| Threat Intel | [MITRE ATT&CK](https://attack.mitre.org/) | Mapping adversary TTPs |

---

## Documentation

| Section | Description |
|---------|-------------|
| [Architecture](docs/architecture/) | Network topology, VLAN design, and firewall rules |
| [Setup Guide](docs/setup/) | Step-by-step environment build instructions |
| [Tools](docs/tools/) | Configuration and usage notes for each tool |
| [Projects](docs/projects/) | Individual lab exercises and walkthroughs |

---

## Projects & Labs

| Project | Description | Skills |
|---------|-------------|--------|
| [01 — Environment Setup](docs/projects/01-environment-setup.md) | Provisioning the hypervisor, VMs, and network segments | Virtualization, networking |
| [02 — Active Directory Lab](docs/projects/02-active-directory-lab.md) | Deploy AD, simulate common AD attacks (Kerberoasting, Pass-the-Hash) | Windows, AD, offense/defense |
| [03 — Network Scanning & Enumeration](docs/projects/03-network-scanning-enumeration.md) | Reconnaissance using Nmap, enum4linux, and Nessus | Recon, vulnerability scanning |
| [04 — Exploitation with Metasploit](docs/projects/04-exploitation-metasploit.md) | Exploiting vulnerabilities on Metasploitable 3 | Exploitation, post-exploitation |
| [05 — Web Application Testing](docs/projects/05-web-app-testing.md) | Testing DVWA for OWASP Top 10 vulnerabilities | Web security, Burp Suite |
| [06 — SIEM & Log Analysis](docs/projects/06-siem-log-analysis.md) | Aggregating logs in Splunk and building detection rules | SIEM, log analysis, alerting |
| [07 — Intrusion Detection with Security Onion](docs/projects/07-intrusion-detection.md) | Tuning Suricata rules and analyzing IDS alerts | IDS/NSM, Suricata |
| [08 — Incident Response Simulation](docs/projects/08-incident-response.md) | End-to-end IR exercise: detect, contain, eradicate, recover | Incident response, forensics |

---

## Skills Demonstrated

- ✅ **Virtualization & Network Engineering** — Proxmox, pfSense, VLAN segmentation
- ✅ **Penetration Testing** — Reconnaissance, scanning, exploitation, post-exploitation
- ✅ **Active Directory Security** — AD deployment, enumeration, and attack techniques (Kerberoasting, Pass-the-Hash, BloodHound)
- ✅ **Web Application Security** — OWASP Top 10, Burp Suite, manual testing
- ✅ **Security Monitoring** — IDS rule tuning, NSM with Security Onion
- ✅ **SIEM Operations** — Log ingestion, search queries (SPL), dashboards, and correlation rules in Splunk
- ✅ **Incident Response** — PICERL methodology (Preparation, Identification, Containment, Eradication, Recovery, Lessons Learned)
- ✅ **Threat Intelligence** — MITRE ATT&CK framework mapping
- ✅ **Documentation** — Detailed write-ups for all lab activities

---

> **Note:** This lab is entirely isolated and built for educational and portfolio purposes. All offensive techniques are performed only within the controlled lab environment against intentionally vulnerable machines.

