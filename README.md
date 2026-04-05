# Cybersecurity Home Lab | SOC / Blue Team Detection Lab

This project is a self-hosted cybersecurity home lab built to simulate real-world Security Operations Center (SOC) workflows in an enterprise-style environment.

The lab focuses on **threat detection, log analysis, alert triage, and attack investigation** using Elastic SIEM, Sysmon, Windows Event Forwarding (WEF), and Active Directory.

It is designed as a portfolio project to demonstrate hands-on experience with **security monitoring, detection engineering, and incident investigation**.

---

## 🔥 Key Highlights

- Built a simulated **Active Directory enterprise environment (corp.lab)**
- Deployed a **self-hosted Elastic Stack SIEM (Elasticsearch, Kibana, Fleet)**
- Configured **Windows Event Forwarding (WEF)** for centralized logging
- Integrated **Sysmon** for enhanced endpoint telemetry
- Enrolled endpoints using **Elastic Agent + Fleet Server**
- Created and tuned **custom detection rules (e.g., Event ID 4625)**
- Simulated real-world attacks (Kerberoasting, password spraying, lateral movement)
- Investigated logs and alerts using **Kibana + KQL queries**

---

## 🏗️ Lab Architecture

This lab simulates a small enterprise network:

- **Attack Machine:** Kali Linux (on Intel Mac)
- **Domain Controller (DC01):** Windows Server (AD DS, DNS, Sysmon, WEF Collector)
- **Workstation (WS01):** Domain-joined Windows endpoint
- **SIEM Server:** Ubuntu Server (Elastic Stack + Fleet)

**Domain:** `corp.lab`  
**Network:** `192.168.0.0/24`

See full architecture → `docs/architecture/network-topology.md`

---

## ✅ Implemented Components

| Component | Status | Notes |
|----------|--------|------|
| Active Directory Domain | ✅ Complete | corp.lab fully functional |
| Sysmon Telemetry | ✅ Complete | Installed on DC01 & WS01 |
| Windows Event Forwarding | ✅ Complete | Centralized log collection |
| Elastic Stack Deployment | ✅ Complete | Elasticsearch + Kibana |
| Fleet Server | ✅ Complete | Agent management |
| Elastic Agent | ✅ Complete | Enrolled endpoints |
| Log Ingestion | ✅ Complete | Real-time logs in Kibana |
| Detection Rules | ✅ Complete | Custom + prebuilt rules |
| Attack Simulation | 🔄 Ongoing | Expanding detection coverage |

---

## 🛠️ Technologies & Tools

### Infrastructure
- VirtualBox
- Windows Server (Active Directory)
- Windows Client
- Ubuntu Server

### SIEM & Monitoring
- Elasticsearch
- Kibana
- Elastic Security
- Fleet Server
- Elastic Agent

### Telemetry & Logging
- Sysmon
- Windows Event Forwarding (WEF)

### Adversary Simulation
- Kali Linux
- Impacket
- CrackMapExec
- Hashcat

### Analysis & Investigation
- KQL (Kibana Query Language)
- Wireshark
- MITRE ATT&CK Framework

---

## 🧠 Detection & Investigation Focus

This lab was built to simulate real SOC analyst workflows:

- Monitoring Windows Security and Sysmon logs in Elastic SIEM
- Investigating authentication anomalies (e.g., failed logon spikes — Event ID 4625)
- Detecting Kerberos abuse (Kerberoasting activity — Event ID 4769)
- Correlating logs across multiple systems to identify attack patterns
- Writing and tuning detection rules to reduce false positives
- Validating detections through controlled adversary simulation

---

## 🎯 Detection Scenarios

- Brute-force detection (Event ID 4625)
- Kerberoasting detection (Event ID 4769)
- Lateral movement detection (SMB / WMI)
- PowerShell logging and detection tuning
- Alert triage and investigation workflows

Detailed lab walkthroughs are located in the `docs/` directory.

---

## 💪 Skills Demonstrated

- Active Directory security and attack detection
- SIEM deployment and log analysis (Elastic Stack)
- Sysmon and Windows Event Forwarding configuration
- Endpoint telemetry collection and event correlation
- Detection engineering and alert tuning
- Threat detection (brute force, Kerberos abuse, lateral movement)
- Incident investigation using Kibana and KQL
- Security documentation and reporting

---


## 📁 Repository Structure

```text
.
├── README.md
└── docs/
    ├── architecture/
    ├── assets/
    ├── projects/
    ├── setup/
    └── tools/
```

---

## 🚀 Future Improvements

* Expand Kerberoasting and SPN abuse detections
* Add lateral movement detections (Pass-the-Hash, WMI)
* Enable PowerShell Script Block logging
* Build dashboards for authentication anomalies
* Document full investigation playbooks

---

## ⚠️ Disclaimer

This lab is fully self-contained and intended for educational and portfolio purposes only.
All offensive techniques were executed only within a controlled environment owned by the lab operator.

