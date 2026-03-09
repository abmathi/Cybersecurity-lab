# Project 03 — Network Scanning & Enumeration

**Skills:** Network Reconnaissance, Port Scanning, Vulnerability Assessment, Nmap, Nessus

---

## Objective

Perform systematic network reconnaissance and vulnerability scanning against all corporate segment targets. Document findings in a format similar to a real penetration testing engagement report.

---

## Environment

| VM | IP | Role |
|----|----|------|
| kali | 10.10.10.10 | Attacker |
| winserver2019 | 10.20.20.10 | Windows DC |
| win10-client1 | 10.20.20.20 | Windows workstation |
| win10-client2 | 10.20.20.21 | Windows workstation |
| metasploitable3 | 10.20.20.30 | Vulnerable Linux target |
| dvwa | 10.20.20.40 | Web application target |

---

## Phase 1 — Host Discovery

```bash
# ICMP sweep (fast, but may miss hosts with ICMP blocked)
nmap -sn 10.20.20.0/24

# ARP scan (more reliable on local network)
sudo arp-scan --interface=eth0 10.20.20.0/24
```

**Discovered hosts:**

| IP | Status | MAC Vendor |
|----|--------|------------|
| 10.20.20.1 | Up | Netgate (pfSense) |
| 10.20.20.10 | Up | VMware (Windows DC) |
| 10.20.20.20 | Up | VMware (Win10) |
| 10.20.20.21 | Up | VMware (Win10) |
| 10.20.20.30 | Up | VMware (Metasploitable3) |
| 10.20.20.40 | Up | VMware (DVWA) |

---

## Phase 2 — Port Scanning

### 2.1 Quick Scan (Top 1000 Ports)

```bash
nmap -sV --open -T4 10.20.20.0/24 -oG quick_scan.gnmap
```

### 2.2 Full TCP Port Scan — Metasploitable 3

```bash
nmap -sV -sC -p- -T4 10.20.20.30 -oN metasploitable3_full.txt
```

**Results — Metasploitable 3 (10.20.20.30):**

| Port | Service | Version | Notes |
|------|---------|---------|-------|
| 21/tcp | FTP | ProFTPD 1.3.5 | Vulnerable to CVE-2015-3306 (mod_copy) |
| 22/tcp | SSH | OpenSSH 6.6.1 | |
| 80/tcp | HTTP | Apache 2.4.7 | Multiple web apps |
| 445/tcp | SMB | Samba 3.x | MS17-010 EternalBlue |
| 631/tcp | IPP | CUPS 1.7 | |
| 3306/tcp | MySQL | MySQL 5.5 | No root password |
| 6667/tcp | IRC | UnrealIRCd | Backdoor CVE-2010-2075 |
| 8080/tcp | HTTP-Proxy | Apache Tomcat 8 | Manager accessible |
| 8181/tcp | HTTP | WEBrick 1.3.1 | Ruby WEBrick |

### 2.3 Full TCP Port Scan — Windows DC

```bash
nmap -sV -sC -p- -T4 10.20.20.10 -oN dc_full.txt
```

**Results — Windows Server 2019 DC (10.20.20.10):**

| Port | Service | Notes |
|------|---------|-------|
| 53/tcp | DNS | Active Directory DNS |
| 88/tcp | Kerberos | AD authentication |
| 135/tcp | MSRPC | |
| 139/tcp | NetBIOS | |
| 389/tcp | LDAP | Active Directory LDAP |
| 445/tcp | SMB | |
| 464/tcp | Kpasswd | Kerberos password change |
| 593/tcp | MSRPC | |
| 636/tcp | LDAPS | AD LDAP over SSL |
| 3268/tcp | LDAP | Global Catalog |
| 3389/tcp | RDP | Remote Desktop |

---

## Phase 3 — Service Enumeration

### 3.1 SMB Enumeration

```bash
# Check for EternalBlue (MS17-010)
nmap -p 445 --script smb-vuln-ms17-010 10.20.20.30

# Enumerate shares
smbclient -L //10.20.20.30 -N
enum4linux -a 10.20.20.30
```

**Finding:** Metasploitable 3 is vulnerable to MS17-010 (EternalBlue).

### 3.2 HTTP Enumeration

```bash
# Discover web directories
gobuster dir -u http://10.20.20.30 -w /usr/share/wordlists/dirb/common.txt

# Discover subdomains/vhosts
nikto -h http://10.20.20.30
```

**Discovered paths on Metasploitable 3:**
- `/phpmyadmin` — phpMyAdmin admin panel (unauthenticated access)
- `/wordpress` — WordPress installation (outdated version)
- `/mutillidae` — Mutillidae OWASP training app
- `/dvwa` — DVWA instance

### 3.3 FTP Anonymous Login Check

```bash
nmap -p 21 --script ftp-anon 10.20.20.30
# Result: Anonymous FTP allowed — read/write access
```

---

## Phase 4 — Vulnerability Scanning with Nessus

Ran a credentialed Nessus Essentials scan against `10.20.20.0/24`.

**Summary of Critical/High Findings:**

| Host | CVE | Severity | Description |
|------|-----|----------|-------------|
| 10.20.20.30 | CVE-2017-0144 | Critical | MS17-010 EternalBlue — SMB RCE |
| 10.20.20.30 | CVE-2015-3306 | Critical | ProFTPD mod_copy arbitrary file copy |
| 10.20.20.30 | CVE-2010-2075 | Critical | UnrealIRCd backdoor RCE |
| 10.20.20.30 | CVE-2009-4484 | High | MySQL auth bypass |
| 10.20.20.40 | CVE-2019-9978 | Medium | DVWA — stored XSS |
| 10.20.20.10 | — | Info | Kerberos without strong encryption |

---

## Phase 5 — OS Detection

```bash
sudo nmap -O 10.20.20.0/24 --osscan-guess
```

| IP | Detected OS |
|----|------------|
| 10.20.20.10 | Windows Server 2019 |
| 10.20.20.20 | Windows 10 |
| 10.20.20.30 | Linux 3.x (Ubuntu 14.04) |
| 10.20.20.40 | Linux 5.x (Ubuntu 20.04) |

---

## Findings Summary

| Priority | Host | Finding |
|----------|------|---------|
| Critical | 10.20.20.30 | EternalBlue (MS17-010) — unauthenticated RCE over SMB |
| Critical | 10.20.20.30 | UnrealIRCd backdoor — direct shell via IRC port |
| Critical | 10.20.20.30 | ProFTPD mod_copy — arbitrary file read/write as root |
| High | 10.20.20.30 | MySQL running without root password |
| High | 10.20.20.30 | phpMyAdmin exposed without authentication |
| Medium | 10.20.20.40 | DVWA accessible (multiple web vulnerabilities) |

---

## Key Takeaways

- Automated scanning (Nessus) identified the same critical vulnerabilities as manual enumeration, but manual steps revealed additional context (anonymous FTP, exposed admin panels)
- Combining Nmap, enum4linux, Nikto, and Nessus provides comprehensive coverage
- Even a quick scan would flag this network as critically exposed — demonstrates the importance of regular vulnerability assessments

---

## Next Steps

→ [Project 04 — Exploitation with Metasploit](04-exploitation-metasploit.md)
