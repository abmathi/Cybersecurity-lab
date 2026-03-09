# Project 08 — Incident Response Simulation

**Skills:** Incident Response, Digital Forensics, PICERL Methodology, Log Analysis, Containment, Remediation

---

## Objective

Conduct a full end-to-end incident response exercise simulating an attacker compromising a domain workstation, moving laterally, and exfiltrating data. Follow the PICERL incident response methodology (Preparation, Identification, Containment, Eradication, Recovery, Lessons Learned) to respond to the incident.

---

## Scenario

> **Date/Time:** Simulated weekday morning, business hours
>
> A Splunk alert fires at 09:14: *"Brute Force Detected — 47 failed logins from 10.10.10.10 targeting win10-client1 in 5 minutes."*
>
> Initial investigation reveals this may be a larger incident in progress.

---

## Phase 1 — Preparation

Before the simulation began, the following preparations were made:

- **Playbooks defined:** Documented response procedures for brute force, malware, and data exfiltration scenarios
- **Communication plan:** Defined escalation path (who to notify and when)
- **Tools staged:** IR toolkit ready on the analyst machine (Splunk, Security Onion SOC, Volatility, Autopsy)
- **Baseline established:** Known-good process and network baseline captured for comparison
- **Snapshots taken:** Proxmox snapshots of all VMs before the exercise (for recovery)

---

## Phase 2 — Identification

### 2.1 Initial Alert Triage (T+0:00)

**Splunk alert:** `Brute Force Detected` from `10.10.10.10` against `win10-client1 (10.20.20.20)`

**Initial SPL investigation:**

```spl
# Confirm the brute force
index=windows EventCode=4625 src_ip="10.10.10.10" dest_host="win10-client1"
| timechart count span=1m

# Check if any login succeeded
index=windows EventCode=4624 src_ip="10.10.10.10" dest_host="win10-client1"
| table _time, Account_Name, Logon_Type
```

**Finding:** 47 failed logins, then 1 successful login as `jsmith` at 09:14:32 (Logon Type 3 — network logon).

### 2.2 Scope Expansion (T+0:15)

```spl
# What did jsmith do after logging in?
index=windows (src_ip="10.10.10.10" OR Account_Name="jsmith") earliest=-1h
| transaction Account_Name
| table _time, Account_Name, EventCode, dest_host, CommandLine
```

**Finding:** After logging in, `jsmith`'s account was used to:
1. Access `win10-client2 (10.20.20.21)` via SMB (EventCode 4624, Logon Type 3)
2. Execute PowerShell with an encoded command (Sysmon EventCode 1)
3. Establish outbound connection to `10.10.10.10:4444` (Sysmon EventCode 3)

### 2.3 Security Onion Confirmation

Reviewed Security Onion SOC:
- EternalBlue alert fired at 09:12:15 (2 minutes before the brute force — the initial compromise vector)
- Meterpreter reverse shell detected at 09:14:45
- `conn.log` shows long-duration connection from `10.20.20.20` to `10.10.10.10:4444` (still active)

### 2.4 Incident Classification

| Attribute | Assessment |
|-----------|-----------|
| **Severity** | High |
| **Type** | Unauthorized access + lateral movement + active C2 |
| **Scope** | win10-client1 confirmed compromised; win10-client2 potentially compromised |
| **Status** | Active — attacker has live C2 session |

---

## Phase 3 — Containment

### 3.1 Short-Term Containment (T+0:25)

**Action 1:** Isolate compromised hosts in pfSense by blocking their IPs at the firewall.

```
pfSense Firewall → Add rule:
  Block ALL from 10.20.20.20 and 10.20.20.21 (source: these IPs, destination: any)
```

**Action 2:** Disable the `jsmith` Active Directory account.

```powershell
# On Windows DC
Disable-ADAccount -Identity jsmith
```

**Action 3:** Revoke active Kerberos sessions.

```powershell
# Force re-authentication for all jsmith sessions
Get-ADUser jsmith -Properties LockedOut | Select LockedOut
Invoke-GPUpdate -Computer win10-client1 -Force
```

### 3.2 Evidence Preservation (T+0:30)

Before making any further changes, captured evidence:

```bash
# Memory dump of win10-client1 using Proxmox snapshot
# Proxmox: Snapshot → Take Snapshot (includes RAM state)

# Network capture on Security Onion (download PCAP for C2 session)
# SOC → Alerts → EternalBlue event → PCAP download

# Export Windows event logs from compromised hosts
wevtutil epl Security C:\evidence\security.evtx
wevtutil epl System C:\evidence\system.evtx
```

---

## Phase 4 — Eradication

### 4.1 Identify Attack Entry Point

Confirmed attack chain:
1. **09:12:15** — EternalBlue exploit against `win10-client1` from `10.10.10.10` (Kali)
2. **09:14:32** — Brute force of `jsmith` account (via Meterpreter pivot)
3. **09:14:45** — Meterpreter reverse shell established
4. **09:16:00** — Lateral movement to `win10-client2` via Pass-the-Hash

