# Project 04 — Workstation Deployment

**Skills:** Windows Client, Domain Join, VirtualBox, Endpoint Configuration, Active Directory

---

## Objective

Create a domain-joined Windows workstation (WS01) that simulates an enterprise endpoint — a realistic lateral movement target used in later attack detection exercises.

---

## Environment

| VM | Hostname | Role | IP |
|----|----------|------|----|
| Windows Client VM | WS01 | Domain Workstation | DHCP (192.168.0.0/24) |
| Windows Server VM | DC01 | Domain Controller | 192.168.0.10 (static) |

---

## Steps Completed

### 1. Created the Windows Client VM in VirtualBox

- Downloaded the Windows 10/11 Enterprise evaluation ISO from [Microsoft Evaluation Center](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-10-enterprise)
- Created a new VM in VirtualBox:
  - Type: **Microsoft Windows**, Version: **Windows 10 (64-bit)**
  - RAM: 4 GB | CPU: 2 | Disk: 60 GB (dynamically allocated)
  - Network adapter: **Bridged Adapter** (same home LAN as DC01 and Kali)
- Installed Windows without a product key

### 2. Renamed the System to WS01

After Windows installed and initial setup was complete:

```powershell
Rename-Computer -NewName "WS01" -Restart
```

### 3. Configured DNS to Point at DC01

Before joining the domain, the workstation's DNS server must point to DC01 so it can resolve `corp.lab`:

```powershell
# Run as Administrator
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.0.10
```

Verify DNS resolves the domain:

```powershell
nslookup corp.lab
# Should return 192.168.0.10
```

### 4. Joined WS01 to the corp.lab Domain

```powershell
Add-Computer `
  -DomainName "corp.lab" `
  -Credential (Get-Credential)    # enter CORP\Administrator credentials
  -Restart
```

After the reboot, WS01 was a member of the `corp.lab` domain.

### 5. Moved WS01 Computer Account into the Workstations OU

On DC01 (run as Domain Admin):

```powershell
# Get the distinguished name of the WS01 computer object
Get-ADComputer -Identity "WS01" | Select-Object DistinguishedName

# Move it into the Workstations OU
Move-ADObject `
  -Identity "CN=WS01,CN=Computers,DC=corp,DC=lab" `
  -TargetPath "OU=Workstations,DC=corp,DC=lab"
```

### 6. Logged In Using Domain Users

Rebooted WS01, then signed in using the domain accounts created in Project 02:

- `CORP\jsmith` — standard user
- `CORP\mbrown` — standard user
- `CORP\sjones` — standard user

Each login was verified to succeed and generate Windows Event ID 4624 (successful logon) on both WS01 and DC01.

### 7. Verified Network Communication with DC01

From WS01 (logged in as a domain user):

```powershell
# Ping DC01 by hostname and IP
ping DC01
ping 192.168.0.10

# Verify Kerberos tickets were issued on login
klist
```

---

## Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| Domain join failed — "domain not found" | WS01 DNS was still pointing to DHCP-assigned DNS; changed it to `192.168.0.10` (DC01) |
| VM clock skew causing Kerberos errors | Synchronized the VM clock via VirtualBox Guest Additions and confirmed both VMs matched DC01 time |
| Login as domain user failed after join | Waited for DC01 to finish propagating the computer account; reboot loop resolved it |

---

## Key Takeaways

- Setting DNS to the DC before attempting a domain join is the most common step skipped by beginners
- Moving computer accounts into the correct OU ensures GPOs scoped to the Workstations OU will apply (important for later Sysmon deployment via GPO)
- Domain user logins generate Event ID 4624 on the authenticating DC — confirming end-to-end Kerberos auth and establishing the baseline event log activity

---

## Next Steps

→ [Project 05 — Endpoint Telemetry with Sysmon](05-sysmon-setup.md)
