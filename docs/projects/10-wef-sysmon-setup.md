# Project 10 — WEF & Sysmon Setup

**Status:** ✅ Complete  
**Skills:** Sysmon, Windows Event Forwarding (WEF), WinRM, Group Policy, Windows Event Log, Log Collection

---

## Overview

This project documents the complete configuration of Sysmon and Windows Event Forwarding (WEF) on both DC01 (Domain Controller) and WS01 (domain-joined workstation). WEF enables WS01 to automatically push its Windows event logs to DC01, where a single Elastic Agent ships everything to the SIEM.

### Goals

- Install and configure Sysmon on DC01 and WS01
- Configure DC01 as a WEF Collector (WEC)
- Configure WS01 as a WEF Source via Group Policy
- Verify events from WS01 appear in DC01's Forwarded Events log
- Confirm forwarded events are shipped to Elastic SIEM

---

## Architecture

```
WS01
├── Sysmon → Microsoft-Windows-Sysmon/Operational
├── Windows Security/System/Application logs
└── WEF Source (push via WinRM) ──────────────────────────────┐
                                                               ▼
DC01 (WEF Collector / WEC)                              DC01 Forwarded Events log
├── Sysmon → Microsoft-Windows-Sysmon/Operational             │
├── Windows Security/System/Application logs                   │
└── Forwarded Events log ◄─────────────────────────────────────┘
     │
     └── Elastic Agent → Fleet Server → Elasticsearch → Kibana
```

---

## Environment

| Machine | Role | OS |
|---------|------|----|
| DC01 | WEF Collector, Domain Controller | Windows Server |
| WS01 | WEF Source, Domain Workstation | Windows 10 / 11 |
| Both | Sysmon installed | Sysmon 15.x |

---

## Part 1: Install Sysmon on DC01

### 1.1 Create a Tools Directory

```powershell
New-Item -ItemType Directory -Path "C:\Tools" -Force
New-Item -ItemType Directory -Path "C:\Tools\Sysmon" -Force
```

### 1.2 Download Sysmon

Download from [Microsoft Sysinternals](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon):

```powershell
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" `
  -OutFile "C:\Tools\Sysmon.zip"
Expand-Archive -Path "C:\Tools\Sysmon.zip" -DestinationPath "C:\Tools\Sysmon" -Force
```

### 1.3 Download SwiftOnSecurity Sysmon Config

The [SwiftOnSecurity sysmon-config](https://github.com/SwiftOnSecurity/sysmon-config) is a well-maintained, community-trusted baseline that balances detection coverage with noise reduction:

```powershell
Invoke-WebRequest `
  -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" `
  -OutFile "C:\Tools\Sysmon\sysmonconfig.xml"
```

### 1.4 Install Sysmon with Config

```powershell
cd C:\Tools\Sysmon
.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```

### 1.5 Verify Installation

```powershell
# Check service status
Get-Service Sysmon64

