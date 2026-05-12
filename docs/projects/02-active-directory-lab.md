# Project 02 — Active Directory Domain Setup

**Skills:** Windows Server, Active Directory Domain Services, DNS, Server Manager GUI, Active Directory Users and Computers, Organizational Units

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

After Windows installed and the first login was complete:

![Renaming Server](../assets/DC01%20setup/1%20naming%20server.png)



### 3. Configured a Static IP

After reboot, set a static IP so DNS remains stable when the system becomes a DC:


![static ip setup](../assets/DC01%20setup/2%20ip%20configuration.png)


Verified the static IP was applied:


![ip verification](../assets/DC01%20setup/3%20network%20verification.png)

### 4. Rebooted the Server

```powershell
Restart-Computer
```

### 5. Installed Active Directory Domain Services Using Server Manager

After logging back in as local Administrator:


![installl active directory](../assets/DC01%20setup/4%20setting%20up%20roles.png)



### 6. Promoted the Server to Domain Controller


Promoted DC01 to a domain controller and created a new forest named `corp.lab` using the Active Directory Domain Services Configuration Wizard:

![set up domain controller](../assets/DC01%20setup/5%20setting%20up%20domain%20controller.png)

The server rebooted automatically. After reboot, logged in as `CORP\Administrator`.


### 7. Created Organisational Units

![created OUs](../assets/DC01%20setup/7%20created%20OUs.png)



### 8. Created Domain Users

![created domain users](../assets/DC01%20setup/8%20created%20standard%20users.png)

![admin accounts](../assets/DC01%20setup/9%20created%20admin.png)



### 9. Created Service Accounts

![service accounts](../assets/DC01%20setup/10%20created%20service%20accounts.png)



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

