# Project 11 — Custom Detection Rules

**Status:** 🔜 Next Step  
**Skills:** Elastic Security, Detection Rules, KQL, EQL, Threshold Rules, MITRE ATT&CK, Alert Management

---

## Overview

This project builds custom detection rules in Elastic Security to automatically alert on Active Directory attack patterns observed in the lab. Two primary detections are implemented:

1. **Excessive Failed Logins (4625)** — Brute force / password spray detection
2. **Kerberoasting — RC4 Service Ticket Requests (4769)** — Kerberoasting detection

These rules demonstrate the full SOC detection lifecycle: identify attack patterns → write detection logic → tune threshold → test against simulated attacks → verify alerts fire → investigate.

### Goals

- Create a threshold-based rule detecting excessive Event ID 4625 (failed logons)
- Create a query-based rule detecting Kerberoasting via Event ID 4769 + RC4 encryption
- Configure alert severity, MITRE ATT&CK mappings, and notification actions
- Test rules against simulated attacks and verify alerts fire correctly
- Document the investigation workflow for each alert type

---

## Part 1: Navigate to Elastic Security Detection Rules

In Kibana:

1. **Security → Rules → Detection rules (SIEM)**
2. Click **Create new rule**

---

## Part 2: Rule 1 — Excessive Failed Logons (Brute Force / Password Spray)

![failed login rule](../assets/elastic/custom%20rule%20for%20failed%20logins.png)

### 2.1 Rule Type: Threshold

Threshold rules alert when the number of matching events exceeds a configurable count within a time window.

### 2.2 Rule Configuration

**Step 1: Define rule**

| Setting | Value |
|---------|-------|
| Rule type | Threshold |
| Index patterns | `logs-windows.*`, `logs-*` |
| Custom query | `event.code: "4625"` |
| Threshold field | `winlog.event_data.TargetUserName` |
| Threshold count | `5` |
| Time window | `5 minutes` (using rule schedule) |

The KQL query:

```kql
event.code: "4625"
```

**Step 2: About rule**

| Setting | Value |
|---------|-------|
| Name | `Excessive Failed Logon Attempts — Possible Brute Force` |
| Description | `Detects 5 or more failed logon attempts (Event ID 4625) for the same target account within a short window. Indicates possible brute force or password spraying against the domain.` |
| Severity | Medium |
| Risk score | 47 |
| Tags | `Windows`, `Active Directory`, `Credential Access`, `Brute Force` |

**Step 3: MITRE ATT&CK**

| Field | Value |
|-------|-------|
| Tactic | Credential Access |
| Technique | T1110 — Brute Force |
| Sub-technique | T1110.003 — Password Spraying |

**Step 4: Schedule**

- Runs every: `5 minutes`
- Additional look-back time: `1 minute`

**Step 5: Threshold**

```
Group by: winlog.event_data.TargetUserName
Threshold: 5 or more matches
```

This fires a separate alert for each targeted username that hits 5+ failures.

### 2.3 Test the Rule

![alerts](../assets/elastic/alerts%20coming%20in.png)

Simulate failed logins from Kali:

```bash
# Password spray — one wrong password against multiple users
crackmapexec smb 192.168.0.10 -u users.txt -p 'WrongPassword1' -d corp.lab

# Or brute force a single account
crackmapexec smb 192.168.0.10 -u jsmith -p /usr/share/wordlists/rockyou.txt -d corp.lab
```

In Kibana: **Security → Alerts** — the rule should fire within 5 minutes.

### 2.4 Investigate the Alert

When the alert fires, click it to open the **Alert Details** panel:

1. Check `winlog.event_data.TargetUserName` — which account is being targeted?
2. Check `winlog.event_data.IpAddress` — what source IP is attacking?
3. Check `winlog.event_data.LogonType` — Type 3 = network logon (SMB spray); Type 10 = remote interactive
4. Check `@timestamp` range — is this a sustained attack or a burst?
5. Click **Investigate in timeline** to open a full investigation view