# Verify event log channel is active
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 5 | Select-Object TimeCreated, Id, Message
```

You should see Event ID 1 (Process Created) entries appearing immediately.

---

## Part 2: Configure WEF on DC01 (WEC Collector)

### 2.1 Enable WinRM Service

```powershell
# Run as Administrator on DC01
winrm quickconfig -q
```

### 2.2 Configure Windows Event Collector Service

```powershell
wecutil qc -q
```

This enables and starts the `Windows Event Collector` service (`wecsvc`).

### 2.3 Verify WEC Service

```powershell
Get-Service wecsvc
# Status should be: Running
# StartType should be: Automatic
```

### 2.4 Grant Network Service Read Access to Security Log

The WEF service needs read access to forward Security events:

```powershell
wevtutil sl Security /ca:"O:BAG:SYD:(A;;0xf0007;;;SY)(A;;0x7;;;BA)(A;;0x1;;;BO)(A;;0x1;;;SO)(A;;0x1;;;S-1-5-32-573)(A;;0x1;;;NS)"
```

### 2.5 Create WEF Subscription

Create a source-initiated subscription that collects all key event channels from domain machines:

```powershell
# Save subscription XML
$xml = @'
<Subscription xmlns="http://schemas.microsoft.com/2006/03/windows/events/subscription">
  <SubscriptionId>SOC-Lab-All-Events</SubscriptionId>
  <SubscriptionType>SourceInitiated</SubscriptionType>
  <Description>Collect Security, System, Application, and Sysmon events from domain workstations</Description>
  <Enabled>true</Enabled>
  <Uri>http://schemas.microsoft.com/wbem/wsman/1/windows/EventLog</Uri>
  <ConfigurationMode>MinLatency</ConfigurationMode>
  <Query><![CDATA[
    <QueryList>
      <Query Id="0">
        <Select Path="Security">*</Select>
        <Select Path="System">*</Select>
        <Select Path="Application">*</Select>
        <Select Path="Microsoft-Windows-Sysmon/Operational">*</Select>
        <Select Path="Microsoft-Windows-PowerShell/Operational">*</Select>
      </Query>
    </QueryList>
  ]]></Query>
  <ReadExistingEvents>false</ReadExistingEvents>
  <TransportName>http</TransportName>
  <ContentFormat>RenderedText</ContentFormat>
  <Locale Language="en-US"/>
  <LogFile>ForwardedEvents</LogFile>
  <AllowedSourceDomainComputers>O:NSG:NSD:(A;;GA;;;DC)(A;;GA;;;NS)</AllowedSourceDomainComputers>
</Subscription>
'@

$xml | Out-File "C:\Tools\wef-subscription.xml" -Encoding UTF8
wecutil cs "C:\Tools\wef-subscription.xml"
```

### 2.6 Verify Subscription Created

```powershell
# List all subscriptions
wecutil es

# View subscription details
wecutil gs "SOC-Lab-All-Events"
```

---

## Part 3: Configure WS01 as WEF Source via Group Policy

### 3.1 Create a GPO on DC01

On DC01, open **Group Policy Management** (or run `gpmc.msc`):

1. Right-click the domain `corp.lab` → **Create a GPO in this domain and Link it here**
2. Name: `WEF - Workstation Log Forwarding`

### 3.2 Configure WinRM (WS Management) Service via GPO

Edit the GPO → **Computer Configuration → Policies → Windows Settings → Security Settings → System Services**:

- `Windows Remote Management (WS-Management)`: Set startup to **Automatic**

### 3.3 Configure WinRM Listener via GPO

**Computer Configuration → Policies → Administrative Templates → Windows Components → Windows Remote Management (WinRM) → WinRM Service**:

- **Allow remote server management through WinRM**: Enabled  
  - IPv4 filter: `*` (or limit to DC01's IP for security)

### 3.4 Configure Event Forwarding Subscription Manager

**Computer Configuration → Policies → Administrative Templates → Windows Components → Event Forwarding**:

- **Configure the server address, refresh interval, and issuer certificate authority of a target Subscription Manager**: Enabled
  - `Server=http://DC01.corp.lab:5985/wsman/SubscriptionManager/WEC,Refresh=60`

Replace `DC01.corp.lab` with the actual FQDN or IP of DC01.

### 3.5 Configure Firewall to Allow WinRM

**Computer Configuration → Policies → Windows Settings → Security Settings → Windows Firewall with Advanced Security**:

- Inbound rule: Allow TCP port 5985 from DC01's IP

Or via GPO preference to allow:
- Windows Remote Management (HTTP-In)

### 3.6 Apply GPO on WS01

```powershell
# On WS01 (or wait for normal GPO refresh cycle)
gpupdate /force
```

---

## Part 4: Install Sysmon on WS01

