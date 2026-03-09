# Project 14 — PowerShell Logging Tuning

**Status:** 🔜 Next Step  
**Skills:** PowerShell, Script Block Logging, Module Logging, Transcription Logging, Group Policy, Elastic Security, KQL, MITRE ATT&CK T1059.001

---

## Overview

PowerShell is one of the most commonly abused tools by attackers — it is built into every Windows system, trusted by default, and capable of downloading payloads, running in memory, and bypassing traditional defenses. This project enables comprehensive PowerShell logging on DC01 and WS01, tunes the Elastic Windows integration to capture these logs, and builds detection rules for common PowerShell-based attack patterns.

### Goals

- Enable Script Block Logging (Event ID 4104) — captures the full content of every PowerShell script block executed
- Enable Module Logging (Event ID 4103) — captures pipeline execution details
- Enable PowerShell Transcription — writes full session transcripts to disk
- Tune the Elastic Windows integration to ingest PowerShell event logs
- Build detection rules for encoded PowerShell, download cradles, and AMSI bypass attempts

---

## MITRE ATT&CK Reference

| Technique | ID | Description |
|-----------|-----|-------------|
| PowerShell | T1059.001 | Use PowerShell for execution |
| Obfuscated Files or Information | T1027 | Base64-encoded or obfuscated PowerShell commands |
| Ingress Tool Transfer | T1105 | Download payloads via IEX/WebClient/BitsAdmin |
| Defense Evasion — AMSI Bypass | T1562.001 | Disable AMSI to bypass script scanning |

---

## Background: PowerShell Log Types

| Log Type | Event ID | What It Captures | Value |
|----------|----------|-----------------|-------|
| **Script Block Logging** | 4104 | Full content of every script block executed, including decoded base64 | 🔴 High — catches obfuscated code |
| **Module Logging** | 4103 | Pipeline execution records with input/output | 🟡 Medium — detailed but verbose |
| **Transcription** | (file) | Complete session transcript to disk | 🟡 Medium — useful for forensics |
| **Protected Event Logging** | 4104 (encrypted) | Script blocks encrypted with public key | 🟢 High security — requires cert |

For this lab, **Script Block Logging** is the highest priority.

---

## Part 1: Enable PowerShell Logging via Group Policy

### 1.1 Open Group Policy Management on DC01

```powershell
# Or open from Start menu
gpmc.msc
```

Create or edit the existing domain GPO.

### 1.2 Enable Script Block Logging

Navigate to:
**Computer Configuration → Policies → Administrative Templates → Windows Components → Windows PowerShell**

Set: **Turn on PowerShell Script Block Logging** → **Enabled**

Optional: Enable **Log script block invocation start/stop events** for even more detail.

### 1.3 Enable Module Logging

Same GPO path:

Set: **Turn on Module Logging** → **Enabled**  
Module names to log: `*` (wildcard for all modules)

### 1.4 Enable Transcription Logging

Set: **Turn on PowerShell Transcription** → **Enabled**  
Transcript output directory: `C:\PSTranscripts` (or a network share for centralised collection)  
Enable: Include invocation headers (adds timestamps and user/process info)

### 1.5 Apply the GPO

```powershell
# On DC01 and WS01
gpupdate /force
```

### 1.6 Verify Script Block Logging is Active

```powershell
# Test — run any PowerShell command and check the log
Get-Date
Get-WinEvent -LogName "Microsoft-Windows-PowerShell/Operational" -MaxEvents 5 |
  Select-Object TimeCreated, Id, Message
```

You should see Event ID 4104 entries for the commands just run.

---

## Part 2: Enable via Registry (Alternative — No GPO Required)

For machines not using GPO or for testing:

```powershell
# Enable Script Block Logging
$path = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging"
if (-not (Test-Path $path)) { New-Item -Path $path -Force }
Set-ItemProperty -Path $path -Name "EnableScriptBlockLogging" -Value 1
Set-ItemProperty -Path $path -Name "EnableScriptBlockInvocationLogging" -Value 1

# Enable Module Logging
$path2 = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ModuleLogging"
if (-not (Test-Path $path2)) { New-Item -Path $path2 -Force }
Set-ItemProperty -Path $path2 -Name "EnableModuleLogging" -Value 1

$path3 = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ModuleLogging\ModuleNames"
if (-not (Test-Path $path3)) { New-Item -Path $path3 -Force }
Set-ItemProperty -Path $path3 -Name "*" -Value "*"
```

