# Setup Guide

This guide walks through the complete process of building the SOC / Blue Team cybersecurity home lab from scratch — from setting up Active Directory through to deploying Elastic SIEM and enrolling endpoints.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Set Up DC01 — Windows Domain Controller](#2-set-up-dc01--windows-domain-controller)
3. [Install Sysmon on DC01](#3-install-sysmon-on-dc01)
4. [Configure Windows Event Forwarding (WEF)](#4-configure-windows-event-forwarding-wef)
5. [Set Up WS01 — Domain-Joined Workstation](#5-set-up-ws01--domain-joined-workstation)
6. [Install Sysmon on WS01 & Configure WEF Client](#6-install-sysmon-on-ws01--configure-wef-client)
7. [Deploy Elastic Stack on Ubuntu ARM64](#7-deploy-elastic-stack-on-ubuntu-arm64)
8. [Configure Fleet Server & Enroll Elastic Agent](#8-configure-fleet-server--enroll-elastic-agent)
9. [Add Windows & Sysmon Integrations](#9-add-windows--sysmon-integrations)
10. [Verify End-to-End Log Pipeline](#10-verify-end-to-end-log-pipeline)
11. [Set Up Kali Linux Attack VM](#11-set-up-kali-linux-attack-vm)

---

## 1. Prerequisites

Before starting, ensure you have:

- **Two Windows machines** (or VMs):
  - One for DC01 (Windows Server — any recent version with Desktop Experience)
  - One for WS01 (Windows 10 or 11)
- **One Linux host** for the Elastic Stack:
  - Ubuntu Server 22.04 LTS (ARM64 if using Apple Silicon; x86_64 otherwise)
  - Recommended: ≥ 4 CPU cores, ≥ 8 GB RAM, ≥ 80 GB disk
- **One attack machine**: macOS or Linux host running a Kali Linux VM
- All machines on the same home LAN (192.168.0.0/24)
- ISO images:
  - [Windows Server Evaluation](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2022)
  - [Windows 10/11 Enterprise Evaluation](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-10-enterprise)
  - [Ubuntu Server 22.04 LTS](https://ubuntu.com/download/server) (ARM64 or AMD64)
  - [Kali Linux](https://www.kali.org/get-kali/)

---

## 2. Set Up DC01 — Windows Domain Controller

### 2.1 Assign a Static IP

Before promoting to a DC, assign a static IP so DNS will be stable:

```powershell
# In PowerShell (run as Administrator)
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.0.10 -PrefixLength 24 -DefaultGateway 192.168.0.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 127.0.0.1
```

### 2.2 Install Active Directory Domain Services

```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
```

### 2.3 Promote to Domain Controller

```powershell
Install-ADDSForest `
  -DomainName "corp.lab" `
  -DomainNetbiosName "CORP" `
  -InstallDns `
  -Force
```

The server will reboot. After reboot, log in as `CORP\Administrator`.

### 2.4 Create Domain Users

```powershell
# Create a standard domain user
New-ADUser -Name "John Smith" -SamAccountName "jsmith" `
  -UserPrincipalName "jsmith@corp.lab" `
  -AccountPassword (ConvertTo-SecureString "Password123!" -AsPlainText -Force) `
  -Enabled $true

# Create a service account with an SPN (for Kerberoasting lab)
New-ADUser -Name "svc-sql" -SamAccountName "svc-sql" `
  -AccountPassword (ConvertTo-SecureString "ServicePass1!" -AsPlainText -Force) `
  -Enabled $true

Set-ADUser -Identity "svc-sql" -ServicePrincipalNames @{Add="MSSQLSvc/dc01.corp.lab:1433"}
```

### 2.5 Verify Domain Health

```powershell
# Check DC replication and services
dcdiag /test:replications /test:services /test:netlogons /q
netlogon status

# Confirm SPN registered
setspn -L svc-sql
```

---

## 3. Install Sysmon on DC01

Sysmon provides rich process, network, and registry telemetry beyond what standard Windows Event Logs capture.

### 3.1 Download Sysmon

Download [Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) from Microsoft Sysinternals, or use PowerShell:

```powershell
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "C:\Tools\Sysmon.zip"
Expand-Archive -Path "C:\Tools\Sysmon.zip" -DestinationPath "C:\Tools\Sysmon"
```

### 3.2 Download a Configuration File

Use the community-maintained [SwiftOnSecurity Sysmon config](https://github.com/SwiftOnSecurity/sysmon-config) as a solid baseline:

```powershell
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" `
  -OutFile "C:\Tools\Sysmon\sysmonconfig.xml"
```

### 3.3 Install Sysmon

```powershell
cd C:\Tools\Sysmon
.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```

### 3.4 Verify Sysmon is Running

```powershell
Get-Service Sysmon64
# Open Event Viewer → Applications and Services Logs → Microsoft → Windows → Sysmon → Operational
```

---

## 4. Configure Windows Event Forwarding (WEF)

WEF allows WS01 to push its Windows events to DC01 so a single Elastic Agent on DC01 can collect logs from both machines.

### 4.1 Enable WinRM on DC01 (WEC Collector)

```powershell
# Run on DC01 as Administrator
winrm quickconfig -q
wecutil qc -q
```

### 4.2 Create a WEF Subscription on DC01

Open **Event Viewer** → **Subscriptions** → **Create Subscription**:

| Setting | Value |
|---------|-------|
| Subscription name | `WS01-ForwardedEvents` |
| Subscription type | **Source-initiated** |
| Destination log | **Forwarded Events** |
| Events to collect | Security, System, Application, Microsoft-Windows-Sysmon/Operational |

Or via command line:

```powershell
# Create subscription XML (save as C:\Tools\subscription.xml)
$xml = @'
<Subscription xmlns="http://schemas.microsoft.com/2006/03/windows/events/subscription">
  <SubscriptionId>WS01-ForwardedEvents</SubscriptionId>
  <SubscriptionType>SourceInitiated</SubscriptionType>
  <Description>Collect events from domain workstations</Description>
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
$xml | Out-File "C:\Tools\subscription.xml" -Encoding UTF8
wecutil cs "C:\Tools\subscription.xml"
```

### 4.3 Configure Group Policy to Enable WEF on Workstations

On DC01, open **Group Policy Management** and create/edit a GPO applied to the domain:

- **Computer Configuration → Policies → Administrative Templates → Windows Components → Event Forwarding**
  - *Configure the server address, refresh interval, and issuer certificate authority of a target Subscription Manager*
  - Value: `Server=http://DC01:5985/wsman/SubscriptionManager/WEC,Refresh=60`

- **Computer Configuration → Policies → Administrative Templates → Windows Components → Event Log Service → Security**
  - *Change log access* — add `NT AUTHORITY\NETWORK SERVICE` with read access

---

## 5. Set Up WS01 — Domain-Joined Workstation

### 5.1 Configure Network & DNS

Set DNS to DC01's static IP so the workstation can resolve `corp.lab`:

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.0.10
```

### 5.2 Join the Domain

```powershell
Add-Computer -DomainName "corp.lab" -Credential (Get-Credential) -Restart
```

Log in as `CORP\jsmith` (or another domain user) after reboot to verify domain join.

### 5.3 Verify Domain Join

```powershell
# Confirm domain membership
(Get-WmiObject Win32_ComputerSystem).Domain
# Expected output: corp.lab
```

---

## 6. Install Sysmon on WS01 & Configure WEF Client

### 6.1 Install Sysmon on WS01

Repeat [Section 3](#3-install-sysmon-on-dc01) on WS01 using the same Sysmon binary and config file.

### 6.2 Enable WinRM on WS01

The Group Policy from [Section 4.3](#43-configure-group-policy-to-enable-wef-on-workstations) should configure WinRM automatically. To verify manually:

```powershell
# Run on WS01
winrm quickconfig -q
gpupdate /force
```

### 6.3 Verify WEF is Working

On DC01, check the **Forwarded Events** log in Event Viewer. Events from WS01 should appear.

To generate a test event from WS01:

```cmd
eventcreate /T INFORMATION /ID 100 /L APPLICATION /D "WEF Test Event from WS01"
```

Verify this event appears in DC01's Forwarded Events within 60 seconds.

---

## 7. Deploy Elastic Stack on Ubuntu ARM64

This section covers deploying a self-hosted single-node Elastic Stack on Ubuntu Server (ARM64 for Apple Silicon Macs; swap `arm64` for `amd64` if using x86_64).

### 7.1 Install Java (if needed)

Elasticsearch bundles its own JDK, so no additional Java installation is required.

### 7.2 Add Elastic APT Repository

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/elastic-8.x.list

sudo apt-get update
```

### 7.3 Install Elasticsearch

```bash
sudo apt-get install -y elasticsearch

# Note the auto-generated elastic user password shown during install
# Save it — you'll need it for Kibana and Fleet setup
```

### 7.4 Configure Elasticsearch for Single-Node

```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```

Set:

```yaml
cluster.name: soc-lab
node.name: soc-node-1
network.host: 0.0.0.0
http.port: 9200
discovery.type: single-node
xpack.security.enabled: true
xpack.security.enrollment.enabled: true
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch

# Verify
curl -k -u elastic:<password> https://localhost:9200
```

### 7.5 Install Kibana

```bash
sudo apt-get install -y kibana
sudo nano /etc/kibana/kibana.yml
```

Set:

```yaml
server.port: 5601
server.host: "0.0.0.0"
elasticsearch.hosts: ["https://localhost:9200"]
elasticsearch.username: "kibana_system"
# Set kibana_system password (see below)
```

Generate a kibana_system password:

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system
```

Add to `kibana.yml`:

```yaml
elasticsearch.password: "<kibana_system_password>"
elasticsearch.ssl.verificationMode: none
```

```bash
sudo systemctl enable kibana
sudo systemctl start kibana
```

Access Kibana at `http://<ubuntu-siem-ip>:5601`.

---

## 8. Configure Fleet Server & Enroll Elastic Agent

### 8.1 Generate Fleet Server Enrollment Token

In Kibana: **Management → Fleet → Settings → Add Fleet Server**.

Follow the guided setup to:
1. Generate a Fleet Server service token
2. Install Elastic Agent on the Ubuntu host as the Fleet Server

```bash
curl -L -O https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.x.x-linux-arm64.tar.gz
tar xzvf elastic-agent-8.x.x-linux-arm64.tar.gz
cd elastic-agent-8.x.x-linux-arm64
sudo ./elastic-agent install \
  --fleet-server-es=https://localhost:9200 \
  --fleet-server-service-token=<service-token> \
  --fleet-server-policy=fleet-server-policy \
  --fleet-server-es-insecure
```

### 8.2 Create a Windows Agent Policy

In Kibana: **Management → Fleet → Agent policies → Create agent policy**

- Name: `Windows Endpoints`
- System integration: enabled

### 8.3 Enroll Elastic Agent on DC01

In Kibana, get the enrollment token for the Windows Endpoints policy, then on **DC01** (PowerShell as Administrator):

```powershell
# Download Elastic Agent for Windows
Invoke-WebRequest -Uri "https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.x.x-windows-x86_64.zip" `
  -OutFile "C:\Tools\elastic-agent.zip"
Expand-Archive -Path "C:\Tools\elastic-agent.zip" -DestinationPath "C:\Tools\"
cd "C:\Tools\elastic-agent-8.x.x-windows-x86_64"

# Install and enroll
.\elastic-agent.exe install `
  --url=https://<ubuntu-siem-ip>:8220 `
  --enrollment-token=<enrollment-token> `
  --insecure
```

Verify in Kibana: **Fleet → Agents** — DC01 should appear as **Healthy**.

---

## 9. Add Windows & Sysmon Integrations

### 9.1 Add Windows Integration

In Kibana: **Fleet → Agent Policies → Windows Endpoints → Add integration**

Search for **Windows** → add the integration:
- Enable: Security, System, Application event logs
- Enable: PowerShell logs (optional — see [Project 14](../projects/14-powershell-logging-tuning.md))

### 9.2 Add Sysmon Integration

In Kibana: **Fleet → Agent Policies → Windows Endpoints → Add integration**

Search for **Sysmon** (part of Windows integration or as custom) → add:
- Log path: `Microsoft-Windows-Sysmon/Operational`

The agent will automatically pick up the new configuration without restart.

---

## 10. Verify End-to-End Log Pipeline

### 10.1 Check Logs in Kibana Discover

In Kibana: **Discover** → set index pattern to `logs-*`

Filter: `event.module: "windows"` or `event.module: "sysmon"`

You should see real-time Windows events appearing from DC01.

### 10.2 Generate a Test Event on DC01

```powershell
eventcreate /T WARNING /ID 999 /L APPLICATION /D "Elastic SIEM Pipeline Test"
```

Search in Kibana Discover for `message: "Elastic SIEM Pipeline Test"` — should appear within 30 seconds.

### 10.3 Verify Forwarded Events from WS01

Generate an event on WS01:

```cmd
eventcreate /T INFORMATION /ID 100 /L APPLICATION /D "WEF Test from WS01"
```

This event should appear in DC01's Forwarded Events log, then be shipped by Elastic Agent to Kibana. Verify in Kibana Discover filtering by `host.name: WS01`.

---

## 11. Set Up Kali Linux Attack VM

### 11.1 Create Kali VM

On the Attack Mac, create a Kali Linux VM using your preferred hypervisor (UTM, VirtualBox, VMware Fusion). Ensure the VM is bridged to the home LAN so it gets a 192.168.0.x address.

### 11.2 Install Required Tools

```bash
# Update and install AD attack tools
sudo apt update && sudo apt upgrade -y
sudo apt install -y impacket-scripts crackmapexec hashcat bloodhound

# Verify Impacket Kerberoasting tool
impacket-GetUserSPNs --help
```

### 11.3 Verify Connectivity to DC01

```bash
# Verify reachability
ping 192.168.0.10   # DC01

# Verify AD connectivity (replace with DC01's IP and domain)
crackmapexec smb 192.168.0.10 -u jsmith -p 'Password123!' -d corp.lab
```

---

## Troubleshooting

| Issue | Resolution |
|-------|-----------|
| Elastic Agent shows Unhealthy | Check Fleet Server is reachable from DC01 on port 8220; verify enrollment token |
| No events in Kibana | Check agent status in Fleet; verify Windows/Sysmon integrations are applied to policy |
| WEF not forwarding events | Run `gpupdate /force` on WS01; verify WinRM service running; check WEF subscription status on DC01 |
| Kerberos errors on DC01 | Verify time sync (NTP) between DC01 and all domain members; max skew is 5 minutes |
| Sysmon not logging | Check `Get-Service Sysmon64` and review Event Viewer → Sysmon/Operational |
| Can't reach DC01 from Kali | Verify Kali VM is bridged (not NAT); check DC01 Windows Firewall allows ICMP/SMB |

---

## Next Steps

With the lab running, proceed to the [Projects](../projects/) to start performing and documenting security exercises:

1. [09 — Elastic SIEM Setup](../projects/09-elastic-siem-setup.md)
2. [11 — Kerberoasting Detection](../projects/11-kerberoasting-detection.md)
3. [12 — Custom Detection Rules](../projects/12-custom-detection-rules.md)