Repeat [Part 1](#part-1-install-sysmon-on-dc01) on WS01. Use the same Sysmon binary and `sysmonconfig.xml`.

For centralized deployment via Group Policy (optional):
1. Place `Sysmon64.exe` and `sysmonconfig.xml` on a SYSVOL share
2. Create a startup script GPO to run: `\\DC01\NETLOGON\Sysmon64.exe -accepteula -i \\DC01\NETLOGON\sysmonconfig.xml`

---

## Part 5: Verify the WEF Pipeline

### 5.1 Generate a Test Event on WS01

On WS01 (Command Prompt):

```cmd
eventcreate /T INFORMATION /ID 1000 /L APPLICATION /D "WEF Pipeline Test - from WS01"
```

### 5.2 Check Forwarded Events on DC01

On DC01, open **Event Viewer → Windows Logs → Forwarded Events**

You should see the test event from WS01 within ~60 seconds (the subscription refresh interval).

Or via PowerShell:

```powershell
Get-WinEvent -LogName "ForwardedEvents" -MaxEvents 10 | `
  Select-Object TimeCreated, MachineName, Id, Message | `
  Format-Table -AutoSize
```

Look for events with `MachineName` matching WS01's hostname.

### 5.3 Check Subscription Runtime Status

```powershell
# On DC01 — check which machines are actively forwarding
wecutil gr "SOC-Lab-All-Events"
```

Expected output includes WS01's machine name with a `LastHeartbeatTime` and `LastEventReceivedTime`.

---

## Part 6: Verify Forwarded Events in Kibana

After confirming WEF is working, verify that the Elastic Agent on DC01 is shipping forwarded events to Kibana.

In Kibana **Discover**, search:

```kql
winlog.channel: "ForwardedEvents"
```

Or filter by WS01's hostname:

```kql
host.name: "WS01"
```

You should see events from WS01 appearing in Kibana even though the Elastic Agent is only installed on DC01.

---

## Results

| Check | Result |
|-------|--------|
| Sysmon on DC01 | ✅ Installed, logging to Sysmon/Operational |
| Sysmon on WS01 | ✅ Installed, same config as DC01 |
| WEC service on DC01 | ✅ Running, subscription active |
| WinRM on DC01 | ✅ Configured via `winrm quickconfig` |
| GPO pushing WEF config | ✅ Applied to domain workstations |
| WS01 forwarding events | ✅ Events visible in DC01 Forwarded Events |
| Manual test event | ✅ `eventcreate` test confirmed end-to-end |
| Forwarded events in Kibana | ✅ WS01 events visible in Elastic Security via `winlog.channel: "ForwardedEvents"` |

---

## Troubleshooting

| Issue | Resolution |
|-------|-----------|
| WS01 not appearing in `wecutil gr` | Ensure GPO applied (`gpupdate /force`); check WinRM service running on WS01 |
| Events not showing in Forwarded Events | Verify DC01 firewall allows inbound TCP 5985; check WEC subscription is enabled |
| Sysmon not generating events | Run `Get-Service Sysmon64`; reinstall with `.\Sysmon64.exe -i sysmonconfig.xml` |
| Forwarded events missing in Kibana | Ensure `Forwarded Events` is included in the Windows integration channel list in Elastic agent policy |

---

## Key Observations

1. **WEF is a native Windows capability** — no additional software required on WS01; only GPO configuration needed
2. **Source-initiated subscriptions scale better** — workstations call in to the WEC; no need to enumerate endpoints manually
3. **Single Elastic Agent covers multiple hosts** — DC01's agent ingests both DC01's own logs and WS01's forwarded events, minimising agent deployments
4. **Sysmon enriches WEF** — when Sysmon/Operational is included in the WEF subscription, Sysmon events from WS01 also flow through to DC01 and then to the SIEM

---

## MITRE ATT&CK Coverage Added

| Technique | ID | Evidence Source |
|-----------|----|-----------------|
| Lateral Movement — SMB/PsExec | T1021.002 | Sysmon Event 1 (process creation) on WS01 |
| Credential Access | T1003 | Sysmon Event 10 (lsass process access) on WS01 |
| Persistence — Services | T1543.003 | Security Event 7045 from WS01 |
| Account Logon | T1078 | Security Events 4624/4625 from WS01 |

---

## Next Steps

- [Project 11 — Kerberoasting Detection](11-kerberoasting-detection.md) — Simulate Kerberoast from Kali; detect via Event ID 4769
- [Project 12 — Custom Detection Rules](12-custom-detection-rules.md) — Build Elastic detection rules for brute force and SPN scanning
- [Project 13 — Lateral Movement Lab](13-lateral-movement-lab.md) — Simulate lateral movement to WS01 and detect it
