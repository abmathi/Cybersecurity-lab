# Elastic SIEM SOC Home Lab

This repository documents a self-hosted SOC / Blue Team home lab built to practice endpoint telemetry, centralized log collection, detection engineering, and alert investigation in an Active Directory environment.

The lab uses Elastic Stack, Fleet Server, Elastic Agent, Windows Event Forwarding, Sysmon, and Kali Linux attack simulation to generate and investigate realistic security events.

## Project Goals

- Build a small enterprise-style Active Directory lab.
- Collect Windows Security and Sysmon telemetry from domain systems.
- Deploy Elastic Stack as the SIEM platform.
- Enroll Windows endpoints with Elastic Agent and Fleet.
- Create and validate custom detection rules.
- Simulate attack activity from Kali Linux.
- Investigate generated alerts using Kibana and Elastic Security.

## Lab Architecture

| System | Role | Notes |
|---|---|---|
| Kali Linux | Attack machine | Used for Kerberoasting and lateral movement simulation |
| DC01 | Domain Controller | Active Directory, DNS, Sysmon, WEF collector |
| WS01 | Domain workstation | Domain-joined Windows endpoint with Sysmon and Elastic Agent |
| Ubuntu SIEM Server | Elastic Stack host | Elasticsearch, Kibana, Fleet Server |

Domain: `corp.lab`  
Network: `192.168.0.0/24`

## Completed Work

- Created a Windows Active Directory domain named `corp.lab`.
- Configured a domain controller, DNS, organizational units, users, admin accounts, and service accounts.
- Added intentional lab misconfigurations to support attack simulation.
- Installed Sysmon on Windows systems for enhanced endpoint telemetry.
- Configured Windows Event Forwarding for centralized event collection.
- Installed and configured Elastic Stack on Ubuntu.
- Configured Fleet Server and enrolled Windows endpoints with Elastic Agent.
- Confirmed logs flowing into Elastic/Kibana.
- Installed Elastic prebuilt detection rules.
- Created custom detection rules for failed logons and Kerberoasting activity.
- Simulated Kerberoasting and lateral movement attempts from Kali Linux.
- Validated generated alerts and reviewed investigation details in Elastic Security.

## Skills Demonstrated

- SIEM deployment and administration
- Windows endpoint logging
- Sysmon configuration and validation
- Windows Event Forwarding
- Active Directory administration
- Detection engineering
- KQL-based log analysis
- Alert triage and investigation
- MITRE ATT&CK mapping
- Offensive security simulation in a controlled lab
- Technical documentation

## Documentation

| Phase | Document |
|---|---|
| 01 | [Environment Setup](docs/projects/01-environment-setup.md) |
| 02 | [Active Directory Domain Setup](docs/projects/02-active-directory-lab.md) |
| 03 | [AD Misconfigurations](docs/projects/03-ad-misconfigurations.md) |
| 04 | [Workstation Deployment](docs/projects/04-workstation-deployment.md) |
| 05 | [Sysmon Setup](docs/projects/05-sysmon-setup.md) |
| 06 | [Windows Event Forwarding](docs/projects/06-wef-logging.md) |
| 07 | [SOC Infrastructure](docs/projects/07-soc-infrastructure.md) |
| 08 | [Elastic Stack Setup](docs/projects/08-elastic-stack-setup.md) |
| 09 | [Elastic SIEM Setup](docs/projects/09-elastic-siem-setup.md) |
| 10 | [Kerberoasting Detection](docs/projects/10-kerberoasting-detection.md) |
| 11 | [Custom Detection Rules](docs/projects/11-custom-detection-rules.md) |
| 12 | [Lateral Movement Lab](docs/projects/12-lateral-movement-lab.md) |
| 13 | [PowerShell Logging Tuning - Planned Enhancement](docs/projects/13-powershell-logging-tuning.md) |

## Detection Work Completed

| Detection | Data Source | Status |
|---|---|---|
| Excessive failed logons | Windows Security Event ID 4625 | Validated |
| Kerberoasting activity | Windows Security Event ID 4769 | Validated |
| Remote service creation | Windows Event ID 7045 / Sysmon | Observed |
| Suspicious process creation | Sysmon Event ID 1 | Observed |
| Lateral movement attempts | Windows Security + Sysmon | Initial validation complete |

## Future Enhancements

- Enable and validate PowerShell Script Block Logging.
- Add PowerShell-specific detection rules.
- Add a visual architecture diagram.
- Export custom Elastic detection rules to the repository.
- Add incident-style writeups for each simulated attack.
- Expand lateral movement testing and detection coverage.

## Disclaimer

This lab was built in a private, controlled environment for educational and portfolio purposes only. All attack simulations were performed against systems owned and controlled by the lab owner.