### 2.5 Tuning Guidance

| Issue | Adjustment |
|-------|-----------|
| Too many false positives (legitimate lockouts) | Increase threshold to 10; add exclusion for known service accounts |
| Missing spray attacks (low-volume) | Decrease threshold to 3; group by source IP instead of target username |
| Alert storm during password resets | Add exception for helpdesk IP range |

---

## Part 3: Rule 2 — Kerberoasting (RC4 Service Ticket Requests)

### 3.1 Rule Type: Query (Custom Query)

A custom query rule fires an alert for each event matching the query within the rule's time window.

### 3.2 Rule Configuration

**Step 1: Define rule**

| Setting | Value |
|---------|-------|
| Rule type | Custom query |
| Index patterns | `logs-windows.*`, `logs-*` |

The KQL query:

```kql
event.code: "4769"
  and winlog.event_data.TicketEncryptionType: "0x17"
  and not winlog.event_data.ServiceName: "*$"
```

**Explanation:**
- `event.code: "4769"` — Kerberos TGS request events only
- `TicketEncryptionType: "0x17"` — RC4-HMAC encryption, which is easily crackable and abnormal in modern environments (AES should be used)
- `not ServiceName: "*$"` — excludes machine accounts (e.g., `DC01$`) which routinely use RC4; keeps focus on user service accounts

![custom rule](../assets/elastic/kerberoasting%20rule%20created.png)

**Step 2: About rule**

| Setting | Value |
|---------|-------|
| Name | `Kerberoasting — RC4 Service Ticket Requested` |
| Description | `Detects Kerberos Service Ticket (TGS) requests using RC4 encryption (0x17), a strong indicator of Kerberoasting. Attackers request RC4 tickets because they are faster to crack offline than AES-encrypted tickets.` |
| Severity | High |
| Risk score | 73 |
| Tags | `Windows`, `Active Directory`, `Kerberos`, `Credential Access`, `Kerberoasting` |

**Step 3: MITRE ATT&CK**

| Field | Value |
|-------|-------|
| Tactic | Credential Access |
| Technique | T1558 — Steal or Forge Kerberos Tickets |
| Sub-technique | T1558.003 — Kerberoasting |

**Step 4: Schedule**

- Runs every: `5 minutes`
- Additional look-back time: `1 minute`

### 3.3 Test the Rule

![testing rule](../assets/elastic/kerberoasting%20command%20on%20kali.png)

Run the Kerberoasting simulation from [Project 11](11-kerberoasting-detection.md):

```bash
impacket-GetUserSPNs corp.lab/jsmith:'Password123!' \
  -dc-ip 192.168.0.10 \
  -request \
  -outputfile /tmp/kerberoast.hashes
```

In Kibana: **Security → Alerts** — the rule should fire within 5 minutes.

![alerts firing](../assets/elastic/kerberoasting%20alerts%20coming%20in.png)

### 3.4 Investigate the Alert

When the alert fires:

