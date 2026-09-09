# 🚨 Security Incident Response Report: Operation Ledger Breach

## 📄 1. Incident Overview
* **Incident ID:** IR-2026-0909-001
* **Date of Detection:** September 9, 2026
* **Reported By:** SOC Tier 2 Analyst / Detection Engineering Team
* **Target Environment:** Hybrid Infrastructure (Windows Server & Ubuntu Linux)
* **Incident Classification:** Critical (Data Encryption, Privilege Escalation, Lateral Movement)
* **Status:** Contained & Eradicated

---

## 📊 2. Executive Summary
On September 9, 2026, the SOC team detected a multi-stage cyber intrusion originating from a compromised Windows Server (`10.10.30.20`). The attacker gained initial access via a malicious `.lnk` file, successfully bypassed local Windows Defender protections, and dumped LSASS memory to harvest credentials. 

Utilizing the compromised credentials, the threat actor pivoted (Lateral Movement) to an internal Ubuntu system (`10.10.30.30`) via SSH. Dual persistence mechanisms were established on the Linux host before the attacker executed a ransomware-style encryption attack targeting sensitive financial records. The attack was successfully tracked and reconstructed using custom Splunk SIEM rules.

---

## ⏱️ 3. Incident Timeline & MITRE ATT&CK Mapping
*Note: The attacker exhibited a "low and slow" operational cadence, intentionally delaying lateral movement and impact stages to evade behavioral analysis and alert fatigue thresholds.*

| Timeline | Phase | MITRE ATT&CK | Description of Activity |
| :--- | :--- | :--- | :--- |
| **Phase 1 (Day 1)** | **Initial Access** | T1204.002 | User executed a weaponized `.lnk` file containing hidden PowerShell commands. |
| **Phase 1 (Day 1)** | **Execution & C2** | T1059.001 / T1071.001 | PowerShell script downloaded a payload and established a reverse shell connection to `10.10.30.10:4444`. |
| **Phase 1 (Day 1)** | **Defense Evasion** | T1562.001 | Attacker executed commands to disable Windows Defender Real-time Protection via the C2 channel. |
| **Phase 1 (Day 1)** | **Persistence (Win)** | T1136.001 | A hidden local admin account (`WinUpdate$`) was created and added to the Administrators group. |
| **Phase 1 (Day 1)** | **Credential Access** | T1003.001 | `procdump.exe` was downloaded via `certutil` and used to dump `lsass.exe`. The dump was exfiltrated to the C2 server over SMB. |
| **Phase 2 (Day 3)** | **Lateral Movement** | T1021.004 | After a dwell time of approximately 48 hours, the attacker reused harvested credentials to authenticate via SSH to `10.10.30.30`. |
| **Phase 2 (Day 3)** | **Persistence (Linux)** | T1098.004 / T1548.003 | Attacker injected a public SSH key into `~/.ssh/authorized_keys` and modified `/etc/sudoers.d/backdoor_test` for passwordless root access. |
| **Phase 3 (Day 3)** | **Impact** | T1486 | Attacker encrypted `/var/www/html/reports/financial_audit.txt` using `openssl` and permanently deleted the original file. |
| **Phase 4 (Day 4)** | **Detection** | N/A | SOC team identified the intrusion during proactive threat hunting using custom Splunk SIEM queries targeting anomalous Sysmon and Auth logs. |

---

## 🔍 4. Indicators of Compromise (IoCs)

### Network Indicators
* **Malicious IPs:** `10.10.30.10` (Attacker C2, HTTP Server, SMB Exfiltration Server)
* **Compromised Internal IPs:** `10.10.30.20` (Windows Server / Pivot), `10.10.30.30` (Ubuntu Target)
* **Suspicious Ports:** `4444` (Reverse Shell), `8888` (Python HTTP Server)

### Host-Based Indicators (Windows)
* **Suspicious Account:** `WinUpdate$` (Local Administrator)
* **Artifacts:** `C:\Windows\Temp\procdump64.exe`, `C:\Windows\Temp\lsass.dmp`
* **Suspicious Commands:** PowerShell executions containing `-W Hidden -Exec Bypass`, `certutil.exe -urlcache -split -f`

### Host-Based Indicators (Linux)
* **Compromised Account:** `manhvd`
* **Malicious Files:** `/etc/sudoers.d/backdoor_test`, `/var/www/html/reports/financial_audit.txt.enc`
* **Suspicious Commands:** `echo "manhvd ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/backdoor_test`

---

## 🛡️ 5. Containment, Eradication & Recovery Actions

### Containment
1. Isolate the compromised Windows Server (`10.10.30.20`) and Ubuntu machine (`10.10.30.30`) from the production network to prevent further lateral movement.
2. Block all inbound and outbound traffic to/from the attacker IP (`10.10.30.10`) at the perimeter firewall.

### Eradication
1. **Windows Environment:**
   * Delete the rogue `WinUpdate$` administrator account.
   * Remove malicious artifacts (`procdump64.exe`, `lsass.dmp`) from `C:\Windows\Temp\`.
   * Re-enable and update Windows Defender Real-time Protection. Force a full system scan.
2. **Linux Environment:**
   * Remove the unauthorized public key from `/home/manhvd/.ssh/authorized_keys`.
   * Delete the malicious sudoers configuration (`rm /etc/sudoers.d/backdoor_test`).
   * Force a password reset for the `manhvd` user and all active administrative accounts.

### Recovery
1. Restore `/var/www/html/reports/financial_audit.txt` and `credentials.txt` from secure offline backups.
2. Reconnect the cleaned systems to the network and monitor closely for 72 hours.

---

## 📈 6. Lessons Learned & Detection Improvements
The Splunk SIEM successfully triggered alerts for the majority of the kill chain (Sysmon Event IDs 1, 3, 10, 11). However, the incident highlighted a gap in Linux persistence detection.

**Action Items for Security Engineering:**
* **Implement SSH Key Monitoring:** Deploy File Integrity Monitoring (FIM) or specialized auditd/Sysmon rules targeting modifications to `~/.ssh/authorized_keys` across all endpoints. Relying purely on `auth.log` failed to detect the silent injection of the persistence mechanism.
* **Enhance EDR Configurations:** Configure endpoint protection to enforce block rules against known memory-dumping utilities (like `procdump.exe`) interacting with `lsass.exe`, rather than relying solely on post-execution SIEM alerts.
* **Network Segmentation:** Restrict direct SSH access between internal servers. Implement a Jump Server / Bastion Host architecture with enforced Multi-Factor Authentication (MFA).