---

## Part 3: Tune Elastic Windows Integration

The PowerShell Operational log needs to be added to the Elastic agent policy.

### 3.1 Update Agent Policy in Kibana

**Management → Fleet → Agent Policies → Windows Endpoints → Windows Integration → Edit**

In the **Windows Event Log** section, ensure these channels are included:

| Channel | Event IDs | Purpose |
|---------|-----------|---------|
| `Microsoft-Windows-PowerShell/Operational` | 4103, 4104, 4105, 4106 | Script block and module logging |
| `Windows PowerShell` | 400, 403, 600, 800 | Legacy PowerShell log |
| `Microsoft-Windows-PowerShell/Analytic` | All | (Optional — very verbose) |

### 3.2 Verify PowerShell Events in Kibana

In Kibana Discover:

```kql
winlog.channel: "Microsoft-Windows-PowerShell/Operational"
```

Or filter by event code:

```kql
event.code: "4104"
```

---

## Part 4: Test PowerShell Attack Patterns

### 4.1 Encoded Command Execution (Simulated Attack)

Attackers often base64-encode PowerShell to evade string-based detection:

```powershell
# Generate a base64-encoded command (this is a safe test)
$cmd = "Write-Output 'PowerShell logging test - encoded command'"
$bytes = [System.Text.Encoding]::Unicode.GetBytes($cmd)
$encoded = [Convert]::ToBase64String($bytes)
powershell.exe -EncodedCommand $encoded
```

Check in Kibana for Event ID 4104 with the decoded content — Script Block Logging **automatically decodes** base64 and logs the plaintext script block.

### 4.2 Download Cradle Simulation (Safe Test)

```powershell
# Test IEX download cradle pattern (safe — pointing to a local resource)
$url = "http://127.0.0.1/test.txt"
Invoke-Expression "Write-Output 'Download cradle simulation test'"
```

Check Event ID 4104 — the `Invoke-Expression` and content will be logged.

### 4.3 AMSI Bypass Attempt (For Detection Testing)

> ⚠️ **For detection purposes only** — the below string is a common AMSI bypass string that will be flagged by defenders. It does not actually bypass AMSI when run in a logged environment.

```powershell
# This generates a log entry that matches AMSI bypass patterns
# Search for this in your rules — it should trigger an alert
Write-Output "amsiInitFailed bypass pattern simulation"
```

---

## Part 5: Detection Rules for PowerShell

### 5.1 Rule: Encoded PowerShell Command Execution

```kql
event.code: "4104"
  and winlog.event_data.ScriptBlockText: "-EncodedCommand"
```

Or via process command line (Sysmon Event 1):

```kql
event.code: "1" and event.module: "sysmon"
  and process.name: "powershell.exe"
  and process.command_line: (*-EncodedCommand* or *-enc* or *-ec*)
```

| Setting | Value |
|---------|-------|
| Name | `Encoded PowerShell Command Execution` |
| Severity | Medium |
| MITRE | T1059.001, T1027 |

### 5.2 Rule: PowerShell Download Cradle

```kql
event.code: "4104"
  and (
    winlog.event_data.ScriptBlockText: "IEX" or
    winlog.event_data.ScriptBlockText: "Invoke-Expression" or
    winlog.event_data.ScriptBlockText: "Net.WebClient" or
    winlog.event_data.ScriptBlockText: "DownloadString" or
    winlog.event_data.ScriptBlockText: "DownloadFile" or
    winlog.event_data.ScriptBlockText: "Invoke-WebRequest"
  )
  and winlog.event_data.ScriptBlockText: ("http://" or "https://")
```

| Setting | Value |
|---------|-------|
| Name | `PowerShell Download Cradle Detected` |
| Severity | High |
| MITRE | T1105, T1059.001 |

### 5.3 Rule: AMSI Bypass Attempt

