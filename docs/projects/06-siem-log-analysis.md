# Project 06 — SIEM & Log Analysis

**Skills:** Splunk, SPL (Search Processing Language), Log Analysis, Dashboards, Alert Creation, Security Operations

---

## Objective

Configure Splunk as the lab SIEM, ingest logs from all key sources, write detection queries in SPL, build a SOC-style dashboard, and create automated alerts for suspicious activity observed during lab exercises.

---

## Environment

| Component | IP | Details |
|-----------|-----|---------|
| Splunk | 10.30.30.20:8000 | Splunk Enterprise 9.x |
| Windows DC | 10.20.20.10 | Windows Event Log forwarder |
| win10-client1/2 | 10.20.20.20/21 | Windows Event Log forwarders |
| Metasploitable3 | 10.20.20.30 | Syslog via rsyslog |
| pfSense | 10.20.20.1 | Syslog firewall logs |

---

## Part 1 — Data Ingestion Setup

### 1.1 Enable Splunk Receiving

In Splunk web UI:

1. **Settings → Forwarding and receiving → Configure receiving**
2. Add port `9997`

### 1.2 Install Windows Universal Forwarder

On each Windows host, install Splunk Universal Forwarder and configure it to:
- Forward to `10.30.30.20:9997`
- Collect Security, System, and Application event logs
- Collect Sysmon logs (if Sysmon is installed)

### 1.3 Install Sysmon for Enhanced Windows Logging

Sysmon provides detailed process creation, network connection, and file creation logs that are far more useful for detection than default Windows events.

```powershell
# Download and install Sysmon with community config
Invoke-WebRequest -Uri https://download.sysinternals.com/files/Sysmon.zip -OutFile Sysmon.zip
Expand-Archive Sysmon.zip
.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```

### 1.4 Linux Syslog Forwarding

```bash
# /etc/rsyslog.conf on Metasploitable3
*.* @10.30.30.20:514

# Add UDP input in Splunk: Settings → Data inputs → UDP → Port 514
```

### 1.5 pfSense Log Forwarding

In pfSense: **Status → System Logs → Settings → Remote Logging**
- Remote log servers: `10.30.30.20:514`
- Check: Firewall Events, DHCP, General System Events

---

## Part 2 — Index and Source Type Configuration

| Index | Source Type | Contents |
|-------|------------|----------|
| `windows` | `WinEventLog:Security` | Security events (logons, privilege use) |
| `windows` | `WinEventLog:System` | System events |
| `windows` | `XmlWinEventLog:Microsoft-Windows-Sysmon/Operational` | Sysmon events |
| `linux` | `syslog` | Linux system logs |
| `network` | `pfsense` | Firewall logs |

---

## Part 3 — Key SPL Detection Queries

### 3.1 Failed Login Attempts (Brute Force Detection)

```spl
index=windows EventCode=4625
| stats count by Account_Name, Source_Network_Address, host
| where count > 5
| sort -count
```

### 3.2 Successful Login After Multiple Failures

```spl
index=windows (EventCode=4625 OR EventCode=4624)
| eval status=if(EventCode=4624,"success","failure")
| stats values(status) as statuses, count(eval(status="failure")) as failures,
        count(eval(status="success")) as successes by Account_Name, Source_Network_Address
| where failures > 3 AND successes > 0
| sort -failures
```

### 3.3 Kerberoasting Detection

```spl
index=windows EventCode=4769
  TicketEncryptionType=0x17
  ServiceName!=krbtgt
  ServiceName!="*$"
| stats count by Account_Name, ServiceName, Client_Address
| where count > 1
```

### 3.4 Pass-the-Hash Detection

```spl
index=windows EventCode=4624
  LogonType=3
  AuthenticationPackageName=NTLM
  NOT Account_Name="ANONYMOUS LOGON"
| stats count by Account_Name, Workstation_Name, IpAddress
| sort -count
```

### 3.5 New Local Admin Account Created

```spl
index=windows (EventCode=4720 OR EventCode=4732)
| eval event_type=case(EventCode=4720,"Account Created",EventCode=4732,"Added to Group",true(),"Unknown")
| table _time, host, event_type, Target_Account_Name, Subject_Account_Name
| sort -_time
```

