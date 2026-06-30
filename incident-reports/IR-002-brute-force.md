# IR-002 — Brute Force Authentication Attack Detected

**Classification:** Security Incident Report  
**Severity:** High  
**Status:** Closed — Contained in Lab Environment  
**Date:** June 29, 2026  
**Analyst:** Ahsan Shareef  
**Environment:** Home Lab (Isolated VirtualBox Network)

---

## 1. Executive Summary

A brute force authentication attack targeting the Windows Remote Desktop Protocol (RDP) service was detected and logged by the Wazuh SIEM. The attacker used Hydra, an automated credential stuffing tool, to attempt multiple password combinations against the Administrator account. Wazuh detected the attack in real time, generating Windows Event ID 4625 (Failed Logon) alerts and automatically classifying the technique as Brute Force under the MITRE ATT&CK framework. No credentials were successfully compromised.

---

## 2. Incident Details

| Field | Detail |
|---|---|
| **Incident ID** | IR-002 |
| **Date & Time** | June 29, 2026 @ 20:24 |
| **Detection Source** | Wazuh SIEM — Windows Event ID 4625 |
| **Attack Type** | Brute Force / Credential Stuffing |
| **MITRE ATT&CK Technique** | T1110 — Brute Force |
| **Attacker IP** | 192.168.56.103 (Kali Linux VM) |
| **Attacker Hostname** | kali |
| **Target IP** | 192.168.56.102 (Windows 10 VM) |
| **Target Port** | 3389 (RDP) |
| **Target Account** | Administrator |
| **Tool Used** | Hydra v9.7 |
| **Authentication Method** | NTLM |
| **Wazuh Agent** | DESKTOP-I4K7876 (Agent ID: 001) |
| **Severity** | High |
| **Outcome** | Attack failed — 0 valid credentials found |

---

## 3. Attack Description

Following the initial reconnaissance scan (IR-001), the attacker escalated to an active credential attack against the RDP service discovered on the target. Using Hydra, the attacker automated login attempts against the Administrator account using a wordlist of common passwords.

**Attack Command Used:**
```bash
hydra -l administrator -P /tmp/passwords.txt rdp://192.168.56.102 -V
```

**Hydra Attack Log:**
```
[DATA] attacking rdp://192.168.56.102:3389/
[ATTEMPT] login "administrator" - pass "password"    — FAILED
[ATTEMPT] login "administrator" - pass "123456"      — FAILED
[ATTEMPT] login "administrator" - pass "wazuh"       — FAILED
[ATTEMPT] login "administrator" - pass "admin"       — FAILED
[ATTEMPT] login "administrator" - pass "P@ssword123" — FAILED
[ATTEMPT] login "administrator" - pass "welcome"     — FAILED
[ATTEMPT] login "administrator" - pass "letmein"     — FAILED
Result: 0 valid passwords found
```

---

## 4. Wazuh Detection Evidence

### Windows Event ID 4625 — Failed Logon (Multiple)

Wazuh captured multiple failed authentication events in rapid succession, consistent with automated brute force activity:

| Field | Value |
|---|---|
| **Event ID** | 4625 (An account failed to log on) |
| **Source IP** | 192.168.56.103 |
| **Workstation** | kali |
| **Target Account** | administrator |
| **Auth Package** | NTLM / NtLmSsp |
| **Logon Type** | 3 (Network) |
| **Failure Reason** | %%2313 (Unknown username or bad password) |
| **Severity** | AUDIT_FAILURE |
| **Frequency** | 7 attempts in under 5 seconds |

### Wazuh Threat Hunting Dashboard
- **Authentication failures:** 1 brute force event cluster detected
- **MITRE ATT&CK classification:** Brute Force (automatic)
- **Alert evolution:** Visible spike at 20:24 in timeline graph
- **Affected agent:** DESKTOP-I4K7876

---

## 5. Timeline

