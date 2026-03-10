# Tools Reference

This section provides an overview of each major tool used in the home lab, with notes on why it was chosen, key features leveraged, and links to relevant project write-ups.

---

## Table of Contents

### Infrastructure
- [VirtualBox](#virtualbox)

### SOC / Blue Team Tools
- [Elastic Stack (Elasticsearch + Kibana)](#elastic-stack-elasticsearch--kibana)
- [Elastic Agent & Fleet Server](#elastic-agent--fleet-server)
- [Sysmon](#sysmon)
- [Windows Event Forwarding (WEF)](#windows-event-forwarding-wef)

### Attack Tools
- [Kali Linux](#kali-linux)
- [Impacket](#impacket)
- [CrackMapExec (CME)](#crackmapexec-cme)
- [Hashcat](#hashcat)
- [Nmap](#nmap)
- [Wireshark](#wireshark)

---

## VirtualBox

| | |
|-|---|
| **Type** | Hypervisor / Virtualisation |
| **Version** | 7.x |
| **Official Docs** | https://www.virtualbox.org/wiki/Documentation |

VirtualBox is the hypervisor used on both lab Macs to host all VMs. Version 7.x added native support for Apple Silicon (ARM64) guests, making it usable on both the Intel Mac (attack machine) and the Apple Silicon M1 Pro Mac (SOC machine).

**VMs hosted in VirtualBox:**

| Host Machine | VM | Guest OS |
|-------------|-----|----------|
| Attack Mac (Intel) | Kali Linux | Kali Linux (x86_64) |
| SOC Mac (M1 Pro) | ubuntu-siem | Ubuntu Server 22.04 (ARM64) |

**Key VirtualBox settings used:**
- **Bridged Adapter** — places VMs directly on the home LAN (192.168.0.0/24) so they can communicate with each other and with physical machines without NAT
- **Dynamically allocated disk** — only uses actual disk space, not the full allocated size

**Used in:**
- [Project 01 — Attack Machine Setup](../projects/01-environment-setup.md)
- [Project 07 — SOC Infrastructure Deployment](../projects/07-soc-infrastructure.md)

---

## Elastic Stack (Elasticsearch + Kibana)

| | |
|-|---|
| **Type** | SIEM / Log Management / Security Analytics |
| **Version** | 8.x (free Basic licence) |
| **Official Docs** | https://www.elastic.co/guide/index.html |

Elastic Stack is a self-hosted, enterprise-grade SIEM platform used as the primary detection and investigation tool in this lab. The free Basic licence includes the full Elastic Security app with detection rules, alerts, and investigation timelines.

**Key capabilities used:**
- **Log ingestion** — Windows Event Logs, Sysmon events, Forwarded Events via Elastic Agent
- **KQL (Kibana Query Language)** — querying and investigating log data
- **EQL (Event Query Language)** — sequence-based correlation for multi-step attack detection
- **Elastic Security** — detection rules engine (pre-built MITRE ATT&CK rules + custom rules), alert management, investigation timelines
- **Dashboards** — real-time visibility into endpoint and network telemetry
- **Fleet** — centralised agent policy management

**Architecture in this lab:**
- Single-node Elasticsearch on Ubuntu Server (ARM64, IP: 192.168.0.50)
- Kibana + Fleet Server co-hosted on same Ubuntu VM
- Elastic Agent deployed on DC01 (Windows)
- Security app: `http://192.168.0.50:5601/app/security`

**Configuration (`elasticsearch.yml`):**
```yaml
cluster.name: soc-lab
node.name: soc-node-1
network.host: 0.0.0.0
discovery.type: single-node
```

**Key KQL queries:**
```kql
# All Security events from DC01
event.module: "windows" and host.name: "DC01"

# Failed login events (Event ID 4625)
event.code: "4625"

# Kerberos service ticket requests (Event ID 4769)
event.code: "4769"

# Sysmon process creation events
event.code: "1" and event.module: "sysmon"

# Suspicious PowerShell execution
event.code: "4104" and winlog.channel: "Microsoft-Windows-PowerShell/Operational"
```

**Used in:**
- [Project 08 — Elastic Stack Installation](../projects/08-elastic-stack-setup.md)
- [Project 09 — Elastic SIEM Setup](../projects/09-elastic-siem-setup.md)
- [Project 11 — Kerberoasting Detection](../projects/11-kerberoasting-detection.md)
- [Project 12 — Custom Detection Rules](../projects/12-custom-detection-rules.md)

---

## Elastic Agent & Fleet Server

| | |
|-|---|
| **Type** | Endpoint Agent / Centralised Policy Management |
| **Version** | 8.x |
| **Official Docs** | https://www.elastic.co/guide/en/fleet/current/elastic-agent-installation.html |

Elastic Agent is a single unified agent replacing Beats (Filebeat, Winlogbeat, etc.). Fleet Server provides centralised management of agent policies and integrations.

**Key integrations configured:**
- **Windows** — Security, System, Application event log ingestion
- **Sysmon** — Microsoft-Windows-Sysmon/Operational log ingestion

**Enrollment command (Windows):**
```powershell
.\elastic-agent.exe install `
  --url=https://<fleet-server-ip>:8220 `
  --enrollment-token=<token> `
  --insecure
```

**Agent health check:**
```powershell
# On the enrolled endpoint
Get-Service "Elastic Agent"
# In Kibana: Management → Fleet → Agents → verify "Healthy"
```

**Used in:** [Project 09 — Elastic SIEM Setup](../projects/09-elastic-siem-setup.md)

---

## Sysmon

| | |
|-|---|
| **Type** | Windows Endpoint Telemetry / HIDS |
| **Version** | 15.x |
| **Official Docs** | https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon |

Sysmon (System Monitor) is a Windows service and device driver that monitors and logs system activity to the Windows Event Log. It provides significantly richer telemetry than native Windows event logging.

**Key event IDs generated:**
| Event ID | Description | Detection Use |
|----------|-------------|---------------|
| 1 | Process Created | Detect malicious process launches |
| 3 | Network Connection | Detect C2, lateral movement |
| 7 | Image Loaded | Detect DLL injection |
| 10 | Process Access | Detect credential dumping (lsass) |
| 11 | FileCreate | Detect dropper/implant files |
| 13 | RegistryEvent | Detect persistence mechanisms |
| 22 | DNS Query | Detect C2 domain lookups |

**Installation:**
```powershell
.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```

**Recommended config:** [SwiftOnSecurity sysmon-config](https://github.com/SwiftOnSecurity/sysmon-config) (balanced coverage vs noise)

**Used in:**
- [Project 05 — Endpoint Telemetry with Sysmon](../projects/05-sysmon-setup.md)
- [Project 10 — WEF & Sysmon Setup](../projects/10-wef-sysmon-setup.md)
- [Project 11 — Kerberoasting Detection](../projects/11-kerberoasting-detection.md)
- [Project 13 — Lateral Movement Lab](../projects/13-lateral-movement-lab.md)

---

## Windows Event Forwarding (WEF)

| | |
|-|---|
| **Type** | Windows Log Collection / Forwarding |
| **Version** | Built into Windows Server / Windows 10+ |
| **Official Docs** | https://learn.microsoft.com/en-us/windows/security/threat-protection/use-windows-event-forwarding-to-assist-in-intrusion-detection |

WEF allows Windows endpoints (sources) to push their event logs to a central collector (WEC). In this lab, WS01 forwards all events to DC01, enabling a single Elastic Agent on DC01 to collect logs from both machines.

**Architecture:**
```
WS01 (WEF Source/Client)
  └─ WinRM → DC01 (WEF Collector / WEC)
               └─ Forwarded Events log
                   └─ Elastic Agent → Kibana
```

**Key configuration steps:**
1. Enable WinRM on collector: `winrm quickconfig`
2. Configure WEC service: `wecutil qc`
3. Create subscription (source-initiated) on collector
4. Push GPO to workstations pointing to collector

**Verify WEF is working:**
```powershell
# On DC01 — check active subscriptions
wecutil es

# On WS01 — verify WinRM is running
Get-Service WinRM
winrm enumerate winrm/config/listener

# Generate a test event on WS01
eventcreate /T INFORMATION /ID 100 /L APPLICATION /D "WEF Test"
# Verify it appears in DC01 Event Viewer → Forwarded Events
```

**Used in:**
- [Project 06 — Centralized Logging with WEF](../projects/06-wef-logging.md)
- [Project 10 — WEF & Sysmon Setup](../projects/10-wef-sysmon-setup.md)

---

## Kali Linux

| | |
|-|---|
| **Type** | Attacker OS / Penetration Testing Platform |
| **Version** | 2024.x |
| **Official Docs** | https://www.kali.org/docs/ |

Kali Linux is the industry-standard penetration testing distribution, pre-loaded with hundreds of security tools. It serves as the primary attack machine in the lab, run as a VM on the Intel Mac.

**Tools most used from Kali:**
- `impacket-GetUserSPNs` — Kerberoasting
- `crackmapexec` — SMB enumeration, password spraying
- `hashcat` — offline TGS ticket cracking
- `bloodhound-python` — AD attack path analysis
- `nmap` — network discovery and port scanning
- `responder` — NTLMv2 hash capture

**Used in:**
- [Project 01 — Attack Machine Setup](../projects/01-environment-setup.md)
- [Project 11 — Kerberoasting Detection](../projects/11-kerberoasting-detection.md)
- [Project 13 — Lateral Movement Lab](../projects/13-lateral-movement-lab.md)

---

## Impacket

| | |
|-|---|
| **Type** | AD Attack / Python Network Library |
| **Version** | Latest (from Kali or pip) |
| **Official Docs** | https://github.com/fortra/impacket |

Impacket is a Python library providing tools for interacting with Windows network protocols (SMB, Kerberos, LDAP, MSRPC).

**Key tools used:**
```bash
# Kerberoasting — request TGS tickets for SPNs
impacket-GetUserSPNs corp.lab/jsmith:'Password123!' -dc-ip 192.168.0.10 -request -outputfile kerberoast.hashes

# AS-REP Roasting — attack accounts without pre-auth
impacket-GetNPUsers corp.lab/ -dc-ip 192.168.0.10 -usersfile users.txt -no-pass -format hashcat

# Dump secrets (requires admin)
impacket-secretsdump corp.lab/Administrator:'<password>'@192.168.0.10

# Remote execution via SMB
impacket-psexec corp.lab/Administrator:'<password>'@192.168.0.20
```

**Used in:** [Project 11 — Kerberoasting Detection](../projects/11-kerberoasting-detection.md)

---

## CrackMapExec (CME)

| | |
|-|---|
| **Type** | AD Attack / SMB / LDAP Tool |
| **Version** | Latest (from Kali) |
| **Official Docs** | https://github.com/byt3bl33d3r/CrackMapExec |

CrackMapExec (CME) is a post-exploitation / AD pentesting tool for assessing large Active Directory networks.

**Key uses:**
```bash
# SMB host discovery with credentials
crackmapexec smb 192.168.0.0/24 -u jsmith -p 'Password123!' -d corp.lab

# Password spraying (one password, many users)
crackmapexec smb 192.168.0.10 -u users.txt -p 'Winter2024!' -d corp.lab --continue-on-success

# Enumerate shares
crackmapexec smb 192.168.0.10 -u jsmith -p 'Password123!' --shares

# Enumerate logged-on users
crackmapexec smb 192.168.0.10 -u jsmith -p 'Password123!' --loggedon-users
```

**Used in:** [Project 13 — Lateral Movement Lab](../projects/13-lateral-movement-lab.md)

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

# Kerberoast — crack Kerberos TGS tickets (RC4)
hashcat -m 13100 -a 0 kerberoast_hashes.txt /usr/share/wordlists/rockyou.txt

# Kerberoast — crack Kerberos TGS tickets (AES256)
hashcat -m 19700 -a 0 kerberoast_hashes.txt /usr/share/wordlists/rockyou.txt

# Rule-based attack (adds common substitutions/additions)
hashcat -m 1000 -a 0 -r /usr/share/hashcat/rules/best64.rule hashes.txt wordlist.txt
```

**Used in:** [Project 11 — Kerberoasting Detection](../projects/11-kerberoasting-detection.md)

---

## Nmap

| | |
|-|---|
| **Type** | Network Scanner |
| **Version** | 7.x |
| **Official Docs** | https://nmap.org/docs.html |

Nmap is the most widely used open-source network scanner. It is included in Kali Linux and used for host discovery and service enumeration across the lab network.

**Common scans used:**
```bash
# Host discovery on home LAN
nmap -sn 192.168.0.0/24

# Full port scan with service version detection
nmap -sV -sC -p- 192.168.0.10

# OS detection
nmap -O 192.168.0.10

# Scan specific AD service ports on DC01
nmap -sV -p 88,135,139,389,445,464,636 192.168.0.10
```

**Used in:** [Project 11 — Kerberoasting Detection](../projects/11-kerberoasting-detection.md)

---

## Wireshark

| | |
|-|---|
| **Type** | Packet Capture / Network Analysis |
| **Version** | 4.x |
| **Official Docs** | https://www.wireshark.org/docs/ |

Wireshark is the world's most popular network protocol analyser. It is used in the lab to inspect attack traffic and verify that expected network events are occurring during simulations.

**Common uses:**
- Inspecting Kerberos traffic during Kerberoasting exercises
- Verifying NTLMv2 hash capture during Responder simulations
- Analysing lateral movement (SMB, WMI, RDP) traffic

**Common filters:**
```
# Filter by IP
ip.addr == 192.168.0.10

# Kerberos traffic
kerberos

# SMB traffic
smb or smb2

# Show only DNS queries
dns and dns.flags.response == 0
```
