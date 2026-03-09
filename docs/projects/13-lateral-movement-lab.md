# Project 13 — Lateral Movement Lab

**Status:** 🔜 Next Step  
**Skills:** Lateral Movement, Pass-the-Hash, PsExec, WMI, SMB, Impacket, CrackMapExec, Sysmon, Elastic Security, MITRE ATT&CK T1021

---

## Overview

This project simulates lateral movement from DC01 to WS01 using credentials obtained during the Kerberoasting lab. Multiple lateral movement techniques are tested — SMB/PsExec, WMI, and Pass-the-Hash — with detection validated in Elastic Security at each step.

### Goals

- Simulate lateral movement using Impacket tools (psexec, wmiexec, atexec)
- Simulate Pass-the-Hash (PtH) using NTLM hashes
- Observe detection evidence: Event IDs 4624 (Logon Type 3), 4648, 7045; Sysmon Event 1 and 3
- Validate alerts fire in Elastic Security
- Document the investigation timeline for a lateral movement incident

---

## MITRE ATT&CK Reference

| Technique | ID | Description |
|-----------|-----|-------------|
| Remote Services: SMB/Windows Admin Shares | T1021.002 | Use SMB to execute commands on remote systems |
| Windows Management Instrumentation | T1047 | Use WMI for remote code execution |
| Pass the Hash | T1550.002 | Authenticate with an NTLM hash without knowing the plaintext password |
| Service Execution | T1569.002 | Use PsExec-style service creation for execution |

---

## Environment

| Component | Details |
|-----------|---------|
| Attacker | Kali Linux VM (192.168.0.x) |
| Pivot Point | DC01 (compromised via Kerberoasting + password cracking) |
| Target | WS01 — domain-joined workstation |
| Credentials | `CORP\svc-sql` (cracked from Kerberoast) or NTLM hash |
| Detection | Elastic Security (Kibana) |

---

## Pre-Requisites

- Complete [Project 11 — Kerberoasting Detection](11-kerberoasting-detection.md) to obtain a cracked credential
- WS01 is domain-joined and WEF is configured (see [Project 10](10-wef-sysmon-setup.md))
- Elastic Agent on DC01 is healthy; WS01 events are appearing in Kibana via WEF
- Alternatively, obtain the local Administrator hash from DC01 via `impacket-secretsdump`:

```bash
impacket-secretsdump corp.lab/Administrator:'<password>'@192.168.0.10
```

---

## Part 1: Lateral Movement via SMB / PsExec

### 1.1 PsExec-Style Execution with Impacket

PsExec works by:
1. Connecting to the target's `ADMIN$` share over SMB
2. Uploading a service binary to `C:\Windows\`
3. Creating a new Windows service (Event ID 7045)
4. Executing the service to spawn a shell (Event ID 4624, Sysmon 1)

```bash
# From Kali — move laterally to WS01
impacket-psexec corp.lab/svc-sql:'ServicePass1!'@<ws01-ip>
```

Expected output: Interactive shell on WS01 (`C:\Windows\system32>`)

### 1.2 Verify You're on WS01

```cmd
hostname
whoami
ipconfig
```

### 1.3 Detection: What Events are Generated on WS01?

| Event ID | Channel | Description | Indicator |
|----------|---------|-------------|-----------|
| 4624 | Security | Successful logon | Logon Type 3 (network); `CORP\svc-sql` from Kali IP |
| 7045 | System | New service installed | Service binary in temp path; unusual name |
| 4697 | Security | Service installed in system | Same as 7045 but Security log |
| Sysmon 1 | Sysmon/Operational | Process created | `services.exe` spawning unusual child |
| Sysmon 3 | Sysmon/Operational | Network connection | Inbound SMB connection from Kali IP |
| Sysmon 11 | Sysmon/Operational | File created | Service binary written to disk |

---

## Part 2: Lateral Movement via WMI

WMI execution is often preferred by attackers because it doesn't create a new service (harder to detect via 7045).

### 2.1 WMI Remote Execution

```bash
# From Kali
impacket-wmiexec corp.lab/svc-sql:'ServicePass1!'@<ws01-ip>
```

Expected: Semi-interactive shell via WMI.

### 2.2 Detection: WMI Events

| Event ID | Channel | Description | Indicator |
|----------|---------|-------------|-----------|
| 4624 | Security | Successful logon | Logon Type 3; source = Kali IP |
| Sysmon 1 | Sysmon | Process creation | `WmiPrvSE.exe` spawning `cmd.exe` or other shells |
| Sysmon 3 | Sysmon | Network connection | RPC/DCOM connection (port 135, then dynamic port) |
| 4648 | Security | Explicit credential logon | If using alternate credentials |

### 2.3 WMI in Kibana

```kql
event.code: "1" and event.module: "sysmon"
  and process.parent.name: "WmiPrvSE.exe"
  and not process.name: ("WmiPrvSE.exe" or "svchost.exe")
```

This query catches child processes spawned by the WMI provider host — a common lateral movement pattern.

---

## Part 3: Pass-the-Hash (PtH)

Pass-the-Hash allows an attacker to authenticate using an NTLM hash without needing to crack it.

### 3.1 Obtain NTLM Hash from DC01

If the Administrator account is enabled on WS01 and shares an NTLM hash with DC01:

```bash
# Dump NTLM hashes from DC01 (requires DC admin)
impacket-secretsdump corp.lab/Administrator:'<password>'@192.168.0.10 -just-dc-ntlm
```

Or dump local SAM from a previously compromised machine.

### 3.2 Pass-the-Hash to WS01

```bash
# CrackMapExec — Pass-the-Hash
crackmapexec smb <ws01-ip> -u Administrator -H <ntlm-hash> -d corp.lab