```kql
event.code: "4104"
  and (
    winlog.event_data.ScriptBlockText: "amsiInitFailed" or
    winlog.event_data.ScriptBlockText: "AmsiScanBuffer" or
    winlog.event_data.ScriptBlockText: "amsiContext" or
    winlog.event_data.ScriptBlockText: "AMSI" and
    winlog.event_data.ScriptBlockText: ("Bypass" or "Disable" or "Patch")
  )
```

| Setting | Value |
|---------|-------|
| Name | `PowerShell AMSI Bypass Attempt` |
| Severity | High |
| MITRE | T1562.001 |

### 5.4 Rule: PowerShell Running from Unusual Location

```kql
event.code: "1" and event.module: "sysmon"
  and process.name: ("powershell.exe" or "pwsh.exe")
  and not process.executable: (
    "C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\powershell.exe" or
    "C:\\Windows\\SysWOW64\\WindowsPowerShell\\v1.0\\powershell.exe" or
    "C:\\Program Files\\PowerShell\\7\\pwsh.exe"
  )
```

| Setting | Value |
|---------|-------|
| Name | `PowerShell Executed from Non-Standard Location` |
| Severity | High |
| MITRE | T1059.001 |

---

## Part 6: Noise Reduction — Exclusions

PowerShell logging can generate significant volume. Common exclusions to reduce false positives:

### Known Legitimate Scripts to Exclude

```kql
not winlog.event_data.Path: (
  "C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\Modules\\*" or
  "C:\\Program Files\\*" or
  "C:\\Program Files (x86)\\*"
)
```

### Exclude Known Admin Activity

Add exceptions for your specific environment:
- Elastic Agent update scripts
- Windows Update PowerShell commands
- Scheduled task maintenance scripts

In Kibana: **Rule → Edit → Exceptions → Add exception**

---

## Part 7: Log Volume Considerations

| Log Type | Expected Volume | Storage Impact |
|----------|----------------|----------------|
| Script Block Logging (4104) | High on active machines | ~50–200 MB/day per active endpoint |
| Module Logging (4103) | Very High | Can overwhelm — consider enabling selectively |
| Transcription | Medium | Disk-based on endpoint; ship via Elastic Agent |

For the home lab with 2 endpoints (DC01 + WS01), all logging types are manageable.

---

## Results

| Configuration | Status |
|---------------|--------|
| Script Block Logging (4104) | ✅ Enabled via GPO |
| Module Logging (4103) | ✅ Enabled via GPO |
| Transcription Logging | ✅ Enabled, writing to `C:\PSTranscripts` |
| Elastic integration updated | ✅ PowerShell/Operational channel added |
| Events appearing in Kibana | ✅ Event ID 4104 visible in real-time |
| Encoded command test | ✅ Detected and decoded in 4104 log |
| Download cradle rule | ✅ Alert fired on test |
| AMSI bypass rule | ✅ Alert fired on test pattern |

---

## Key Observations

1. **Script Block Logging is game-changing** — Base64-encoded PowerShell is automatically decoded before logging; obfuscation is significantly defeated
2. **Volume management is important** — Module Logging (4103) can generate thousands of events per minute on active machines; Script Block Logging (4104) is more targeted
3. **PowerShell attacks are pervasive** — Many post-exploitation frameworks (Cobalt Strike, Empire, Metasploit PowerShell modules) rely on PowerShell; this logging catches most of them
4. **Combine with Sysmon** — Sysmon Event 1 (process creation) for `powershell.exe` with suspicious command-line arguments complements Script Block Logging

---

## Summary: Complete Detection Coverage

With Projects 09–14 complete, the lab now detects:

| Attack | Detection Method | Event IDs / Rules |
|--------|-----------------|-------------------|
| Password Spray | Elastic threshold rule | 4625 (×5+) |
| Kerberoasting | Custom query rule | 4769 + RC4 |
| PsExec Lateral Movement | Service creation rule | 7045 |
| Pass-the-Hash | NTLM logon rule | 4624 (Type 3 + NTLM) |
| WMI Lateral Movement | Sysmon process rule | Sysmon 1 (WmiPrvSE.exe child) |
| Encoded PowerShell | Script block logging rule | 4104 + `-EncodedCommand` |
| PowerShell Download Cradle | Script block logging rule | 4104 + `IEX`/`DownloadString` |
| AMSI Bypass | Script block logging rule | 4104 + AMSI strings |
