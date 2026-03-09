# Project 07 — Intrusion Detection with Security Onion

**Skills:** IDS/IPS, Suricata, Zeek, Network Security Monitoring, Rule Writing, Alert Triage

---

## Objective

Configure Security Onion to monitor all corporate segment traffic, tune Suricata detection rules, analyze alerts generated during lab attack exercises, and write a custom Suricata rule to detect a specific attack pattern.

---

## Environment

| Component | IP | Details |
|-----------|-----|---------|
| Security Onion | 10.30.30.10 | Manager + sensor (all-in-one) |
| Monitoring NIC | — | Span port on VLAN 20 (no IP) |

---

## Part 1 — Security Onion Configuration

### 1.1 Initial Setup

Security Onion was installed in **Standalone** mode — a single node acting as both the sensor and the manager.

During the setup wizard:
- Network interfaces:
  - `ens18` → management IP (`10.30.30.10/24`)
  - `ens19` → sniffing interface (no IP, VLAN 20 span port)
- Sensor type: `Import` initially for testing, then `Production`
- Enabled: Suricata (IDS), Zeek (NSM), Strelka (file analysis)

### 1.2 Accessing the Security Onion Console (SOC)

```bash
# Access SOC at https://10.30.30.10
# Default admin user created during setup
```

The SOC provides:
- **Alerts** — Suricata IDS alerts
- **Dashboards** — Zeek-based traffic dashboards
- **PCAP** — packet capture download for any session
- **Hunt** — threat hunting interface over Zeek logs

---

## Part 2 — Analyzing Attack Traffic

The following attacks from previous projects were re-run while monitoring Security Onion for detections.

### 2.1 Nmap Port Scan Detection

**Attacker command:**
```bash
nmap -sV -sC -p- 10.20.20.30
```

**Security Onion alerts triggered:**
- `ET SCAN Nmap Scripting Engine User-Agent Detected`
- `ET SCAN Potential SSH Scan`
- `ET SCAN NMAP -f -sV Probe`

**Zeek observations:**
- `conn.log` shows hundreds of short-lived connections from `10.10.10.10`
- Multiple connection states: `S0` (SYN sent, no reply) and `SF` (normal close)

### 2.2 EternalBlue (MS17-010) Exploitation

**Attacker command:**
```bash
# Metasploit: use exploit/windows/smb/ms17_010_eternalblue
```

**Security Onion alerts triggered:**
- `ET EXPLOIT MS17-010 EternalBlue SMB Remote Code Execution Attempt`
- `ETPRO EXPLOIT MS17-010 EternalBlue RCE — Trans2 SESSION_SETUP`
- `GPL NETBIOS SMB-DS IPC$ share access`

**Zeek observations:**
- `smb_files.log` — unusual file operations after exploit
- `conn.log` — large data transfer on port 445 following initial connection

### 2.3 Meterpreter Reverse Shell

**Security Onion alerts triggered:**
- `ET MALWARE Meterpreter or Other Reverse Shell Payload`
- `ET POLICY Suspicious inbound to mySQL port 3306` (Meterpreter staging)

**Zeek observations:**
- `conn.log` — long-duration connection from `10.20.20.30` to `10.10.10.10` on a high port (beacon pattern)
- Consistent data transfer interval — classic C2 beaconing

### 2.4 Kerberoasting

Kerberoasting uses legitimate Kerberos protocol calls, so network-level detection is harder. However:

**Zeek observations:**
- `kerberos.log` — multiple TGS requests with `rc4-hmac` encryption type from same client in short window

**Custom correlation:** Created a hunt query in SOC:
```
event_type:kerberos AND client:10.10.10.10 AND request_type:TGS AND cipher:rc4-hmac
```

---

## Part 3 — Suricata Rule Writing

### 3.1 Rule Syntax Overview

```
action proto src_ip src_port direction dst_ip dst_port (options)
```

Example:
```
alert tcp any any -> any 4444 (msg:"Possible Reverse Shell on Port 4444"; \
  flow:established; classtype:trojan-activity; sid:9000001; rev:1;)
```