### 4.2 Identify and Remove Persistence Mechanisms

```bash
# Memory analysis with Volatility
python3 vol.py -f win10-client1-memory.vmem windows.pstree
python3 vol.py -f win10-client1-memory.vmem windows.cmdline
python3 vol.py -f win10-client1-memory.vmem windows.netscan

# Check registry run keys for persistence
python3 vol.py -f win10-client1-memory.vmem windows.registry.printkey \
  --key "SOFTWARE\Microsoft\Windows\CurrentVersion\Run"
```

**Finding:** Meterpreter persistence installed as a registry Run key:
```
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
  "WindowsUpdate" = "C:\Users\jsmith\AppData\Roaming\svchost32.exe"
```

**Removal:**
```powershell
Remove-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" \
  -Name "WindowsUpdate"
Remove-Item "C:\Users\jsmith\AppData\Roaming\svchost32.exe" -Force
```

### 4.3 Patch the Vulnerability

```powershell
# Apply MS17-010 patch
Install-WindowsUpdate -KBArticleID KB4012212 -AcceptAll -AutoReboot
```

### 4.4 Reset Compromised Credentials

```powershell
# Reset jsmith password and re-enable account
Set-ADAccountPassword -Identity jsmith -Reset `
  -NewPassword (ConvertTo-SecureString "NewStr0ngP@ssw0rd!" -AsPlainText -Force)
Enable-ADAccount -Identity jsmith

# Force password change at next logon
Set-ADUser -Identity jsmith -ChangePasswordAtLogon $true

# Reset Administrator account password (hash was dumped)
net user administrator <new_strong_password>
```

---

## Phase 5 — Recovery

### 5.1 Validate Eradication

```bash
# Verify no Meterpreter processes running
# Verify persistence mechanism removed
# Verify patch applied
```

```powershell
# Check Windows Update history
Get-HotFix -Id KB4012212
```

### 5.2 Restore Systems to Production

1. **Remove firewall isolation rules** added during containment
2. **Re-enable jsmith account** after password reset confirmed
3. **Monitor closely for 48 hours** — all events from previously compromised hosts watched with heightened scrutiny

```spl
# Enhanced monitoring query for post-recovery period
index=windows (host="win10-client1" OR host="win10-client2")
| eval priority=if(Account_Name="jsmith","HIGH","NORMAL")
| table _time, host, EventCode, Account_Name, priority
| sort -priority, -_time
```

---

## Phase 6 — Lessons Learned

### 6.1 Timeline Reconstruction

| Time | Event |
|------|-------|
| T-00:00 | EternalBlue exploit hits win10-client1 |
| T+02:17 | Brute force of jsmith begins |
| T+02:32 | jsmith account successfully authenticated |
| T+02:45 | Meterpreter reverse shell established |
| T+04:00 | Lateral movement to win10-client2 |
| T+14:00 | Splunk brute force alert fires (first detection) |
| T+16:00 | Analyst begins investigation |
| T+23:00 | Incident contained |

### 6.2 What Went Well

- Splunk alert correctly identified the brute force attack
- Security Onion captured the full attack chain at the network level
- Evidence preservation (memory dump + PCAP) before eradication maintained forensic integrity

### 6.3 What Could Be Improved

| Gap | Improvement |
|-----|------------|
| 14-minute detection delay | Tune EternalBlue Splunk alert to fire immediately from Security Onion alert feed |
| MS17-010 still unpatched | Implement automated patch management (WSUS/SCCM) |
| jsmith had local admin rights on client2 | Implement least-privilege model; use LAPS for local admin passwords |
| No EDR on endpoints | Deploy Wazuh agents on all Windows hosts for real-time endpoint visibility |

### 6.4 MITRE ATT&CK Mapping

| Technique | ID | Phase |
|-----------|-----|-------|
| Exploit Public-Facing Application (EternalBlue) | T1190 | Initial Access |
| Brute Force | T1110 | Credential Access |
| OS Credential Dumping — LSASS | T1003.001 | Credential Access |
| Pass the Hash | T1550.002 | Lateral Movement |
| Remote Service — SMB | T1021.002 | Lateral Movement |
| Registry Run Keys / Startup | T1547.001 | Persistence |
| Non-Application Layer Protocol (Meterpreter) | T1095 | C2 |

---

## Key Takeaways

- Following a structured IR methodology (PICERL) ensures nothing is missed and actions are taken in the correct order
- Evidence preservation must happen before eradication — memory dumps and PCAPs are critical for understanding what happened
- The attack chain was fully visible across logs: Splunk (authentication), Security Onion (network), and Sysmon (endpoint) each contributed essential context
- Patching and least-privilege are the most impactful mitigations — this entire incident was enabled by an unpatched vulnerability and excessive user privileges
- MITRE ATT&CK framework is invaluable for structuring post-incident reports and communicating with stakeholders
