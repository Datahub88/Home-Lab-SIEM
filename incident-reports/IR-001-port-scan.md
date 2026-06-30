# IR-001 — Network Reconnaissance / Port Scan Detected

**Classification:** Security Incident Report  
**Severity:** Medium  
**Status:** Closed — Contained in Lab Environment  
**Date:** June 29, 2026  
**Analyst:** Ahsan Shareef  
**Environment:** Home Lab (Isolated VirtualBox Network)

---

## 1. Executive Summary

A network reconnaissance scan was detected originating from the Kali Linux attacker machine targeting the monitored Windows 10 endpoint. The Wazuh SIEM identified the scan activity in real time, logging the source IP, target IP, and open ports discovered. The scan revealed an exposed FTP service (port 21) running on the Windows 10 VM — a high-risk finding that would require immediate remediation in a production environment.

---

## 2. Incident Details

| Field | Detail |
|---|---|
| **Incident ID** | IR-001 |
| **Date & Time** | June 29, 2026 @ 18:21 |
| **Detection Source** | Wazuh SIEM — Network alert |
| **Attack Type** | Network Reconnaissance (Port Scan) |
| **MITRE ATT&CK Technique** | T1046 — Network Service Discovery |
| **Attacker IP** | 192.168.56.103 (Kali Linux VM) |
| **Target IP** | 192.168.56.102 (Windows 10 VM) |
| **Tool Used** | Nmap 7.99 — SYN Scan (`-sS`) |
| **Wazuh Agent** | DESKTOP-I4K7876 (Agent ID: 001) |
| **Severity** | Medium |

---

## 3. Attack Description

The attacker executed an Nmap SYN scan (`nmap -sS`) against the target Windows 10 endpoint. A SYN scan works by sending TCP SYN packets to each port — if the port responds with SYN-ACK it is open, if RST it is closed. This technique is commonly used by threat actors during the reconnaissance phase of an attack to map the attack surface before exploiting vulnerabilities.

**Nmap Scan Results:**
```
Host: 192.168.56.102 — Up (0.00066s latency)
999 ports: filtered (no response)
PORT    STATE  SERVICE
21/tcp  open   FTP
MAC Address: 08:00:27:D2:4A:75 (Oracle VirtualBox)
Scan completed in 6.70 seconds
```

---

## 4. Findings

### Finding 1 — Open FTP Port (Critical)
| Field | Detail |
|---|---|
| **Port** | 21/tcp |
| **Service** | FTP (File Transfer Protocol) |
| **Risk** | Critical |
| **Description** | FTP transmits data including credentials in plaintext. This service should not be exposed on any endpoint. Running FileZilla FTP Server was identified as the source. |
| **Compliance Impact** | Violates PCI DSS Requirement 4.2.1 (encrypt data in transit), NIST 800-53 SC-8 (transmission confidentiality), and ISO 27001 A.10.1 (cryptographic controls) |
| **Remediation** | Disable FTP service immediately. Replace with SFTP (port 22) if file transfer capability is required. |

### Finding 2 — Reconnaissance Activity Not Blocked
| Field | Detail |
|---|---|
| **Risk** | Medium |
| **Description** | The Windows 10 endpoint did not block or rate-limit the incoming port scan, allowing full reconnaissance to complete in under 7 seconds |
| **Remediation** | Implement host-based firewall rules to limit inbound connection attempts. Deploy an IDS/IPS to detect and block scan activity. |

---

## 5. Timeline

| Time | Event |
|---|---|
| 18:21:00 | Nmap SYN scan initiated from Kali (192.168.56.103) |
| 18:21:06 | Scan completed — port 21 identified as open |
| 18:21:07 | Wazuh SIEM generated alert — spike visible in dashboard |
| 18:21:10 | Alert reviewed in Wazuh Discover view |
| 18:21:15 | Incident documented and classified |

---

## 6. MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|---|---|---|
| Reconnaissance | Network Service Discovery | T1046 |
| Reconnaissance | Active Scanning | T1595 |

---

## 7. Impact Assessment

| Category | Assessment |
|---|---|
| **Confidentiality** | Low — no data exfiltrated |
| **Integrity** | None — read-only reconnaissance |
| **Availability** | None — service not disrupted |
| **Overall Impact** | Low in lab — High risk in production |

---

## 8. Recommendations

1. **Immediate:** Disable FTP service (FileZilla) on the Windows 10 endpoint
2. **Short term:** Implement Windows Firewall rules blocking inbound port scan patterns
3. **Long term:** Deploy network segmentation to prevent lateral movement from attacker machines
4. **Compliance:** Address PCI DSS and NIST 800-53 violations related to unencrypted service exposure
5. **Monitoring:** Configure Wazuh alert rule to notify on-call analyst immediately when port scan detected

---

## 9. Lessons Learned

This incident demonstrated the effectiveness of SIEM-based detection for identifying reconnaissance activity. The Wazuh platform successfully detected the scan in real time and provided sufficient forensic detail to identify the source, target, and tools used. The discovery of an open FTP port highlights the importance of regular vulnerability scanning and port auditing as part of a GRC program.

---

*Report prepared by: Ahsan Shareef | Home Lab SIEM Project*  
*Tools: Wazuh 4.14.5, Nmap 7.99, VirtualBox 7.x*
