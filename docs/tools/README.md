# Tools Reference

This section provides an overview of each major tool used in the home lab, with notes on why it was chosen, key features leveraged, and links to relevant project write-ups.

---

## Table of Contents

- [Proxmox VE](#proxmox-ve)
- [pfSense](#pfsense)
- [Kali Linux](#kali-linux)
- [Metasploit Framework](#metasploit-framework)
- [Nmap](#nmap)
- [Nessus Essentials](#nessus-essentials)
- [Burp Suite Community Edition](#burp-suite-community-edition)
- [Security Onion](#security-onion)
- [Splunk](#splunk)
- [Wazuh](#wazuh)
- [Wireshark](#wireshark)
- [BloodHound](#bloodhound)
- [Hashcat](#hashcat)

---

## Proxmox VE

| | |
|-|---|
| **Type** | Hypervisor / Virtualization Platform |
| **Version** | 8.x |
| **Official Docs** | https://pve.proxmox.com/pve-docs/ |

Proxmox VE is an open-source type-1 hypervisor based on KVM and LXC. It was chosen for its:
- **No-cost licensing** for home lab use
- **Web-based GUI** for easy VM management
- **VLAN-aware bridging** for network segmentation
- **Snapshot and backup support** for safe lab experimentation

**Key features used:**
- VLAN-aware Linux bridge (`vmbr0`) for all inter-VM networking
- VM snapshots before/after each lab exercise
- ISO storage for all OS images

---

## pfSense

| | |
|-|---|
| **Type** | Firewall / Router |
| **Version** | 2.7.x |
| **Official Docs** | https://docs.netgate.com/pfsense/en/latest/ |

pfSense is an open-source firewall/router distribution based on FreeBSD. In this lab it provides:
- **Inter-VLAN routing** between all lab segments
- **Stateful firewall rules** to enforce zone isolation
- **NAT** for outbound internet access from the lab
- **VPN** (OpenVPN) for remote lab access

**Key features used:**
- Separate firewall rule sets per interface
- Firewall aliases for IP groups (e.g., `CORP_HOSTS`, `MONITORING_HOSTS`)
- Traffic shaping to simulate bandwidth constraints
- DNS resolver for local `lab.local` hostname resolution

---

## Kali Linux

| | |
|-|---|
| **Type** | Attacker OS / Penetration Testing Platform |
| **Version** | 2024.x |
| **Official Docs** | https://www.kali.org/docs/ |

Kali Linux is the industry-standard penetration testing distribution, pre-loaded with hundreds of security tools. It serves as the primary attack machine in the lab.

**Tools most used from Kali:**
- `nmap` — network discovery and port scanning
- `metasploit-framework` — exploitation
- `enum4linux` / `ldapsearch` — Active Directory enumeration
- `impacket` — AD attack suite (GetUserSPNs, secretsdump, etc.)
- `BloodHound` / `SharpHound` — AD attack path analysis
- `burpsuite` — web application testing
- `hashcat` / `john` — offline password cracking
- `wireshark` — traffic capture and analysis

---

## Metasploit Framework

| | |
|-|---|
| **Type** | Exploitation Framework |
| **Version** | 6.x |
| **Official Docs** | https://docs.metasploit.com/ |

Metasploit is the world's most widely used penetration testing framework. It provides:
- **Exploit modules** for hundreds of known CVEs
- **Auxiliary modules** for scanning and enumeration
- **Post-exploitation modules** for privilege escalation and persistence
- **Meterpreter** — advanced in-memory payload for post-exploitation

**Used in:**
- [Project 04 — Exploitation with Metasploit](../projects/04-exploitation-metasploit.md)
- [Project 08 — Incident Response Simulation](../projects/08-incident-response.md)

---

## Nmap

| | |
|-|---|
| **Type** | Network Scanner |
| **Version** | 7.x |
| **Official Docs** | https://nmap.org/docs.html |

Nmap is the most widely used open-source network scanner.

**Common scans used:**
```bash
# Host discovery
nmap -sn 10.20.20.0/24

# Full port scan with service version detection
nmap -sV -sC -p- 10.20.20.30

# OS detection
nmap -O 10.20.20.10

# Export to grepable format
nmap -oG scan_results.gnmap 10.20.20.0/24
```

**Used in:** [Project 03 — Network Scanning & Enumeration](../projects/03-network-scanning-enumeration.md)

---

## Nessus Essentials

| | |
|-|---|
| **Type** | Vulnerability Scanner |
| **Version** | 10.x (Essentials — free for home lab) |
| **Official Docs** | https://docs.tenable.com/nessus/ |

Nessus Essentials is the free version of Tenable's industry-leading vulnerability scanner, limited to 16 IP addresses — sufficient for this lab.

**Key capabilities:**
- Authenticated and unauthenticated vulnerability scanning
- CVE identification and CVSS scoring
- Compliance checks (CIS benchmarks)
- Plugin-based scan policies

**Used in:** [Project 03 — Network Scanning & Enumeration](../projects/03-network-scanning-enumeration.md)

---

## Burp Suite Community Edition

| | |
|-|---|
| **Type** | Web Application Security Testing |
| **Version** | 2024.x |
| **Official Docs** | https://portswigger.net/burp/documentation |

Burp Suite is the industry-standard tool for web application penetration testing.

**Key features used:**
- **Intercept Proxy** — intercept and modify HTTP/HTTPS requests
- **Repeater** — manually craft and replay requests
- **Intruder** — automated fuzzing and brute-force
- **Decoder** — encode/decode data (Base64, URL encoding, etc.)

**Used in:** [Project 05 — Web Application Testing](../projects/05-web-app-testing.md)

---

## Security Onion

| | |
|-|---|
| **Type** | Network Security Monitoring (NSM) / IDS Platform |
| **Version** | 2.4.x |
| **Official Docs** | https://docs.securityonion.net/ |

Security Onion is a free, open-source NSM platform that bundles Suricata (IDS), Zeek (network analysis), and a full analysis stack.

**Components leveraged:**
- **Suricata** — signature-based IDS generating alerts for known attack patterns
- **Zeek** — deep network traffic logging (DNS, HTTP, SSL, conn logs)
- **Security Onion Console (SOC)** — alert triage and investigation UI
- **Kibana** — log visualization dashboards

**Used in:** [Project 07 — Intrusion Detection with Security Onion](../projects/07-intrusion-detection.md)

---

## Splunk

| | |
|-|---|
| **Type** | SIEM / Log Management |
| **Version** | 9.x (Free trial / Developer license) |
| **Official Docs** | https://docs.splunk.com/ |

Splunk is a leading SIEM platform used in enterprise security operations centers (SOCs).

**Key capabilities used:**
- **Log ingestion** — Windows Event Logs, syslog, and custom sources via Splunk Universal Forwarder
- **SPL (Search Processing Language)** — querying and correlating log data
- **Dashboards** — building SOC-style dashboards for real-time visibility
- **Alerts** — threshold-based and scheduled alerts for suspicious activity
- **TAs (Technology Add-ons)** — pre-built field extractions for Windows, Linux, pfSense

**Example SPL queries documented in:** [Project 06 — SIEM & Log Analysis](../projects/06-siem-log-analysis.md)

---

## Wazuh

| | |
|-|---|
| **Type** | HIDS / EDR / Security Platform |
| **Version** | 4.x |
| **Official Docs** | https://documentation.wazuh.com/ |

Wazuh is a free, open-source security platform combining HIDS, EDR, log analysis, and compliance capabilities.

**Key capabilities used:**
- **File Integrity Monitoring (FIM)** — detecting unauthorized file changes
- **Rootkit detection**
- **Active Response** — automated threat response (e.g., blocking IPs)
- **MITRE ATT&CK mapping** — alerts mapped to ATT&CK techniques
- **Vulnerability detection** — CVE identification on monitored endpoints

---

## Wireshark

| | |
|-|---|
| **Type** | Packet Capture / Network Analysis |
| **Version** | 4.x |
| **Official Docs** | https://www.wireshark.org/docs/ |

Wireshark is the world's most popular network protocol analyzer.

**Common uses in the lab:**
- Capturing and analyzing exploit traffic
- Examining clear-text credentials in unencrypted protocols
- Analyzing malware C2 traffic patterns
- Verifying firewall rule effectiveness

**Common filters:**
```
# Filter by IP
ip.addr == 10.20.20.10

# Show only HTTP traffic
http

# Filter for DNS queries (not responses)
dns and dns.flags.response == 0
```

---

## BloodHound

| | |
|-|---|
| **Type** | Active Directory Attack Path Analysis |
| **Version** | 4.x (CE) |
| **Official Docs** | https://bloodhound.readthedocs.io/ |

BloodHound uses graph theory to reveal hidden relationships in Active Directory environments and identify attack paths to Domain Admin.

**Workflow:**
1. Run SharpHound collector on a domain-joined machine to gather AD data
2. Import JSON data into BloodHound
3. Query for shortest paths to Domain Admin
4. Use results to prioritize hardening efforts

**Used in:** [Project 02 — Active Directory Lab](../projects/02-active-directory-lab.md)

---

## Hashcat

| | |
|-|---|
| **Type** | Password Cracking |
| **Version** | 6.x |
| **Official Docs** | https://hashcat.net/wiki/ |

Hashcat is the world's fastest GPU-based password cracking tool.

**Common attack modes used:**
```bash
# Dictionary attack on NTLM hashes
hashcat -m 1000 -a 0 hashes.txt /usr/share/wordlists/rockyou.txt

# Kerberoast — crack Kerberos TGS tickets
hashcat -m 13100 -a 0 kerberoast_hashes.txt /usr/share/wordlists/rockyou.txt

# Rule-based attack (adds common substitutions/additions)
hashcat -m 1000 -a 0 -r /usr/share/hashcat/rules/best64.rule hashes.txt wordlist.txt
```

**Used in:** [Project 02 — Active Directory Lab](../projects/02-active-directory-lab.md)
