# 🔐 Home Lab SIEM – Threat Detection with Wazuh

![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=flat-square)
![SIEM](https://img.shields.io/badge/SIEM-Wazuh-005571?style=flat-square)
![Platform](https://img.shields.io/badge/Platform-VirtualBox-183A61?style=flat-square)
![OS](https://img.shields.io/badge/OS-Windows_10_|_Kali_Linux-0078D6?style=flat-square)

## 📌 Project Overview

A hands-on cybersecurity home lab built to simulate real-world threat detection and security monitoring. This project deploys **Wazuh** (open-source SIEM/XDR) across a virtualized network of Windows and Linux machines, with Kali Linux used to generate realistic attack scenarios.

The goal is to demonstrate core SOC analyst skills: log collection, alert triage, threat detection, and incident documentation.

---

## 🏗️ Lab Architecture

```
┌─────────────────────────────────────────────┐
│              VirtualBox Host (PC)           │
│                                             │
│  ┌──────────────┐    ┌────────────────────┐ │
│  │  Wazuh OVA   │◄───│   Windows 10 VM    │ │
│  │  (SIEM)      │    │   (monitored host) │ │
│  │192.168.56.101│    │  192.168.56.102    │ │
│  └──────┬───────┘    └────────────────────┘ │
│         │                                   │
│  ┌──────▼───────┐                           │
│  │  Kali Linux  │                           │
│  │  (attacker)  │                           │
│  │192.168.56.103│                           │
│  └──────────────┘                           │
│                                             │
│  Network: Host-Only Adapter (192.168.56.0/24)│
└─────────────────────────────────────────────┘
```

---

## 🛠️ Tools & Technologies

| Tool | Purpose |
|---|---|
| **VirtualBox** | Virtualization platform |
| **Wazuh 4.14.5** | SIEM / XDR — log collection, alerting, dashboards |
| **Windows 10** | Monitored endpoint (Wazuh agent installed) |
| **Kali Linux** | Attacker machine for simulating threats |
| **Nmap 7.99** | Network scanning & port enumeration |
| **Hydra v9.7** | Brute force simulation |

---

## ⚙️ Setup & Configuration

### Prerequisites
- VirtualBox installed on host PC
- Minimum 8GB RAM (allocate 4GB to Wazuh OVA, 2GB each to Win10 and Kali)
- Wazuh OVA downloaded from [wazuh.com](https://documentation.wazuh.com/current/deployment-options/virtual-machine/virtual-machine.html)

### Network Configuration
All VMs use a **Host-Only Adapter** (subnet `192.168.56.0/24`) for internal communication, plus NAT for internet access.

```
VirtualBox → File → Tools → Host Network Manager → Create adapter
  IPv4 Address: 192.168.56.1
  Subnet Mask:  255.255.255.0
  DHCP Server:  Enabled
```

### Step 1 — Import Wazuh OVA
1. Download the Wazuh OVA (~5GB) from the official site
2. In VirtualBox: File → Import Appliance → select the OVA
3. Allocate 4GB RAM, set Network Adapter 1 to NAT, Adapter 2 to Host-Only
4. Boot the VM and log in: `wazuh-user` / `wazuh`
5. Run `ip a` to confirm IP — assigned `192.168.56.101`
6. Access dashboard from host browser: `https://192.168.56.101`

### Step 2 — Install Wazuh Agent on Windows 10
1. In Wazuh dashboard → Deploy new agent → select Windows
2. Enter server address: `192.168.56.101`
3. Run the generated PowerShell command in Windows 10 VM as Administrator
4. Start the service: `NET START Wazuh`
5. Verify in Wazuh dashboard → Agents — shows **Active** with IP `192.168.56.102`

### Step 3 — Configure Kali as Attacker
Kali assigned `192.168.56.103` on the Host-Only network. Confirm connectivity:
```bash
ping 192.168.56.102    # ping the Windows VM
ping 192.168.56.101    # ping the Wazuh server
nmap -sn 192.168.56.0/24   # discover all hosts on the network
```

---

## 🧪 Attack Simulations & Detections

### Scenario 1 — Network Reconnaissance (Nmap Scan)
**Attack (from Kali — 192.168.56.103):**
```bash
nmap -sS 192.168.56.102
```
**Results:**
```
Host is up (0.00066s latency)
PORT    STATE  SERVICE
21/tcp  open   FTP
```
**Wazuh Detection:** Alert spike detected in real time. Discovered open FTP port 21 — insecure legacy protocol transmitting credentials in plaintext. High risk finding mapped to PCI DSS Requirement 4.2.1 and NIST 800-53 SC-8.

**Incident Report:** [IR-001-port-scan.md](incident-reports/IR-001-port-scan.md)

---

### Scenario 2 — Brute Force Authentication Attack (RDP)
**Attack (from Kali — 192.168.56.103):**
```bash
echo -e "password\n123456\nwazuh\nadmin\nP@ssword123\nwelcome\nletmein" > /tmp/passwords.txt
hydra -l administrator -P /tmp/passwords.txt rdp://192.168.56.102 -V
```
**Results:**
```
7 login attempts against administrator account
0 valid passwords found
Attack detected via Windows Event ID 4625 (Failed Logon)
MITRE ATT&CK: T1110 — Brute Force (auto-classified by Wazuh)
```
**Wazuh Detection:** Multiple Event ID 4625 alerts in rapid succession. Wazuh Threat Hunting dashboard automatically classified attack as **Brute Force** under MITRE ATT&CK framework. Source IP `192.168.56.103` (Kali) identified in forensic logs.

**Incident Report:** [IR-002-brute-force.md](incident-reports/IR-002-brute-force.md)

---

### Scenario 3 — Privilege Escalation (Planned)
**Method:** Create a new local admin account via cmd on Windows VM
```cmd
net user hacker P@ssword123 /add
net localgroup administrators hacker /add
```
**Expected Wazuh Alert:** Windows Event ID 4732 (User added to privileged group)

---

## 📋 Incident Report Summary

### IR-001 — Port Scan Detected
| Field | Detail |
|---|---|
| **Date** | June 29, 2026 |
| **Severity** | Medium |
| **Source IP** | 192.168.56.103 (Kali Linux) |
| **Target IP** | 192.168.56.102 (Windows 10) |
| **Tool Used** | Nmap 7.99 SYN scan |
| **Finding** | FTP port 21 open — insecure service |
| **MITRE** | T1046 — Network Service Discovery |
| **Action** | Disable FTP, implement firewall rules |

### IR-002 — Brute Force Attempt (RDP)
| Field | Detail |
|---|---|
| **Date** | June 29, 2026 |
| **Severity** | High |
| **Source IP** | 192.168.56.103 (Kali Linux) |
| **Target** | 192.168.56.102 — RDP port 3389 |
| **Event IDs** | 4625 x7 in under 5 seconds |
| **MITRE** | T1110 — Brute Force |
| **Action** | Account lockout policy, disable RDP, implement MFA |

---

## 📸 Screenshots

| Screenshot | Description |
|---|---|
| [Wazuh opening SS](screenshots/Wazuh%20opening%20SS.png) | Wazuh dashboard first login |
| [Agent 1 added](screenshots/Agent%201%20added.png) | Windows 10 agent connected and active |
| [Agent 1 details](screenshots/Agent%201%20details%20.png) | Agent detail view |
| [Adding win 10 VM as agent](screenshots/Adding%20win%2010%20VM%20as%20agent.png) | PowerShell agent installation |
| [Kali Nmap attack SS](screenshots/Kali%20Nmap%20attack%20SS.png) | Nmap scan from Kali terminal |
| [High Severity after running nmap](screenshots/High%20Severity%20after%20running%20nmap.png) | Wazuh alert spike after Nmap |
| [Wazuh alert log for nmap](screenshots/Wazuh%20alert%20log%20for%20nmap.png) | Raw alert logs from Nmap scan |
| [Brute force attack result 1](screenshots/Brute%20force%20attack%20result%201.png) | Hydra brute force in Kali |
| [Brute force attack result 2](screenshots/Brute%20force%20attack%20result%202.png) | Brute force completion |
| [Thread 1](screenshots/Thread%201.png) | Threat Hunting — MITRE Brute Force detected |
| [Thread 2](screenshots/Thread%202.png) | Alert evolution spike |
| [Thread 3](screenshots/Thread%203.png) | Overview dashboard with High severity alerts |
| [Thread 4 prt 1](screenshots/Thread%204%20prt%201.png) | Forensic logs — Event ID 4625 |
| [Thread 4 prt 2](screenshots/Thread%204%20prt%202.png) | Forensic logs continued |
| [Thread 4 prt 3](screenshots/Thread%204%20prt%203.png) | Forensic logs continued |
| [Details of alerts in Agent 1](screenshots/Details%20of%20alerts%20in%20Agent%201.png) | SCA compliance findings |

---

## 🔮 Next Steps / Roadmap

- [ ] Simulate privilege escalation — Windows Event ID 4732
- [ ] Configure Wazuh File Integrity Monitoring (FIM)
- [ ] Add Ubuntu server VM as third monitored host
- [ ] Build Grafana dashboard on top of Wazuh data
- [ ] Deploy honeypot and collect real attacker data

---

## 📚 References & Resources

- [Wazuh Documentation](https://documentation.wazuh.com)
- [MITRE ATT&CK Framework](https://attack.mitre.org)
- [Windows Security Event IDs](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/)
- [VirtualBox Networking Guide](https://www.virtualbox.org/manual/ch06.html)

---

## 👤 Author

**Ahsan Shareef** | CompTIA Security+ | Google Cybersecurity | Azure AZ-900 | MBA | IRS Enrolled Agent

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/ahsanshareef88/)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat-square&logo=github&logoColor=white)](https://github.com/Datahub88)

> *This project is built in a controlled, isolated lab environment. All attack simulations are performed only against machines I own.*
