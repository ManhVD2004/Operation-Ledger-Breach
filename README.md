# Hybrid Network Attack & SOC Detection Lab

## Project Overview
This project simulates a comprehensive 9-stage cyber attack (Red Team) across a hybrid Windows/Linux environment, followed by the development of custom detection rules using Splunk SIEM (Blue Team). 

The objective is to demonstrate practical application of the MITRE ATT&CK framework, adversary tradecraft, lateral movement, and proactive detection engineering (Threat Hunting) by building a logging and monitoring pipeline from scratch.

---

## 1. Lab Architecture & Telemetry Setup
The environment consists of isolated virtual machines communicating over a custom internal network (`10.10.30.0/24`). A centralized logging pipeline was established to ensure complete visibility of both Windows and Linux endpoint activities.

### Network Topology & Data Flow
![Network Topology](images/network-topology.png)

### Attacker Infrastructure
* **Machine:** Kali Linux (`10.10.30.10`)
* **Role:** C2 Server, Payload Hosting (Python HTTP Server), Exfiltration Server (Impacket SMB).

### Victim 1: Initial Breach & Pivot
* **Machine:** Windows Server (`10.10.30.20`)
* **Role:** Initial compromise vector (Phishing/Malicious LNK).
* **Data Telemetry & Log Sources:**
  * **Sysmon (System Monitor):** Installed with a custom XML configuration file to monitor Process Creation (Event ID 1), Network Connections (Event ID 3), and Process Access/LSASS dumping (Event ID 10).
  * **Windows Event Logs:** Security logs and PowerShell operational logs (Script Block Logging enabled).
  * **Log Forwarding:** Splunk Universal Forwarder (UF) installed, configured to monitor `XmlWinEventLog:Microsoft-Windows-Sysmon/Operational` and forward to the SIEM via port `9997`.

### Victim 2: Target & Impact
* **Machine:** Ubuntu Desktop (`10.10.30.30`)
* **Role:** Target for lateral movement and simulated ransomware impact.
* **Data Telemetry & Log Sources:**
  * **Sysmon for Linux:** Deployed and configured via a custom XML rule file to monitor critical file modifications (e.g., `/etc/sudoers.d`) and process executions (e.g., `openssl`, `rm`).
  * **System Logs:** `/var/log/auth.log` monitored for SSH authentication events.
  * **Log Forwarding:** Splunk Universal Forwarder (UF) deployed to forward Sysmon and internal authentication logs to the SIEM.

### SOC / SIEM
* **Machine:** Splunk Enterprise (Hosted on primary host/dedicated VM).
* **Role:** Centralized log aggregation, indexing (`index=main`), and correlation. Configured with receiving port `9997` to accept telemetry from Universal Forwarders.

---

## 2. Execution Workflow (Step-by-Step Demo)
To reproduce this lab and validate the detection rules, the simulation is executed in the following chronological phases:

1. **Environment Preparation:** Boot all VMs. Verify Splunk is receiving data by running a basic `index=main` query.
2. **Attack Emulation (Red Team):** Execute stages 1 through 9 sequentially. Start with the malicious `.lnk` file on Windows to establish the initial C2 connection. Once the reverse shell is active, disable Windows Defender to prepare for tool dropping. Proceed to dump credentials, move laterally via SSH to Ubuntu, establish persistence, and execute the final ransomware simulation.
3. **Log Ingestion & Delay:** Allow 2-3 minutes for the Universal Forwarders to push all generated Sysmon and Auth logs to the Splunk indexer.
4. **Threat Hunting (Blue Team):** Utilize the custom Search Processing Language (SPL) queries (Rules 1-7) in Splunk to identify, correlate, and document the attack chain.

---

## 3. Attack Emulation (Red Team Kill Chain)

### Stage 1: Initial Access (T1204.002 - Malicious File)
The attack begins with a weaponized Windows Shortcut (`.lnk`) file disguised as a legitimate document. The properties reveal a hidden PowerShell execution string (`-W Hidden`).
![LNK Properties](images/stage1-lnk-properties.png)

### Stage 2 & 3: Execution & Command and Control (T1059.001 & T1071.001)
The execution of the `.lnk` file triggers a PowerShell script that downloads a reverse shell payload from the Attacker's server and establishes a C2 connection back to `10.10.30.10`.
![Payload Script](images/stage2-payload-script.png)
![C2 Connection](images/stage3-c2-connection.png)

### Post-Exploitation: Defense Evasion (T1562.001 - Impair Defenses)
With the C2 session established, the attacker executes commands via the reverse shell to disable Windows Defender's real-time monitoring. This crucial step ensures that subsequent heavy operations (like downloading tools and memory dumping) will not be intercepted or quarantined by the local AV.
![Defense Evasion](images/disable-defender.png)

### Stage 4: Windows Persistence & Privilege Escalation (T1136.001 & T1078.001)
To maintain access, a hidden local administrator account named `WinUpdate$` is created and added to the `Administrators` group.
![Windows Persistence](images/stage4-windows-persistence.png)

### Stage 5: Discovery (T1018 - Remote System Discovery)
Using ARP scanning, the attacker maps the internal network and identifies a secondary target: an Ubuntu machine at `10.10.30.30`.
![Network Discovery](images/stage5-discovery.png)

