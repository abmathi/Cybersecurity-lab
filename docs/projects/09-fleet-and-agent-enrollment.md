# Project 09 — Elastic SIEM Setup

**Status:** ✅ Complete  
**Skills:** Elastic Stack, Fleet Server, Ubuntu Server ARM64, Elasticsearch, Kibana, Elastic Agent, Windows Integration, Sysmon Integration

---

## Overview

This project documents the deployment of a self-hosted Elastic SIEM on an Apple Silicon Mac (M1 Pro) running Ubuntu Server (ARM64) as a headless virtual machine. The SIEM ingests Windows Event Logs and Sysmon telemetry from DC01 in real-time via the Elastic Agent and Fleet Server.

### Goals

- Deploy Elasticsearch + Kibana on Ubuntu ARM64 (CLI-only / headless)
- Configure Fleet Server for centralised agent management
- Install and enrol Elastic Agent on DC01 (Windows Server)
- Create a Windows agent policy with Windows + Sysmon integrations
- Confirm real-time log ingestion in Kibana

---

## Environment

| Component | Details |
|-----------|---------|
| SIEM Host | Apple Silicon M1 Pro Mac |
| SIEM OS | Ubuntu Server 22.04 LTS (ARM64 VM, CLI-only) |
| Elastic Version | 8.x (free Basic licence) |
| Endpoint | DC01 — Windows Server (corp.lab domain controller) |
| Network | Flat home LAN: 192.168.0.0/24 |

---


## Part 1: Configure Fleet Server

### 1.1 Generate Fleet Server Service Token

In Kibana: **Management → Fleet → Settings → Outputs**

Confirm the default output points to `https://localhost:9200` (or update with the SIEM IP if connecting from external agents).

In Kibana: **Management → Fleet → Add Fleet Server**

Select **Quick Start** and follow the wizard to generate a service token.

### 1.2 Install Elastic Agent as Fleet Server on Ubuntu

```bash
# Download Elastic Agent for Linux ARM64
curl -L -O https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.x.x-linux-arm64.tar.gz
tar xzvf elastic-agent-8.x.x-linux-arm64.tar.gz
cd elastic-agent-8.x.x-linux-arm64

# Install as Fleet Server
sudo ./elastic-agent install \
  --fleet-server-es=https://localhost:9200 \
  --fleet-server-service-token=<service-token> \
  --fleet-server-policy=fleet-server-policy \
  --fleet-server-es-insecure \
  --fleet-server-insecure-http
```

### 1.3 Verify Fleet Server is Healthy

In Kibana: **Management → Fleet → Agents**

The Fleet Server agent should appear with status **Healthy**.

---

## Part 2: Create Windows Agent Policy

### 2.1 Create Policy in Kibana

**Management → Fleet → Agent Policies → Create agent policy**

- Name: `Windows Endpoints`
- Description: `Policy for Windows domain machines (DC01, WS01)`

### 2.2 Add Windows Integration

**Agent Policy → Add integration → search "Windows" → select Windows**

Configure:
- Enable **Security event logs** (Event ID 4624, 4625, 4634, 4648, 4688, 4769, etc.)
- Enable **System event logs**
- Enable **Application event logs**
- Enable **Forwarded Events** (to capture WEF events from WS01 forwarded to DC01)

### 2.3 Add Sysmon Integration

**Agent Policy → Add integration → search "Custom Windows Event Logs"**

- Channel name: `Microsoft-Windows-Sysmon/Operational`
- Dataset name: `windows.sysmon_operational`

This captures all Sysmon events (process creation, network connections, etc.).

![Create Policies](../assets/WS01%20setup/7-ws01-policy.png)

---

## Part 3: Enrol Elastic Agent on DC01

### 3.1 Get Enrollment Token

In Kibana: **Management → Fleet → Enrollment tokens**

Select the **Windows Endpoints** policy and copy the enrollment token.

### 3.2 Install Elastic Agent on DC01 & WS01 (PowerShell as Administrator)

![Install Agents](../assets/WS01%20setup/8-install-elastic-agent-on-ws01.png)

