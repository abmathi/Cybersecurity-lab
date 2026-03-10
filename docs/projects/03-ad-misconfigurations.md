# Project 03 — Active Directory Misconfigurations

**Skills:** Active Directory Security, Group Policy, Attack Surface Configuration, Kerberos, LLMNR

---

## Objective

Introduce intentional misconfigurations into the `corp.lab` Active Directory environment to create realistic attack surfaces for later detection and offensive exercises. Every setting documented here mirrors weaknesses commonly found in real enterprise environments.

---

## Environment

| VM | Hostname | Role |
|----|----------|------|
| Windows Server VM | DC01 | Domain Controller (`corp.lab`) |

---

## Misconfigurations Introduced

### 1. Weak Password Policy

A permissive domain password policy allows short, simple passwords — enabling password spraying and easier offline cracking of captured hashes.

```powershell
# Set a weak domain-wide password policy (run on DC01 as Domain Admin)
Set-ADDefaultDomainPasswordPolicy `
  -Identity "corp.lab" `
  -MinPasswordLength 4 `
  -PasswordHistoryCount 0 `
  -MaxPasswordAge (New-TimeSpan -Days 0) `
  -MinPasswordAge (New-TimeSpan -Days 0) `
  -ComplexityEnabled $false `
  -ReversibleEncryptionEnabled $false
```

**Attack surface enabled:** password spraying with simple guesses (e.g., `Spring2024`, `Password1`); faster offline cracking of NTLM hashes.

---

### 2. Kerberoastable Service Accounts

Service accounts with registered SPNs (Service Principal Names) can have their Kerberos TGS tickets requested by any authenticated user and cracked offline.

```powershell
# Register SPNs on the service accounts created in Project 02
Set-ADUser -Identity "svc-sql" `
  -ServicePrincipalNames @{Add="MSSQLSvc/dc01.corp.lab:1433"}

Set-ADUser -Identity "svc-http" `
  -ServicePrincipalNames @{Add="HTTP/dc01.corp.lab:80"}

# Confirm SPNs are registered
setspn -L svc-sql
setspn -L svc-http
```

**Attack surface enabled:** Kerberoasting with `GetUserSPNs.py` (Impacket) from Kali; cracking the TGS tickets offline with Hashcat.

---

### 3. LLMNR Enabled (Link-Local Multicast Name Resolution)

LLMNR is enabled by default in Windows and responds to name resolution queries that DNS cannot resolve. An attacker on the same network segment can respond to these queries to capture NTLMv2 credential hashes using Responder.

LLMNR was left **enabled** (no GPO disable applied) to support the Responder/NTLMv2 capture lab.

**Verification — confirmed LLMNR is on (default state):**

```powershell
# Check that no GPO disables LLMNR
Get-GPO -All | Get-GPOReport -ReportType XML | Select-String "LLMNR"
# No results = LLMNR GPO not set = LLMNR active
```

**Attack surface enabled:** NTLMv2 hash capture with Responder (`responder -I eth0`); relay attacks with NTLMRelayx.

---

### 4. Anonymous Access / Null Session Enumeration

Allowed unauthenticated LDAP and SMB enumeration, common in older or poorly hardened environments.

```powershell
# Allow null session enumeration of domain users via registry
Set-ItemProperty `
  -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" `
  -Name "RestrictAnonymous" -Value 0

Set-ItemProperty `
  -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" `
  -Name "RestrictAnonymousSAM" -Value 0
```

**Attack surface enabled:** unauthenticated user enumeration with `enum4linux`, `rpcclient`, and LDAP queries from Kali.

---

### 5. Windows Firewall Disabled

The Windows Firewall on DC01 was disabled to simplify VM-to-VM connectivity and avoid interference with attack traffic during lab exercises.

```powershell
# Disable all firewall profiles
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False

# Verify
Get-NetFirewallProfile | Select-Object Name, Enabled
```

**Attack surface enabled:** unrestricted port access; RDP, SMB, WinRM, Kerberos all accessible from the attacker VM without needing firewall exceptions.

---

## Summary of Attack Surfaces

| Misconfiguration | Attack Technique | Event IDs Generated |
|-----------------|-----------------|---------------------|
| Weak password policy | Password spraying | 4625 (failed logon), 4624 (success) |
| Kerberoastable SPNs | Kerberoasting | 4769 (TGS request, RC4 encryption) |
| LLMNR enabled | NTLMv2 hash capture (Responder) | 4625, network traffic |
| Anonymous access | User enumeration (enum4linux, LDAP) | 4624 (anonymous logon Type 3) |
| Firewall disabled | Unrestricted port access | — |

---

## Key Takeaways

- All five misconfigurations reflect settings commonly found during real-world Active Directory assessments
- Keeping track of exactly what was changed (and why) allows for precise detection tuning: each misconfiguration maps to specific event IDs and detection logic
- The firewall-off setting is a lab-only shortcut; in production, firewall rules should always be maintained and logged

---

## Next Steps

→ [Project 04 — Workstation Deployment](04-workstation-deployment.md)
