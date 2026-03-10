# Project 05 — Endpoint Telemetry with Sysmon

**Skills:** Sysmon, Windows Event Log, Endpoint Telemetry, Log Configuration, XML

---

## Objective

Install and configure Sysmon (System Monitor) on both DC01 and WS01 to enable rich endpoint telemetry — process creation, network connections, command execution, and persistence activity — that forms the foundation of detection in the Elastic SIEM.

---

## Environment

| VM | Hostname | Role |
|----|----------|------|
| Windows Server VM | DC01 | Domain Controller |
| Windows Client VM | WS01 | Domain Workstation |

---

## What Sysmon Provides

Sysmon supplements the default Windows Event Log with high-fidelity events that Windows does not log by default:

| Sysmon Event ID | Description |
|----------------|-------------|
| 1 | Process creation (with full command line and hashes) |
| 3 | Network connection (process, src/dest IP and port) |
| 7 | Image loaded (DLL load events) |
| 11 | File created |
| 12 / 13 | Registry object created / value set |
| 22 | DNS query |
| 23 | File deleted |

---

## Steps Completed

### 1. Downloaded Sysmon and a Community Configuration

On each Windows machine (DC01 and WS01), downloaded Sysmon and the widely-used SwiftOnSecurity Sysmon configuration:

```powershell
# Download Sysmon from Microsoft Sysinternals
Invoke-WebRequest `
  -Uri "https://download.sysinternals.com/files/Sysmon.zip" `
  -OutFile "C:\Tools\Sysmon.zip"

Expand-Archive -Path "C:\Tools\Sysmon.zip" -DestinationPath "C:\Tools\Sysmon"

# Download the SwiftOnSecurity Sysmon config
Invoke-WebRequest `
  -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" `
  -OutFile "C:\Tools\Sysmon\sysmonconfig.xml"
```

### 2. Installed Sysmon on DC01

```powershell
# Run as Administrator on DC01
cd C:\Tools\Sysmon
.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```

**Expected output:**
```
System Monitor v15.x - System activity monitor
...
Sysmon64 started.
```

### 3. Verified Sysmon Was Running on DC01

```powershell
# Check service status
Get-Service Sysmon64

# Verify events are being written
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 10 |
  Select-Object TimeCreated, Id, Message | Format-List
```

### 4. Installed Sysmon on WS01

Repeated the same steps on WS01 (logged in as a domain admin):

```powershell
cd C:\Tools\Sysmon
.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```

Verified:

```powershell
Get-Service Sysmon64
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 10 |
  Select-Object TimeCreated, Id, Message
```

### 5. Confirmed Key Events Were Captured

Triggered a process creation event on WS01 by opening a Command Prompt, then verified Sysmon captured it:

```powershell
# Filter for Event ID 1 (process creation) in the last 5 minutes
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" `
  -FilterHashtable @{Id=1; StartTime=(Get-Date).AddMinutes(-5)} |
  Select-Object TimeCreated, Message
```

**Result:** Sysmon Event ID 1 entries appeared showing the process image, command line, and parent process for `cmd.exe`.

---

## Sysmon Event Log Location

Sysmon events write to a dedicated event log channel:

```
Applications and Services Logs
  └── Microsoft
      └── Windows
          └── Sysmon
              └── Operational
```

In Event Viewer: `Applications and Services Logs → Microsoft → Windows → Sysmon → Operational`

In PowerShell / Elastic Agent config: `Microsoft-Windows-Sysmon/Operational`

---

## Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| Sysmon installation failed — "access denied" | Ran PowerShell as Administrator |
| No events appearing in Sysmon log after install | Confirmed service was running with `Get-Service Sysmon64`; triggered a process to generate test events |
| Config XML download failed on WS01 (no internet) | Copied `sysmonconfig.xml` from DC01 via a shared folder on the LAN |

---

## Key Takeaways

- The SwiftOnSecurity config is a well-tuned starting point that captures high-value events while filtering out known-noisy processes
- Sysmon's process creation events (Event ID 1) include the full command line and parent process — critical for detecting living-off-the-land attacks, PowerShell abuse, and lateral movement tools
- Both DC01 and WS01 need Sysmon installed so all relevant endpoints feed telemetry into the SIEM

---

## Next Steps

→ [Project 06 — Centralized Logging with WEF](06-wef-logging.md)