```powershell
# Download Elastic Agent for Windows
$url = "https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.x.x-windows-x86_64.zip"
Invoke-WebRequest -Uri $url -OutFile "C:\Tools\elastic-agent.zip"
Expand-Archive -Path "C:\Tools\elastic-agent.zip" -DestinationPath "C:\Tools\"

# Navigate to the extracted folder
cd "C:\Tools\elastic-agent-8.x.x-windows-x86_64"

# Install and enrol
.\elastic-agent.exe install `
  --url=https://<ubuntu-siem-ip>:8220 `
  --enrollment-token=<enrollment-token> `
  --insecure
```

### 3.3 Verify Agent is Enrolled

![Verify Enrollment](../assets/WS0101%20setup/9-ws01-enrolled.png)

```powershell
# Check Windows service
Get-Service "Elastic Agent"
```

In Kibana: **Fleet → Agents** — DC01 should appear with status **Healthy**.

---

## Part 4: Verify Real-Time Log Ingestion

### 4.1 Check Logs in Discover

![Check Logs](../assets/elastic/logs%20flowing.png)

In Kibana: **Discover**

- Set index pattern to `logs-*` or `logs-windows.*`
- Add filter: `event.module: "windows"`
- Verify events from DC01 are arriving in real-time

### 4.2 Generate a Test Event

On DC01 (Command Prompt):

```cmd
eventcreate /T WARNING /ID 999 /L APPLICATION /D "Elastic SIEM Pipeline Test - DC01"
```

In Kibana Discover, search:

```kql
message: "Elastic SIEM Pipeline Test"
```

The event should appear within 30 seconds.

### 4.3 Verify Sysmon Events

![Verify Events](../assets/WS01%20setup/10-ws01-sysmon-logs.png)

In Kibana Discover, filter:

```kql
event.module: "sysmon" and event.code: "1"
```

Sysmon Event ID 1 (Process Created) events should be visible, showing process launches on DC01.

### 4.4 Check Elastic Security App

In Kibana: **Security → Overview**

The dashboard should show:
- Events count (last 24h)
- Host data from DC01
- Active rules status

---

## Results

| Check | Result |
|-------|--------|
| Elasticsearch running | ✅ Single-node cluster healthy |
| Kibana accessible | ✅ `http://<siem-ip>:5601` reachable from home LAN |
| Fleet Server healthy | ✅ Visible in Fleet management UI |
| Windows agent policy created | ✅ `Windows Endpoints` policy with Windows + Sysmon integrations |
| DC01 agent enrolled | ✅ Agent status Healthy in Fleet |
| Windows events in Kibana | ✅ Security/System/Application logs arriving in real-time |
| Sysmon events in Kibana | ✅ Process creation, network events visible |
| Forwarded Events | ✅ WS01 events forwarded via WEF visible in Discover |

---

## Key Observations

1. **ARM64 compatibility** — Elastic 8.x has full ARM64 support; no additional steps needed for Apple Silicon
2. **Headless deployment** — entire setup done via SSH and CLI; Kibana accessed from browser on separate machine on the LAN
3. **Single-node is sufficient** — for a home lab with 2-3 endpoints, single-node Elasticsearch handles the event volume with ease
4. **Free Basic licence** — all Elastic Security features used in this lab (detection rules, alerts, investigation timelines) are available under the free Basic licence

---

## MITRE ATT&CK Coverage Enabled

By establishing this logging pipeline, the following MITRE ATT&CK techniques become visible in the SIEM:

| Technique | ID | Events |
|-----------|----|--------|
| Valid Accounts | T1078 | 4624, 4625, 4648 |
| Kerberoasting | T1558.003 | 4769 |
| Brute Force | T1110 | 4625 (multiple) |
| OS Credential Dumping | T1003 | Sysmon Event ID 10 (lsass access) |
| Lateral Movement via SMB | T1021.002 | 4624 (logon type 3), Sysmon net connections |
| PowerShell | T1059.001 | Event ID 4104 (with Script Block Logging) |

---

## Next Steps

- [Project 05 — Endpoint Telemetry with Sysmon](05-sysmon-setup.md) — Sysmon installation and configuration on DC01 and WS01
- [Project 06 — Centralized Logging with WEF](06-wef-logging.md) — WEF and Sysmon configuration walkthrough
- [Project 11 — Kerberoasting Detection](11-kerberoasting-detection.md) — Simulate a Kerberoast and detect via Event ID 4769
- [Project 12 — Custom Detection Rules](12-custom-detection-rules.md) — Build Elastic detection rules for common AD attacks
