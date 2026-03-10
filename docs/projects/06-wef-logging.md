# Project 06 — Centralized Logging with Windows Event Forwarding

**Skills:** Windows Event Forwarding (WEF), WEC, WinRM, Group Policy, Centralized Log Management

---

## Objective

Configure Windows Event Forwarding (WEF) so that Security and Sysmon logs from WS01 are automatically forwarded to DC01 — the Windows Event Collector (WEC). This consolidates endpoint logs into a single collection point before they are shipped to Elastic SIEM by the Elastic Agent on DC01.

---

## Architecture

```
WS01 (Sysmon + Windows Security Log)
  │
  │  Windows Event Forwarding (WinRM / HTTP)
  ▼
DC01 — Windows Event Collector (WEC)
  └── Forwarded Events log
        │
        │  Elastic Agent
        ▼
     Elastic SIEM (Kibana)
```

---

## Environment

| VM | Hostname | Role |
|----|----------|------|
| Windows Server | DC01 | Domain Controller + WEC (collector) |
| Windows Client | WS01 | WEF source (forwarding client) |

---

## Steps Completed

### 1. Configured DC01 as the Windows Event Collector

![WEF](../assets/DC01%20setup/15%20configured%20windows%20event%20collector.png)

On **DC01** (as Domain Admin):

```powershell
# Start and enable the Windows Event Collector service
wecutil qc /q

# Confirm the service is running
Get-Service Wecsvc
```

### 2. Allowed WinRM Through the Firewall on DC01

WEF uses WinRM (port 5985) for event transport:

```powershell
# Enable WinRM (already on for Domain environments, but confirm)
Enable-PSRemoting -Force

# Verify WinRM is listening
winrm enumerate winrm/config/listener
```

### 3. Created the WEF Subscription on DC01

![created subscription](../assets/DC01%20setup/16%20created%20windows%20event%20forwarding%20subscription.png)

A WEF subscription defines which events are collected and from which source machines. Created a **source-initiated subscription** (WS01 pushes to DC01).

In Event Viewer on DC01:
1. Opened **Event Viewer → Subscriptions**
2. Right-clicked → **Create Subscription**
3. Configured:
   - **Subscription name:** `Lab-WS01-Subscription`
   - **Destination log:** `Forwarded Events`
   - **Subscription type:** `Source computer initiated`
   - **Source computers:** Added WS01 (or the `Workstations` security group)
   - **Events to collect:**
     - Security log (all events)
     - `Microsoft-Windows-Sysmon/Operational` (all events)

Alternatively via command line, using a subscription XML:

```xml
<!-- C:\WEF\ws01-subscription.xml -->
<Subscription xmlns="http://schemas.microsoft.com/2006/03/windows/events/subscription">
  <SubscriptionId>Lab-WS01-Subscription</SubscriptionId>
  <SubscriptionType>SourceInitiated</SubscriptionType>
  <Description>Collect Security and Sysmon logs from WS01</Description>
  <Enabled>true</Enabled>
  <Uri>http://schemas.microsoft.com/wbem/wsman/1/windows/EventLog</Uri>
  <ConfigurationMode>MinLatency</ConfigurationMode>
  <Query>
    <![CDATA[
      <QueryList>
        <Query Id="0">
          <Select Path="Security">*</Select>
          <Select Path="Microsoft-Windows-Sysmon/Operational">*</Select>
        </Query>
      </QueryList>
    ]]>
  </Query>
  <ReadExistingEvents>false</ReadExistingEvents>
  <TransportName>http</TransportName>
  <ContentFormat>RenderedText</ContentFormat>
  <Locale Language="en-US"/>
  <LogFile>ForwardedEvents</LogFile>
  <AllowedSourceDomainComputers>O:NSG:BAD:P(A;;GA;;;DC)S:</AllowedSourceDomainComputers>
</Subscription>
```

```powershell
wecutil cs C:\WEF\ws01-subscription.xml
```

### 4. Configured WS01 as a Forwarding Client via GPO

On **DC01**, created a Group Policy Object to configure WS01 to forward events:

```
Group Policy Management → corp.lab → New GPO: "WEF-Client-Config"
Applied to: Workstations OU
```

**Settings configured in the GPO:**

| GPO Path | Setting | Value |
|----------|---------|-------|
| Computer Configuration → Policies → Administrative Templates → Windows Components → Event Forwarding | Configure the server address... | `Server=http://dc01.corp.lab:5985/wsman/SubscriptionManager/WEC,Refresh=60` |
| Computer Configuration → Policies → Windows Settings → Security Settings → System Services | Windows Remote Management (WinRM) | Automatic |
| Computer Configuration → Policies → Administrative Templates → Windows Components → Windows Remote Management (WinRM) → WinRM Client | Allow Basic authentication | Disabled |
| Computer Configuration → Policies → Administrative Templates → Windows Components → Windows Remote Management (WinRM) → WinRM Service | Allow remote server management through WinRM | `*` (all interfaces) |

Applied the GPO on WS01:

```powershell
gpupdate /force
```

### 5. Verified Forwarding Was Working

On **WS01**, confirmed the forwarding service was active:

```powershell
Get-Service WinRM
winrm get winrm/config/client
```

On **DC01**, checked the Forwarded Events log:

```powershell
Get-WinEvent -LogName "ForwardedEvents" -MaxEvents 20 |
  Select-Object TimeCreated, Id, MachineName, Message | Format-List
```

**Result:** Events from WS01 appeared in the `ForwardedEvents` log on DC01, including Sysmon Event ID 1 entries with `MachineName: WS01`.

### 6. Manual Test with `eventcreate`

Generated a synthetic test event from WS01 to confirm end-to-end forwarding:

```cmd
REM Run on WS01
eventcreate /T INFORMATION /ID 999 /L APPLICATION /SO WEFTest /D "WEF pipeline test event from WS01"
```

Waited ~60 seconds (subscription refresh interval), then on DC01:

```powershell
Get-WinEvent -LogName "ForwardedEvents" |
  Where-Object { $_.Id -eq 999 } |
  Select-Object TimeCreated, MachineName, Message
```

**Result:** Test event ID 999 from WS01 appeared in DC01's Forwarded Events log. ✅

![logs flowing](../assets/DC01%20setup/19%20after%20much%20troubleshooting%20we%20now%20have%20logs.png)

---

## Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| WS01 not forwarding — no events in DC01 Forwarded Events | WinRM service on WS01 was not started; ran `gpupdate /force` and manually started WinRM |
| Subscription showing "Error" status in Event Viewer | Added the WS01 computer account to the `Event Log Readers` group on DC01 |
| Sysmon events not appearing in forwarded events | Added `Microsoft-Windows-Sysmon/Operational` as a second `<Select>` element in the subscription XML; re-ran `wecutil cs` |

---

## Key Takeaways

- Source-initiated (push) subscriptions are simpler to manage in a domain environment — the collector doesn't need a list of every source machine; machines enroll themselves via GPO
- WEF requires WinRM but does not require Elastic Agent on WS01 — all telemetry flows through DC01's Elastic Agent, simplifying agent deployment
- Verifying with `eventcreate` provides a quick, reliable end-to-end smoke test before relying on real attack traffic for validation

---

## Next Steps

→ [Project 07 — SOC Infrastructure Deployment](07-soc-infrastructure.md)