# Impacket psexec with hash
impacket-psexec corp.lab/Administrator@<ws01-ip> -hashes :<ntlm-hash>
```

### 3.3 Detection: PtH Events

| Event ID | Channel | Indicator |
|----------|---------|-----------|
| 4624 | Security | Logon Type 3; Package: `NTLM` (not Kerberos); `CORP\Administrator` from Kali IP |
| 4625 | Security | Failed PtH attempts (if hash is wrong) |
| Sysmon 3 | Sysmon | SMB connection (port 445) from Kali IP |

**PtH signature in Event 4624:**

```
Logon Type: 3 (Network)
Authentication Package: NTLM
Logon Process: NtLmSsp
Workstation Name: <Kali hostname>
Source Network Address: <Kali IP>
```

In Kibana KQL:

```kql
event.code: "4624"
  and winlog.event_data.LogonType: "3"
  and winlog.event_data.AuthenticationPackageName: "NTLM"
  and not winlog.event_data.SubjectUserName: "*$"
```

---

## Part 4: Password Spraying Against WS01

Simulate a targeted password spray against local accounts on WS01:

```bash
crackmapexec smb <ws01-ip> -u Administrator -p 'Winter2024!' --local-auth
```

Watch for 4625 events from WS01 appearing in Kibana via WEF forwarding.

---

## Part 5: Investigation Timeline in Kibana

### 5.1 Create a Timeline

In Kibana: **Security → Timelines → Create new timeline**

Name: `Lateral Movement from Kali → DC01 → WS01`

### 5.2 Add Events to Timeline

Add the following event types to build the full picture:

1. **Authentication (4624/4625)** on WS01 showing source Kali IP
2. **Service installation (7045)** on WS01 if PsExec was used
3. **Sysmon Process Creation (Event 1)** on WS01 showing unusual parent processes
4. **Sysmon Network Connection (Event 3)** from Kali to WS01 on port 445

### 5.3 Timeline KQL Filters

Start with all events involving the attacker's IP and WS01:

```kql
host.name: "WS01" and winlog.event_data.IpAddress: "<kali-ip>"
```

Then expand to correlated Sysmon events on WS01 in the same time window.

---

## Part 6: Detection Rules Added

Based on this lab, add/enable these detection rules:

### Rule: Lateral Movement via PsExec

```kql
event.code: "7045"
  and winlog.event_data.ServiceFileName: ("*\\PSEXESVC.exe" or "*\\PSEXESVC-*")
```

| Setting | Value |
|---------|-------|
| Severity | High |
| MITRE | T1021.002, T1569.002 |

### Rule: NTLM Lateral Movement (Pass-the-Hash)

```kql
event.code: "4624"
  and winlog.event_data.LogonType: "3"
  and winlog.event_data.AuthenticationPackageName: "NTLM"
  and winlog.event_data.TargetUserName: "Administrator"
  and not winlog.event_data.IpAddress: ("127.0.0.1" or "::1")
```

| Setting | Value |
|---------|-------|
| Severity | High |
| MITRE | T1550.002 |

### Rule: WMI Lateral Movement

```kql
event.code: "1" and event.module: "sysmon"
  and process.parent.name: "WmiPrvSE.exe"
  and process.name: ("cmd.exe" or "powershell.exe" or "cscript.exe" or "wscript.exe")
```

| Setting | Value |
|---------|-------|
| Severity | High |
| MITRE | T1047 |

---

## Part 7: Results Summary

| Technique | Method | Events Generated | Alert Fired |
|-----------|--------|-----------------|-------------|
| SMB / PsExec | `impacket-psexec` | 4624 (Type 3), 7045, Sysmon 1/3/11 | ✅ Yes |
| WMI | `impacket-wmiexec` | 4624 (Type 3), Sysmon 1 (WmiPrvSE.exe child) | ✅ Yes |
| Pass-the-Hash | `impacket-psexec -hashes` | 4624 (Type 3, NTLM auth) | ✅ Yes |
| Password Spray | CME spray | 4625 (×5+) | ✅ Yes (via Project 12 rule) |

---

## Hardening Recommendations

| Mitigation | Implementation |
|------------|----------------|
| Restrict admin shares | Disable `ADMIN$` and `C$` for regular users via GPO |
| Disable NTLM where possible | Use Kerberos only; disable NTLMv1 via GPO |
| Local admin password randomisation | Deploy LAPS (Local Administrator Password Solution) |
| Privilege tiering | Don't use domain admin accounts on workstations |
| Credential Guard | Enable Windows Credential Guard to protect NTLM hashes in memory |
| Network segmentation | Segment workstations from servers; block direct lateral SMB |

---

## Key Observations

1. **PsExec is loud** — Event ID 7045 (service creation) is a high-fidelity indicator; very few legitimate tools create services with random names in temp paths
2. **WMI is stealthier** — No service creation, but `WmiPrvSE.exe` spawning `cmd.exe` is still detectable with Sysmon
3. **Pass-the-Hash** — The NTLM authentication package in Event 4624 is a clear indicator; Kerberos logons use "Kerberos" not "NTLM"
4. **WEF is essential** — Without WEF forwarding WS01 events to DC01, the Elastic Agent would miss all workstation-side evidence

---

## Next Steps

- [Project 14 — PowerShell Logging Tuning](14-powershell-logging-tuning.md) — Enable Script Block Logging to catch encoded PowerShell commands used in lateral movement
