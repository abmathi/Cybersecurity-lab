# Project 02 — Active Directory Domain Setup

**Skills:** Windows Server, Active Directory Domain Services, DNS, PowerShell, Organisational Units

---

## Objective

Create the Windows Server VM that will become the domain controller, promote it to a DC for the `corp.lab` domain, and populate Active Directory with realistic Organisational Units, users, and service accounts — forming the enterprise environment that all subsequent exercises target.

---

## Environment

| VM | Hostname | Role | IP |
|----|----------|------|----|
| Windows Server VM | DC01 | Domain Controller | 192.168.0.10 (static) |

**Domain:** `corp.lab` | **NetBIOS:** `CORP`

---

## Steps Completed

### 1. Created the Windows Server VM in VirtualBox

- Downloaded the Windows Server evaluation ISO from [Microsoft Evaluation Center](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2022)
- Created a new VM in VirtualBox:
  - Type: **Microsoft Windows**, Version: **Windows 2019 (64-bit)**
  - RAM: 4 GB | CPU: 2 | Disk: 60 GB (dynamically allocated)
  - Network adapter: **Bridged Adapter** (same home LAN as Kali)
- Installed Windows Server **without a product key** (evaluation mode)
- Selected **Desktop Experience** to get a full GUI

### 2. Renamed the Server to DC01

![Renaming Server](../assets/DC01%20setup/1%20naming%20server.png)

After Windows installed and the first login was complete:

```powershell
Rename-Computer -NewName "DC01" -Restart
```

### 3. Configured a Static IP

After reboot, set a static IP so DNS remains stable when the system becomes a DC:

```powershell
# Run as Administrator
New-NetIPAddress `
  -InterfaceAlias "Ethernet" `
  -IPAddress 192.168.0.10 `
  -PrefixLength 24 `
  -DefaultGateway 192.168.0.1

# Point DNS to itself (required before DC promotion)
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 127.0.0.1
```

Verified the static IP was applied:

```powershell
ipconfig /all
```

### 4. Rebooted the Server

```powershell
Restart-Computer
```

### 5. Installed Active Directory Domain Services

After logging back in as local Administrator:

```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
```

### 6. Promoted the Server to Domain Controller

Created a new forest with the domain name `corp.lab`:

```powershell
Install-ADDSForest `
  -DomainName "corp.lab" `
  -DomainNetbiosName "CORP" `
  -InstallDns `
  -Force
```

The server rebooted automatically. After reboot, logged in as `CORP\Administrator`.

### 7. Created Organisational Units

```powershell
New-ADOrganizationalUnit -Name "Corp Users"     -Path "DC=corp,DC=lab"
New-ADOrganizationalUnit -Name "Corp Admins"    -Path "DC=corp,DC=lab"
New-ADOrganizationalUnit -Name "Service Accounts" -Path "DC=corp,DC=lab"
New-ADOrganizationalUnit -Name "Workstations"   -Path "DC=corp,DC=lab"
```

### 8. Created Domain Users

```powershell
# Standard domain users
New-ADUser -Name "John Smith"   -SamAccountName "jsmith" `
  -UserPrincipalName "jsmith@corp.lab" `
  -Path "OU=Corp Users,DC=corp,DC=lab" `
  -AccountPassword (ConvertTo-SecureString "Password123!" -AsPlainText -Force) `
  -Enabled $true

New-ADUser -Name "Mike Brown"   -SamAccountName "mbrown" `
  -UserPrincipalName "mbrown@corp.lab" `
  -Path "OU=Corp Users,DC=corp,DC=lab" `
  -AccountPassword (ConvertTo-SecureString "Password123!" -AsPlainText -Force) `
  -Enabled $true

New-ADUser -Name "Sarah Jones"  -SamAccountName "sjones" `
  -UserPrincipalName "sjones@corp.lab" `
  -Path "OU=Corp Users,DC=corp,DC=lab" `
  -AccountPassword (ConvertTo-SecureString "Password123!" -AsPlainText -Force) `
  -Enabled $true

# Privileged admin account
New-ADUser -Name "AD Admin"     -SamAccountName "aadmin" `
  -UserPrincipalName "aadmin@corp.lab" `
  -Path "OU=Corp Admins,DC=corp,DC=lab" `
  -AccountPassword (ConvertTo-SecureString "Admin@1234!" -AsPlainText -Force) `
  -Enabled $true

Add-ADGroupMember -Identity "Domain Admins" -Members "aadmin"
```

### 9. Created Service Accounts

```powershell
# SQL service account (used as a Kerberoasting target in later labs)
New-ADUser -Name "svc-sql" -SamAccountName "svc-sql" `
  -UserPrincipalName "svc-sql@corp.lab" `
  -Path "OU=Service Accounts,DC=corp,DC=lab" `
  -AccountPassword (ConvertTo-SecureString "Sqlservice1!" -AsPlainText -Force) `
  -Enabled $true

# HTTP service account
New-ADUser -Name "svc-http" -SamAccountName "svc-http" `
  -UserPrincipalName "svc-http@corp.lab" `
  -Path "OU=Service Accounts,DC=corp,DC=lab" `
  -AccountPassword (ConvertTo-SecureString "Httpservice1!" -AsPlainText -Force) `
  -Enabled $true
```

### 10. Verified Domain Health

```powershell
# Check domain services and replication health
dcdiag /test:services /test:replications /q

# Confirm users were created
Get-ADUser -Filter * | Select-Object Name, SamAccountName, Enabled | Format-Table
```

---

## Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| Windows Server didn't see the correct NIC alias for the `New-NetIPAddress` command | Ran `Get-NetAdapter` first to confirm the interface alias name (was `Ethernet0` not `Ethernet`) |
| DC promotion failed — DNS prerequisite check warning | Set DNS server address to `127.0.0.1` before running `Install-ADDSForest`, then promotion succeeded |
| Users not appearing in correct OUs after creation | Confirmed OU paths with `Get-ADOrganizationalUnit -Filter *` before running `New-ADUser` |

---

## Key Takeaways

- Assigning a static IP and pointing DNS to itself before DC promotion avoids common deployment failures
- Organisational Units (OUs) provide a realistic structure for applying Group Policy and scoping permissions in later exercises
- Creating multiple user types (standard users, privileged admins, service accounts) mirrors a real enterprise and enables a broader set of attack simulations

---

## Next Steps

→ [Project 03 — Active Directory Misconfigurations](03-ad-misconfigurations.md)