| Time | Event |
|---|---|
| 18:21 | IR-001 — Nmap scan revealed RDP port open (see IR-001) |
| 20:24:23.070 | First Hydra authentication attempt detected by Wazuh |
| 20:24:23.109 | Second attempt — Event ID 4625 logged |
| 20:24:23.140 | Third attempt — Event ID 4625 logged |
| 20:24:23.169 | Fourth attempt — Event ID 4625 logged |
| 20:24:23.189 | Fifth attempt — Event ID 4625 logged |
| 20:24:25.421 | Sixth attempt — Event ID 4625 logged |
| 20:24:25.427 | Seventh attempt — AUDIT_FAILURE logged |
| 20:24:25.427 | Wazuh brute force alert triggered |
| 20:25:00 | Attack concluded — no successful logins |
| 20:35:00 | Incident reviewed in Wazuh Discover — 643 total hits |

---

## 6. MITRE ATT&CK Mapping

| Tactic | Technique | Sub-technique | ID |
|---|---|---|---|
| Credential Access | Brute Force | Password Guessing | T1110.001 |
| Initial Access | Exploit Public-Facing Application | RDP Exposure | T1190 |
| Reconnaissance | Active Scanning | (from IR-001) | T1595 |

---

## 7. Impact Assessment

| Category | Assessment |
|---|---|
| **Confidentiality** | None — no credentials compromised |
| **Integrity** | None — no unauthorized access gained |
| **Availability** | Low — RDP service remained functional |
| **Overall Impact** | Low in lab — Critical risk in production |

**Risk in production environment:** An exposed RDP port with a weak or common Administrator password would result in full system compromise. RDP brute force is one of the most common initial access vectors used by ransomware groups including LockBit and BlackCat.

---

## 8. Recommendations

### Immediate Actions
1. **Disable RDP** if not required — remove attack surface entirely
2. **Rename Administrator account** — reduces effectiveness of targeted brute force
3. **Implement account lockout policy** — lock account after 3-5 failed attempts (GPO setting)

### Short Term
4. **Enable Network Level Authentication (NLA)** for RDP — requires valid credentials before session established
5. **Restrict RDP access by IP** — whitelist only known management IPs via firewall rules
6. **Implement Multi-Factor Authentication (MFA)** for all remote access

### Long Term
7. **Deploy a VPN** — eliminate direct RDP exposure entirely, require VPN before RDP
8. **Regular password auditing** — ensure no accounts use common/weak passwords
9. **Wazuh active response** — configure automatic IP blocking after threshold of failed logins

### Compliance Mapping
| Recommendation | Framework | Control |
|---|---|---|
| Account lockout policy | NIST 800-53 | AC-7 |
| MFA for remote access | PCI DSS | Requirement 8.4 |
| Encrypt remote sessions | HIPAA | §164.312(e) |
| Access control restrictions | ISO 27001 | A.9.4.2 |
| Monitor authentication failures | SOC 2 | CC6.1 |

---

## 9. Lessons Learned

This incident demonstrated a complete attack chain — from reconnaissance (IR-001) to active exploitation attempt (IR-002). The attacker used information gathered in the port scan to target a specific service with a credential attack. This highlights the importance of:

1. **Defense in depth** — multiple security controls at each layer
2. **Minimizing attack surface** — disable any service not explicitly required
3. **Real-time monitoring** — Wazuh detected and classified the attack automatically using MITRE ATT&CK framework without any manual configuration
4. **GRC relevance** — multiple compliance frameworks (NIST, PCI DSS, HIPAA, ISO 27001) all require controls that would have prevented or mitigated this attack

The Wazuh SIEM platform performed as expected, providing full forensic detail including source IP, target account, authentication method, failure reason codes, and automatic threat classification — giving an analyst everything needed to respond effectively.

---

*Report prepared by: Ahsan Shareef | Home Lab SIEM Project*  
*Tools: Wazuh 4.14.5, Hydra v9.7, VirtualBox 7.x*  
*References: MITRE ATT&CK T1110, NIST SP 800-53, PCI DSS v4.0*