### Stage 6: Credential Access & Exfiltration (T1003.001 & T1041)
The attacker dumps the LSASS process to extract plaintext credentials. To evade local EDR, the dump file is exfiltrated to the Kali machine for offline extraction.
* **Ingress Tool Transfer:** Downloading `procdump.exe` using `certutil` from the attacker's HTTP server.
![Kali HTTP Server](images/stage6a-kali-httpserver.png)
![Certutil Download](images/stage6b-download-procdump.png)
* **Execution:** Dumping LSASS.
![Execute Procdump](images/stage6c-execute-procdump.png)
* **Exfiltration over SMB:** Moving `lsass.dmp` to Kali via Impacket SMB server.
![Impacket SMB](images/stage6d-impacket-smb.png)
![Copy via SMB](images/stage6e-smb-copy.png)
![Verify Dump](images/stage6f-verify-dump.png)
* **Offline Extraction:** Using `pypykatz` to extract the password (`Password123`).
![Pypykatz Extract](images/stage6g-pypykatz-extract.png)

### Stage 7: Lateral Movement (T1021.004 - SSH)
By inspecting the PowerShell history (`ConsoleHost_history.txt`), the attacker discovers previous SSH activity to the Ubuntu machine. Utilizing the extracted password, the attacker successfully SSHs into the Ubuntu machine (`10.10.30.30`) by pivoting through the compromised Windows Server (`10.10.30.20`) and directly from Kali (`10.10.30.10`).
![PowerShell History](images/stage7a-ps-history.png)
![SSH Lateral Movement](images/stage7b-ssh-lateral.png)

### Stage 8: Linux Persistence (T1098.004 & T1548.003)
The attacker establishes dual persistence on the Linux machine:
1. Pushing a public SSH key to `~/.ssh/authorized_keys` for passwordless login.
2. Modifying `/etc/sudoers.d/backdoor_test` to grant the user `manhvd` root privileges without a password (Sudo Caching/PrivEsc).
![Kali Keygen](images/stage8a-kali-keygen.png)
![Authorized Keys](images/stage8b-authorized-keys.png)
![Sudoers Backdoor](images/stage8c-sudoers-backdoor.png)

### Stage 9: Impact (T1486 - Data Encrypted for Impact)
The attacker locates sensitive financial reports, encrypts them using `openssl` (AES-256-CBC), and permanently deletes the original files, simulating a targeted Ransomware deployment.
![Target Discovery](images/stage9a-target-discovery.png)
![OpenSSL Encrypt](images/stage9b-openssl-encrypt.png)
![Encrypted Result](images/stage9c-encrypted-result.png)

---

## 4. Detection Engineering (Blue Team Splunk Rules)

Following the attack execution, custom SPL (Search Processing Language) queries were developed in Splunk to detect each stage of the attack lifecycle based on the ingested telemetry.

### Rule 1: Malicious PowerShell Execution (Defense Evasion)
**Logic:** Detects PowerShell processes attempting to hide the window, bypass execution policies, or utilize `WebClient` for external downloads. Evaluates risk based on the parent process.
![Rule 1 - PowerShell Evasion](images/rule1-splunk.png)

### Rule 2: C2 Network Connection via LOLBins
**Logic:** Correlates Sysmon Event ID 1 (Process Creation) and Event ID 3 (Network Connection) via `ProcessGuid` to identify `powershell.exe` establishing external connections to unusual ports.
![Rule 2 - C2 Connection](images/rule2-splunk.png)

### Rule 3: Suspicious Local Account Creation (Persistence)
**Logic:** Detects the execution of `net.exe` spawned by `powershell.exe` to create new local users and add them to the `Administrators` group (e.g., `WinUpdate$`).
![Rule 3 - Local Account Creation](images/rule3-splunk.png)

### Rule 4: LSASS Memory Dumping (Credential Access)
**Logic:** Monitors Sysmon logs for command-line arguments indicative of process dumping tools (e.g., `procdump`, `-ma lsass.exe`) interacting with the Local Security Authority Subsystem Service.
![Rule 4 - LSASS Access](images/rule4-splunk.png)

### Rule 5: Internal SSH Lateral Movement
**Logic:** Parses `/var/log/auth.log` to identify successful SSH authentications. Highlights anomalies such as logins originating from the Attacker IP (`10.10.30.10`) or Pivot IP (`10.10.30.20`).
![Rule 5 - SSH Lateral Movement](images/rule5-splunk.png)

### Rule 6: Linux Privilege Escalation (Sudoers Modification)
**Logic:** Monitors for File Creation/Modification events (Sysmon Event ID 11) within the highly sensitive `/etc/sudoers.d/` directory, identifying potential root backdoors.
![Rule 6 - Linux Persistence](images/rule6-splunk.png)

### Rule 7: Ransomware Impact Simulation
**Logic:** Correlates a rapid succession of process creation events (`ls`, `openssl`, `rm`) originating from the same host, targeting a specific directory (`reports`). Identifies data encryption followed by data destruction.
![Rule 7 - Ransomware Impact](images/rule7-splunk.png)

---

## 5. Detection Gaps & Recommendations
During the log analysis phase, a critical detection gap was identified:
* **The Gap:** While the SIEM successfully detected the `sudoers.d` modification (Rule 6), it did not alert on the insertion of the SSH public key into `~/.ssh/authorized_keys` (Stage 8b). 
* **Recommendation:** SOC teams should implement additional File Integrity Monitoring (FIM) or Sysmon Event ID 11 rules specifically targeting changes to `.ssh` directories across all endpoints. Relying solely on `auth.log` is insufficient for detecting persistence mechanisms established before authentication occurs.

---
## 📑 Incident Response Report
For a detailed post-incident breakdown, IOCs, and eradication steps, please refer to the [Incident Response Report](Incident-report.md).
