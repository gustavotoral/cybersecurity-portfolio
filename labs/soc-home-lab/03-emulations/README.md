# Phase 3: Adversary Simulation & Detection Validation

## Objective

In this phase, I used Kali Linux to generate controlled security activity against the Windows 11 endpoint and validate whether the resulting events were captured and surfaced in Wazuh.

I wanted to answer a simple question:

> **If suspicious activity occurs on the Windows endpoint, what evidence does it generate, and does that evidence reach the SIEM?**

I tested two scenarios:

1. RDP password-guessing activity
2. Windows Security event log clearing

The deeper analyst investigation, risk assessment, and recommended response are documented in **Phase 4: SOC Investigation, Triage & Risk Assessment**.

---

## 1. Prepare the Windows Endpoint

Before generating RDP authentication activity, I verified whether Remote Desktop was accessible from Kali Linux.

From Kali, I scanned TCP port 3389:

```bash
nmap -p 3389 192.168.10.10
```

The initial scan showed that TCP 3389 was not accepting connections.

### Nmap Scan

[View Screenshot](https://github.com/gustavotoral/cybersecurity-portfolio/blob/main/labs/soc-home-lab/assets/kali_nmap_scan_rdp_port_3389.png)

### Troubleshooting

I enabled Remote Desktop on the Windows test endpoint and then verified that TCP 3389 was listening:

```powershell
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0

Get-NetTCPConnection -LocalPort 3389 | Select-Object LocalAddress, LocalPort, State
```

### RDP Listening

[View Screenshot](https://github.com/gustavotoral/cybersecurity-portfolio/blob/main/labs/soc-home-lab/assets/win11_powershell_verify_rdp_listening.png)

This confirmed that the endpoint was ready for the controlled authentication test.

---

## 2. Generate RDP Password-Guessing Activity

I used Hydra from Kali Linux to generate repeated RDP authentication attempts against the Windows endpoint.

```bash
hydra -l soc_analyst -P /usr/share/wordlists/rockyou.txt rdp://192.168.10.10 -V -t 4
```

### Hydra RDP Test

[View Screenshot](https://github.com/gustavotoral/cybersecurity-portfolio/blob/main/labs/soc-home-lab/assets/kali_hydra_rdp_brute_force_attack.png)

The purpose was not to gain access.

The goal was to generate repeated failed authentication activity and determine whether the Windows endpoint and Wazuh monitoring pipeline would capture useful evidence.

### MITRE ATT&CK

**T1110.001 — Password Guessing**

This technique represents repeated attempts to authenticate using potential passwords.

---

## 3. Validate Windows Authentication Evidence

The RDP attempts generated Windows Security Event ID **4625 — An account failed to log on**.

Relevant fields included:

* **Event ID:** 4625
* **Logon Type:** 10 — Remote Interactive
* **Source IP:** `192.168.10.20`
* **Target Account:** `soc_analyst`
* **Target:** Windows 11 endpoint

The source IP matched the Kali Linux machine used to generate the activity.

This allowed me to connect the failed authentication events back to the system that generated them.

### Windows Event ID 4625

> **Screenshot path needs correction before publishing:** the current repository link references an Event ID 4658 image rather than the Event ID 4625 evidence described here.

### Detection Observation

One failed authentication by itself would not necessarily indicate malicious activity.

What made this test useful was the repeated pattern:

> **Multiple failed remote authentication attempts from the same source against the same account within a short period.**

That pattern provides much more useful context for detection and investigation than an isolated failed login.

---

## 4. Validate Wazuh Detection

The Windows authentication events were collected by the Wazuh agent and forwarded to the Wazuh manager.

Wazuh recorded the individual authentication failures as **Rule 60122 (Level 5)** and correlated the repeated failures into **Rule 60204 (Level 10) — Multiple Windows Logon Failures**.

### Wazuh Authentication Events

[View Screenshot](https://github.com/gustavotoral/cybersecurity-portfolio/blob/main/labs/soc-home-lab/assets/wazuh_alert_brute_force_logon_failures.png)

This validated both event ingestion and Wazuh's ability to correlate the repeated authentication failures into a higher-severity detection.

```text
Kali RDP Attempts
        ↓
Windows Authentication Activity
        ↓
Windows Event ID 4625
        ↓
Wazuh Agent
        ↓
Wazuh Manager
        ↓
Wazuh Dashboard
```

The important result was not simply that an alert appeared.

I was able to connect the activity generated from Kali to the Windows event data and then verify that the same activity was available for analysis in Wazuh.

---

## 5. Simulate Windows Security Log Clearing

For the second scenario, I tested whether clearing the Windows Security log would generate detectable evidence.

I executed:

```powershell
wevtutil cl Security
```

Windows generated Event ID **1102 — The audit log was cleared**.

Wazuh received the event and generated an alert.

### Wazuh Log-Clearing Alert

[View Screenshot](https://github.com/gustavotoral/cybersecurity-portfolio/blob/main/labs/soc-home-lab/assets/wazuh_alert_rule_63103_audit_log_cleared.png)

The Wazuh alert included:

* **Wazuh Rule:** 63103
* **Windows Event:** 1102
* **Activity:** Security audit log cleared

### MITRE ATT&CK

**T1070.001 — Clear Windows Event Logs**

This technique is associated with defense evasion because an attacker may attempt to remove evidence of previous activity.

An important observation from the test was:

> **Attempting to remove evidence can itself generate evidence.**

---

## 6. Detection Validation Results

| Test | Evidence | Wazuh Result |
| --- | --- | --- |
| RDP password guessing | Windows Event ID 4625 | Rule 60122 (Level 5) individual failures; Rule 60204 (Level 10) multiple-logon-failure detection |
| Security log clearing | Windows Event ID 1102 | Rule 63103 (Level 5) alert generated |

Both scenarios demonstrated that security activity on the Windows endpoint could be captured locally and forwarded into the SIEM for further analysis.

---

## What I Learned

This phase helped me understand the relationship between **activity, telemetry, and detection**.

The basic flow was:

```text
Generate Activity
      ↓
Create Endpoint Evidence
      ↓
Collect the Event
      ↓
Forward to Wazuh
      ↓
Validate Detection
```

I also learned that seeing an event in a SIEM is not the end of the process.

The next step is determining what the evidence actually means:

* Is the activity expected?
* Is it suspicious or malicious?
* How strong is the evidence?
* What is the potential impact?
* What should happen next?

Those questions became the focus of the next phase.

---

## Phase 3 Summary

In this phase, I:

* Verified that RDP was available before testing authentication activity
* Troubleshot the RDP service when TCP 3389 was initially unavailable
* Generated controlled password-guessing activity from Kali Linux
* Identified Windows Event ID 4625 authentication failures
* Verified that the authentication activity reached Wazuh
* Simulated Windows Security log clearing
* Identified Windows Event ID 1102
* Verified Wazuh Rule 63103 for the log-clearing activity
* Mapped both scenarios to MITRE ATT&CK
* Validated the path from endpoint activity to SIEM visibility

## Key Takeaway

I used controlled attack simulations to generate known security activity and then traced the resulting evidence from the Windows endpoint into Wazuh.

This phase demonstrated **detection validation**.

The next phase focuses on what happens after the detection:

> **Investigate the evidence → assess risk → make a triage decision → recommend a response.**

---

## Next Phase

**Phase 4: SOC Investigation, Triage & Risk Assessment**

Phase 4 takes the detections generated here and examines them from an analyst perspective, including:

* Investigation workflow
* Evidence correlation
* Event timelines
* True-positive / false-positive analysis
* Risk assessment
* Potential business impact
* MITRE ATT&CK context
* Recommended response and remediation
* Detection gaps and limitations
