# Phase 3: Adversary Simulation & Alert Investigation

## What I Built

In this phase, I used Kali Linux to simulate attacks against my Windows 11 endpoint and then investigated the resulting security events in Wazuh.

I wanted to answer a simple question:

> **If an attacker tries to break into my Windows machine, what evidence will they leave behind, and will Wazuh detect it?**

I tested two scenarios:

1. RDP brute-force attempts
2. Windows Security event log clearing

---

## 1. Prepare the Windows Endpoint

I first checked whether Remote Desktop was accessible from the Kali Linux machine.

From Kali, I scanned TCP port 3389:

```bash
nmap -p 3389 192.168.10.10
```

The first scan showed that port 3389 was not accepting connections.

**Nmap Scan**

[View Screenshot](https://github.com/gustavotoral/cybersecurity-portfolio/blob/main/labs/soc-home-lab/assets/kali_nmap_scan_rdp_port_3389.png)

### Troubleshooting

I enabled Remote Desktop on the Windows test machine and then verified that TCP 3389 was listening:

```powershell
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0

Get-NetTCPConnection -LocalPort 3389 | Select-Object LocalAddress, LocalPort, State
```

**RDP Listening**

[View Screenshot](https://github.com/gustavotoral/cybersecurity-portfolio/blob/main/labs/soc-home-lab/assets/win11_powershell_verify_rdp_listening.png)

This confirmed that the endpoint was ready for the test.

---

# 2. Simulate an RDP Brute-Force Attack

I used Hydra from Kali to generate repeated RDP login attempts against the Windows endpoint.

```bash
hydra -l soc_analyst -P /usr/share/wordlists/rockyou.txt rdp://192.168.10.10 -V -t 4
```

**Hydra Brute-Force Attempt**

[View Screenshot](https://github.com/gustavotoral/cybersecurity-portfolio/blob/main/labs/soc-home-lab/assets/kali_hydra_rdp_brute_force_attack.png)

The goal was not to gain access. I wanted to generate the same type of failed authentication activity that a SOC analyst might investigate.

---

# 3. Investigate the Windows Security Events

The failed login attempts generated Windows Security Event ID **4625**.

Event 4625 means that an account failed to log on.

I reviewed the event to understand what happened and where the activity came from.

The event showed:

* **Event ID:** 4625
* **Logon Type:** 10 — Remote Interactive
* **Source IP:** `192.168.10.20`
* **Target Account:** `soc_analyst`

The source IP matched my Kali machine, confirming that the failed logins came from the attack simulation.

**Windows Event ID 4625**

[View Screenshot](https://github.com/gustavotoral/cybersecurity-portfolio/blob/main/labs/soc-home-lab/assets/windows_event_viewer_security_id_4658.png)

### What I Learned

A single failed login doesn't necessarily mean an attack is happening.

The important part was looking at the **pattern**:

> Multiple failed remote logins from the same source in a short period of time.

That pattern is much more suspicious than one incorrect password.

---

# 4. Investigate the Wazuh Alert

The Windows events were collected by the Wazuh agent and sent to the Wazuh server.

Wazuh then generated an alert based on the repeated authentication failures.

The alert I observed was:

* **Rule:** 60122
* **Severity:** Level 10
* **Activity:** Multiple Windows logon failures
* **Source:** `192.168.10.20`

**Wazuh Brute-Force Alert**

[View Screenshot](https://github.com/gustavotoral/cybersecurity-portfolio/blob/main/labs/soc-home-lab/assets/wazuh_alert_brute_force_logon_failures.png)

This allowed me to follow the activity from the original Windows event to the SIEM alert.

### My Triage

Based on the evidence, I would classify this as a **True Positive security event** within the lab.

**Reasoning:**

* The source IP belonged to my Kali attack machine.
* The activity targeted Remote Desktop.
* Multiple authentication failures occurred in a short period.
* The activity was generated intentionally using Hydra.

---

# 5. Simulate Log Clearing

For the second test, I wanted to see what would happen if someone attempted to remove evidence from the Windows Security log.

I used:

```powershell
wevtutil cl Security
```

This generated Windows Security Event ID **1102**, which indicates that the audit log was cleared.

Wazuh detected the activity and generated an alert.

**Wazuh Log Clearing Alert**

[View Screenshot](https://github.com/gustavotoral/cybersecurity-portfolio/blob/main/labs/soc-home-lab/assets/wazuh_alert_rule_63103_audit_log_cleared.png)

The activity maps to MITRE ATT&CK **T1070.001 — Clear Windows Event Logs**.

### Why This Matters

An attacker may try to remove evidence after gaining access to a system.

This test showed me that the attempt to clear the Windows Security log itself creates a detectable event.

---

# 6. What I Learned From the Investigation

This phase changed how I think about SIEM alerts.

Before this lab, I mostly thought about alerts as something that simply appears in a dashboard.

Now I understand the basic investigation process:

```text
Attack Activity
      ↓
Windows Event
      ↓
Wazuh Agent
      ↓
Wazuh Manager
      ↓
Detection Rule
      ↓
Alert
      ↓
Analyst Investigation
```

I also learned that **the alert is only the starting point**.

An analyst still needs to look at things like:

* What happened?
* Which system was targeted?
* What account was involved?
* Where did the activity come from?
* How many times did it happen?
* Does the activity make sense in the environment?
* Is this a true positive or a false positive?

---

# Phase 3 Summary

| Test            | Evidence              | Result              |
| --------------- | --------------------- | ------------------- |
| RDP brute force | Windows Event ID 4625 | Detected            |
| RDP brute force | Wazuh Rule 60122      | High-severity alert |
| Log clearing    | Windows Event ID 1102 | Detected            |
| Log clearing    | Wazuh Rule 63103      | Alert generated     |

## Key Takeaway

I used Kali Linux to generate attack activity against a Windows endpoint and followed the resulting evidence from the endpoint logs into Wazuh.

The biggest lesson was that effective alert investigation requires more than recognizing a rule name. I had to connect the **source IP, account, event type, timestamps, and attack behavior** to determine what actually happened.

## Next Steps

The next iteration of this lab will focus on improving the detection itself rather than only testing existing Wazuh rules.

Planned improvements:

* Create a custom Wazuh detection rule.
* Tune an existing rule to reduce false positives.
* Add additional attack simulations.
* Write a standardized SOC incident report for each investigation.
* Explore using AI to summarize alerts while manually validating its conclusions.