1. Identify `winlog.event_data.TargetUserName` — which domain user made the request (the attacker's account)?
2. Identify `winlog.event_data.ServiceName` — which service account was targeted?
3. Identify `winlog.event_data.IpAddress` — source IP (should be Kali)
4. Identify `winlog.event_data.IpPort` — ephemeral source port
5. Check `@timestamp` — time of the attack
6. Open investigation timeline and search for related 4768 events (TGT issuance) to confirm the full auth sequence
7. Escalate: the `ServiceName` account's password should be treated as compromised and rotated immediately

### 3.5 EQL Version (Advanced)

For more sophisticated detection using Elastic's Event Query Language (EQL), which can correlate sequences:

```
sequence by host.id with maxspan=1m
  [authentication where event.code == "4769" and
   winlog.event_data.TicketEncryptionType == "0x17" and
   not winlog.event_data.ServiceName like~ "*$"]
  [authentication where event.code == "4769" and
   winlog.event_data.TicketEncryptionType == "0x17" and
   not winlog.event_data.ServiceName like~ "*$"]
```

This EQL rule fires when 2+ RC4 TGS requests occur from the same host within 1 minute — useful for catching bulk SPN enumeration.

---

## Part 4: Rule 3 — Suspicious New Service Installation

### 4.1 Overview

Attackers often install malicious services for persistence or lateral movement. Event ID 7045 logs new service creation.

### 4.2 Rule Configuration

**Custom query:**

```kql
event.code: "7045"
  and not winlog.event_data.ServiceFileName: "C:\\Windows\\*"
  and not winlog.event_data.ServiceFileName: "C:\\Program Files\\*"
  and not winlog.event_data.ServiceFileName: "C:\\Program Files (x86)\\*"
```

**About:**

| Setting | Value |
|---------|-------|
| Name | `Suspicious Service Installation — Non-Standard Path` |
| Severity | High |
| Risk score | 67 |
| MITRE | T1543.003 — Create or Modify System Process: Windows Service |

---

## Part 5: Manage and Review Alerts

### 5.1 View All Active Alerts

**Security → Alerts**

Filter by:
- Rule name
- Severity (High/Critical first)
- Status (Open / Acknowledged / Closed)

### 5.2 Alert Workflow

| Status | Meaning | Action |
|--------|---------|--------|
| Open | New, unreviewed | Triage: determine if true positive or false positive |
| Acknowledged | Under investigation | Analyst assigned; building investigation timeline |
| Closed | Resolved | Document: true positive → incident created; false positive → rule tuned |

### 5.3 Bulk Actions

For lab-generated test alerts:
1. Select all alerts from the simulation
2. **Actions → Close** (or set to Closed with reason: "Test — lab simulation")

---

## Part 6: Review Pre-Built Elastic Rules

![prebuilt alerts](../assets/elastic/installing%20pre%20built%20rules.png)

Elastic ships 600+ pre-built detection rules mapped to MITRE ATT&CK. Enable relevant ones for the lab:

**Security → Rules → Detection rules → Load Elastic prebuilt rules**

Recommended rules to enable for this lab:

| Rule Name | Technique |
|-----------|-----------|
| Kerberos Traffic from Unusual Process | T1558 |
| Windows Failed Logon Events — Detailed | T1110 |
| Mimikatz Command Line | T1003 |
| PowerShell Encoded Command Execution | T1059.001 |
| PsExec Network Connection | T1021.002 |
| New Service Created via PowerShell | T1543.003 |
| Suspicious SMB Activity | T1021.002 |

---

## Part 7: Alert Summary from Lab Exercises

| Rule | Trigger | Alerts Fired | True Positives | False Positives |
|------|---------|-------------|----------------|-----------------|
| Excessive Failed Logons | CME password spray | 3 alerts (by username) | 3 | 0 |
| Kerberoasting RC4 | Impacket GetUserSPNs | 1 alert (per SPN) | 1 | 0 |
| Suspicious Service Install | PsExec (lateral movement) | 1 alert | 1 | 0 |

---

## Key Observations

1. **Threshold rules are powerful for spray detection** — grouping by target username catches spray immediately; grouping by source IP catches single-account brute force
2. **RC4 filter is high-fidelity** — in modern Windows environments (2012+), AES is the default; RC4 TGS requests stand out clearly
3. **EQL enables sequence correlation** — multiple events can be chained for higher-confidence detections with fewer false positives
4. **Pre-built rules save time** — Elastic's rule library provides immediate coverage; custom rules fill gaps specific to your environment

---

## Next Steps

- [Project 13 — Lateral Movement Lab](13-lateral-movement-lab.md) — Simulate lateral movement to WS01 and detect with these rules
- [Project 14 — PowerShell Logging Tuning](14-powershell-logging-tuning.md) — Enable Script Block Logging to detect encoded PowerShell commands