### 3.6 PowerShell Execution (Encoded Commands — Common Malware Indicator)

```spl
index=windows source="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=1
  CommandLine="*-enc*" OR CommandLine="*-EncodedCommand*"
| table _time, host, User, ParentCommandLine, CommandLine
| sort -_time
```

### 3.7 Lateral Movement — New Network Connections

```spl
index=windows source="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=3
  DestinationPort IN (135, 445, 3389, 5985, 5986)
  NOT DestinationIp="127.0.0.1"
| stats count by SourceIp, DestinationIp, DestinationPort, Image
| sort -count
```

### 3.8 Suspicious Process Execution (Living off the Land)

```spl
index=windows source="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=1
  (Image="*\\cmd.exe" OR Image="*\\powershell.exe" OR Image="*\\wscript.exe"
   OR Image="*\\cscript.exe" OR Image="*\\mshta.exe" OR Image="*\\regsvr32.exe")
  ParentImage IN ("*\\winword.exe","*\\excel.exe","*\\outlook.exe","*\\svchost.exe")
| table _time, host, User, ParentImage, Image, CommandLine
```

### 3.9 pfSense — Traffic from Attack Segment to Blocked Zones

```spl
index=network sourcetype=pfsense
  src_ip="10.10.10.*"
  (dest_ip="10.30.30.*" OR dest_ip="10.0.0.*")
  action=block
| stats count by src_ip, dest_ip, dest_port
| sort -count
```

### 3.10 Linux — Sudo Command Execution

```spl
index=linux "sudo" "command" NOT "sudo: pam_unix"
| rex "(?<user>[a-z_][a-z0-9_-]{0,31}) : .* COMMAND=(?<command>.*)"
| stats count by user, command, host
| sort -count
```

---

## Part 4 — SOC Dashboard

Built a Splunk dashboard (`SOC Overview`) with the following panels:

| Panel | Query | Visualization |
|-------|-------|--------------|
| Failed Logins (Last 24h) | EventCode=4625 | Single value |
| Logins by Country | EventCode=4624 with geo lookup | Choropleth map |
| Top Talkers | Network connections by source IP | Bar chart |
| Alert Timeline | All triggered alerts | Area chart |
| Recent Suspicious Events | Combined alert query | Table |
| Firewall Blocks | pfSense block events | Pie chart |

---

## Part 5 — Automated Alerts

Configured the following scheduled alerts in Splunk (run every 5 minutes on a rolling 5-minute window):

| Alert Name | Trigger Condition | Action |
|-----------|-------------------|--------|
| Brute Force Detected | > 10 failed logins from same IP in 5 min | Email + log event |
| Kerberoasting Attempt | Any RC4 TGS request | Email + log event |
| New Admin Account | Any 4720 event outside business hours | Email |
| Encoded PowerShell | Any `-enc` in PowerShell args | Email + log event |
| Attack Segment to Monitoring | Any blocked traffic from VLAN 10 → VLAN 30 | Email |

---

## Part 6 — Results from Lab Exercises

After running the attacks from Projects 02–04, Splunk captured:

| Exercise | Events Captured | Alerts Triggered |
|----------|----------------|-----------------|
| AD brute force (Hydra) | 847 EventCode=4625 events | ✅ Brute Force alert (fired in < 1 min) |
| Kerberoasting | 3 EventCode=4769 RC4 tickets | ✅ Kerberoasting alert |
| EternalBlue exploit | Sysmon process injection events | ✅ Encoded PowerShell alert |
| Pass-the-Hash | EventCode=4624 Type 3 NTLM logons | ✅ Pass-the-Hash alert |
| Reverse shell (nc) | Sysmon network connection events | ✅ Lateral Movement alert |

---

## Key Takeaways

- Sysmon dramatically improves detection capability over default Windows logging — essential for any Windows environment
- SPL is a powerful query language; investment in learning it pays off significantly in SOC work
- Correlating events across multiple data sources (Windows + network + Linux) reveals attack chains that individual log sources would miss
- Properly tuned alerts with appropriate thresholds prevent alert fatigue while still catching real threats

---

## Next Steps

→ [Project 07 — Intrusion Detection with Security Onion](07-intrusion-detection.md)