### 3.2 Custom Rule — Detect Nmap Default User-Agent

```suricata
alert http any any -> $HOME_NET any (
  msg:"CUSTOM SCAN Nmap HTTP User-Agent Detected";
  flow:established,to_server;
  http.header;
  content:"User-Agent|3a 20|Mozilla/5.0 (compatible|3b| Nmap Scripting Engine";
  classtype:network-scan;
  sid:9000002;
  rev:1;
)
```

### 3.3 Custom Rule — Detect DVWA SQL Injection Attempt

```suricata
alert http $EXTERNAL_NET any -> $HTTP_SERVERS any (
  msg:"CUSTOM WEB SQL Injection Attempt in URI";
  flow:established,to_server;
  http.uri;
  content:"UNION"; nocase;
  content:"SELECT"; nocase; distance:0;
  pcre:"/UNION[\s\+]+SELECT/Ui";
  classtype:web-application-attack;
  sid:9000003;
  rev:1;
)
```

### 3.4 Custom Rule — Meterpreter Reverse Shell (Port 4444)

```suricata
alert tcp $HOME_NET any -> any 4444 (
  msg:"CUSTOM MALWARE Potential Meterpreter Reverse Shell Port 4444";
  flow:established,to_server;
  dsize:>100;
  flags:PA;
  classtype:trojan-activity;
  sid:9000004;
  rev:1;
)
```

### Deploying Custom Rules

```bash
# On Security Onion — add rules to local rules file
sudo nano /opt/so/rules/local.rules
# Paste rules above

# Reload Suricata
sudo so-rule-update
```

---

## Part 4 — Alert Triage Workflow

For each alert, a consistent triage process was followed:

1. **Review alert details** — rule triggered, source/dest IP, timestamp
2. **Examine PCAP** — download full packet capture for the triggering session
3. **Review Zeek logs** — correlate with `conn.log`, `dns.log`, `http.log`, `smb_files.log`
4. **Classify** — True Positive, False Positive, or Needs Investigation
5. **Escalate/Document** — document findings in a case note

**Sample triage for EternalBlue alert:**

| Field | Value |
|-------|-------|
| Alert | ET EXPLOIT MS17-010 EternalBlue |
| Source | 10.10.10.10 (Kali) |
| Destination | 10.20.20.30 (Metasploitable3) |
| Port | 445/TCP |
| Classification | True Positive |
| Action | Confirmed RCE; documented in incident report |

---

## Part 5 — Zeek Log Analysis

Zeek generates rich structured logs even when no Suricata signatures fire. Key log types used:

| Log File | Contents | Use Case |
|---------|----------|---------|
| `conn.log` | All network connections with duration, bytes | Baseline, C2 detection |
| `dns.log` | All DNS queries and responses | DNS tunneling, C2 domain detection |
| `http.log` | HTTP requests with URI, user-agent, status | Web attack detection |
| `ssl.log` | TLS/SSL certificate info | Malicious cert detection |
| `files.log` | Files transferred over the network | Malware delivery detection |
| `kerberos.log` | Kerberos authentication events | Kerberoasting detection |
| `smb_files.log` | SMB file operations | Lateral movement detection |

**Example — hunting for long-duration connections (C2 beaconing):**

In Security Onion Hunt interface:
```
event_type:conn AND duration:>300 AND NOT resp_p:443
| groupby id.orig_h id.resp_h
```

---

## Key Takeaways

- Security Onion out-of-the-box detected all major attacks with no custom rule tuning required for known exploits
- Zeek logs provide deep network visibility even when IDS signatures don't fire
- Custom Suricata rules allow detection of lab-specific attack patterns that commercial feeds don't cover
- Alert triage is an essential skill — blindly escalating every alert leads to fatigue; structured triage builds analyst efficiency
- PCAP access is invaluable for alert validation — being able to see exactly what happened at the packet level removes ambiguity

---

## Next Steps

→ [Project 08 — Incident Response Simulation](08-incident-response.md)
