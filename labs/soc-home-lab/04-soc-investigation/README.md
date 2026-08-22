# SOC Investigation, Triage & Risk Assessment (Deep Dive)

## Purpose

The previous phases focused on building the environment, collecting telemetry, and generating security events.

This phase focuses on the analyst's job:

> **What happened, how serious is it, what evidence supports the conclusion, and what should happen next?**

I used the alerts generated during the lab to practice a basic SOC investigation workflow, including event validation, timeline analysis, risk assessment, triage, and recommended response actions.

---

## 1. Investigation Workflow

For each security event, I followed the same basic process:

```text
Alert
  ↓
Validate the alert
  ↓
Identify the source and target
  ↓
Review the timeline
  ↓
Correlate supporting evidence
  ↓
Assess risk
  ↓
Determine True Positive / False Positive
  ↓
Recommend next steps
```

This helped me move beyond simply recognizing an alert and practice explaining what the alert means.

A detection tells the analyst that something happened. The investigation determines what it means.

---

## 2. Investigation: RDP Brute Force

### What Happened

Kali Linux generated repeated RDP login attempts against the Windows 11 endpoint.

The activity generated Windows Security Event ID **4625**, indicating failed logon attempts.

The relevant evidence included:

| Evidence       | Finding                                 |
| -------------- | --------------------------------------- |
| Event ID       | 4625 — Failed Logon                     |
| Logon Type     | 10 — Remote Interactive                 |
| Source IP      | `192.168.10.20`                         |
| Target Account | `soc_analyst`                           |
| Target         | Windows 11 endpoint                     |
| Protocol       | RDP                                     |
| Port           | TCP 3389                                |
| Attack Source  | Kali Linux                              |
| Activity       | Repeated failed authentication attempts |

The source IP matched the Kali Linux machine used for the attack simulation.

This allowed me to connect the SIEM alert back to the activity generated from the attack system.

### Timeline

The activity followed this sequence:

```text
Kali reconnaissance
      ↓
RDP port 3389 made available
      ↓
Hydra generated repeated login attempts
      ↓
Windows generated Event ID 4625
      ↓
Wazuh Agent collected the events
      ↓
Wazuh processed the authentication failures
      ↓
Wazuh generated a high-severity alert
      ↓
Analyst reviewed the evidence
```

The important lesson was that the SIEM alert was the end result of multiple events moving through the telemetry pipeline.

### Risk Assessment

#### Lab Risk

**Low**

The activity was intentionally generated against dedicated systems inside an isolated VirtualBox internal network.

Although the simulated behavior represented a credential attack, there was no production business impact.

#### Potential Production Risk

**High**

If the same activity occurred against an internet-facing RDP service, a successful password attack could provide unauthorized remote access to an endpoint.

Potential impact could include:

* Unauthorized system access
* Malware deployment
* Credential theft
* Data access or theft
* Privilege escalation
* Lateral movement
* Ransomware deployment
* Disruption to business operations

The actual risk would depend on factors such as whether the account was compromised, whether MFA was enabled, whether RDP was exposed to the internet, what privileges the account had, and what network access the endpoint provided.

### Triage Decision

**Classification:** True Positive — Malicious Activity Simulated in Lab

The activity was classified as malicious behavior because:

1. Multiple authentication failures occurred within a short period.
2. The attempts targeted a remote access service.
3. The source IP matched the known Kali attack system.
4. The activity was generated using an automated password-guessing tool.
5. The failures occurred repeatedly against the same account.

A single failed login could easily represent normal user behavior. Repeated remote authentication failures from the same source are significantly more suspicious.

In a real environment, I would still verify whether the source system and activity were authorized before escalating the alert.

### Recommended Response

If this occurred in production, initial investigation and response actions could include:

1. Confirm whether the source IP belongs to an authorized system.
2. Identify the targeted account.
3. Determine whether any login attempt succeeded.
4. Review additional authentication events for the account.
5. Check whether the account has elevated privileges.
6. Review activity from the source IP against other systems or accounts.
7. Determine whether additional endpoints were targeted.
8. Consider blocking or restricting the source if confirmed malicious.
9. Reset credentials if compromise is suspected.
10. Isolate the endpoint if evidence of compromise exists.
11. Escalate according to the organization's incident response process.

Long-term defensive controls could include MFA, restricted RDP exposure, account lockout policies, stronger password requirements, VPN-based remote access, and monitoring for repeated authentication failures.

---

## 3. Investigation: Windows Security Log Clearing

### What Happened

I simulated an attempt to remove evidence from the Windows Security log:

```powershell
wevtutil cl Security
```

The activity generated:

**Windows Security Event ID 1102 — The audit log was cleared**

Wazuh detected the activity and generated an alert.

### MITRE ATT&CK

**T1070.001 — Clear Windows Event Logs**

This technique represents an attempt to remove security evidence and is associated with defense evasion activity.

### Evidence Collected

| Evidence      | Finding                              |
| ------------- | ------------------------------------ |
| Target        | Windows 11 endpoint                  |
| Action        | Windows Security log cleared         |
| Windows Event | Event ID 1102                        |
| Wazuh Rule    | 63103                                |
| Activity      | Security event log clearing          |
| MITRE ATT&CK  | T1070.001 — Clear Windows Event Logs |

### Risk Assessment

#### Lab Risk

**Low**

The command was intentionally executed inside the isolated lab environment.

The test demonstrated that an attempt to remove security evidence generated its own detectable event.

#### Potential Production Risk

**High**

Clearing security logs can interfere with an investigation and may indicate an attempt to hide previous activity.

Potential impact includes:

* Loss of forensic evidence
* Reduced visibility into previous activity
* Difficulty reconstructing an attack timeline
* Difficulty identifying affected accounts or processes
* Increased difficulty determining whether other systems were affected
* Increased likelihood of malicious activity remaining undetected

### Triage Decision

**Classification:** True Positive — Suspicious Defense Evasion Activity Simulated in Lab

The event itself does **not** prove that an attacker compromised the system.

However, clearing the Windows Security log is significant behavior and should be investigated, especially if it occurs outside an approved administrative or maintenance procedure.

The analyst should determine:

* Who performed the action?
* Which account was used?
* Was the activity authorized?
* What process or command initiated it?
* What occurred immediately before the log was cleared?
* Was the endpoint already associated with another security alert?

An important lesson from this test was:

> **An attempt to remove evidence can itself create evidence.**

### Recommended Response

If this occurred in production, I would:

1. Identify the account responsible for clearing the log.
2. Determine whether the action was authorized.
3. Review events immediately before the log-clearing activity.
4. Review other available telemetry sources.
5. Check for suspicious authentication activity.
6. Check for suspicious processes or command execution.
7. Review network connections and persistence indicators.
8. Preserve available evidence before making major changes to the system.
9. Escalate if the activity cannot be explained by legitimate administration.

Centralized logging is also important because deleting a local event log should not remove the organization's only copy of security evidence.

---

## 4. Risk-Based Triage

Not every alert should receive the same response.

Rather than relying only on the severity number assigned by the SIEM, I would consider additional context:

| Factor          | Question                                           |
| --------------- | -------------------------------------------------- |
| Severity        | How serious could the activity be?                 |
| Confidence      | How strong is the evidence?                        |
| Asset           | What system was targeted?                          |
| Account         | What user or service account was involved?         |
| Source          | Where did the activity originate?                  |
| Frequency       | Was this a single event or repeated behavior?      |
| Business Impact | What could happen if the activity were successful? |
| Authorization   | Could this be legitimate administrative activity?  |

For example, one failed login is not automatically malicious.

I would look at frequency, source IP, target account, time of activity, authentication type, surrounding events, and whether the behavior was expected before deciding how to classify it.

Potential false-positive or benign explanations for repeated authentication failures could include:

* A user repeatedly entering an incorrect password
* Help desk troubleshooting
* An automated service using an expired credential
* An administrator testing connectivity
* A recently changed password still being used by another application or device

The purpose of triage is to add context to the alert before deciding what action should be taken.

---

## 5. MITRE ATT&CK Mapping

The simulated activity was mapped to the following MITRE ATT&CK techniques:

| Activity                   | MITRE ATT&CK                             | Purpose                     |
| -------------------------- | ---------------------------------------- | --------------------------- |
| RDP password guessing      | **T1110.001 — Password Guessing**        | Simulated credential attack |
| Windows event log clearing | **T1070.001 — Clear Windows Event Logs** | Simulated defense evasion   |

MITRE ATT&CK mapping helped connect the technical events to known attacker behaviors rather than treating each alert as an isolated log entry.

---

## 6. Detection Gaps & Visibility

One of the most valuable lessons from the lab was discovering what the environment **did not** detect automatically.

I tested Nmap reconnaissance from Kali against the Windows endpoint, but the scan did not automatically produce the type of Wazuh alert I initially expected.

This highlighted an important limitation of the current environment:

> **A SIEM cannot reliably detect telemetry that the environment is not collecting.**

The lab primarily relied on:

* Windows Security events
* Sysmon telemetry
* Wazuh Agent collection
* Wazuh SIEM correlation

This provided endpoint visibility but did not provide the same visibility as a dedicated network intrusion detection sensor inspecting network traffic.

The experience helped me better understand the distinction between:

* Host telemetry
* Windows Event Logs
* Sysmon telemetry
* SIEM correlation
* Network telemetry
* Network IDS monitoring

A future version of the lab could add:

* Windows Firewall logging
* Windows Filtering Platform events
* Suricata
* Snort
* Additional Wazuh rules for network-related activity

This would allow me to compare what an endpoint detects with what a network sensor detects.

---

## 7. Detection Engineering Roadmap

The current lab primarily demonstrated **detection validation** rather than full detection engineering.

A future iteration would focus on improving and tuning the detections themselves.

### Authentication Detection

Potential improvements include:

* Create or tune a custom Wazuh rule for repeated authentication failures
* Correlate failures from the same source IP
* Consider the target account and endpoint
* Define a meaningful frequency threshold
* Define an appropriate time window
* Test the rule using controlled attack simulations
* Evaluate potential false positives
* Adjust thresholds based on results

A basic detection engineering cycle would look like:

```text
Observe existing detection
        ↓
Identify detection limitation
        ↓
Modify or create rule
        ↓
Generate controlled test activity
        ↓
Validate detection
        ↓
Evaluate false positives
        ↓
Tune threshold / timeframe
        ↓
Retest
```

### Log Tampering Detection

Additional context around Event ID 1102 could include:

* User account
* Process
* Command line
* Parent process
* Authentication activity
* Events immediately before the log was cleared

### Additional Telemetry

Future improvements could include:

* Expanded Sysmon coverage
* Windows Firewall telemetry
* Windows Filtering Platform events
* Network IDS telemetry

### Additional Attack Simulations

Future simulations could include:

* Network reconnaissance
* Suspicious PowerShell activity
* Account creation
* Privilege changes
* Suspicious process execution
* Persistence techniques

---

## 8. SOC Ticket Documentation Example

A simplified SOC ticket based on the RDP investigation could look like this:

| Field                  | Details                                                                                                                                                           |
| ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Alert**              | Multiple Windows Logon Failures                                                                                                                                   |
| **Severity**           | High                                                                                                                                                              |
| **Affected Host**      | Windows 11 endpoint                                                                                                                                               |
| **Source**             | `192.168.10.20`                                                                                                                                                   |
| **Account**            | `soc_analyst`                                                                                                                                                     |
| **Evidence**           | Windows Event ID 4625, Logon Type 10, followed by Wazuh detection                                                                                                 |
| **Assessment**         | True Positive security activity. Repeated RDP authentication failures originated from the lab's Kali Linux attack host.                                           |
| **Impact**             | No production impact. Activity occurred within an isolated test environment.                                                                                      |
| **Recommended Action** | In production, investigate the source, verify whether the account was compromised, review successful authentication events, and restrict unauthorized RDP access. |
| **Status**             | Closed — Lab Simulation                                                                                                                                           |

This exercise helped reinforce the importance of documenting not only what the SIEM detected, but also the evidence, analyst assessment, potential impact, and recommended action.

---

## 9. What I Learned

### Troubleshooting Is Part of Security Work

The lab did not work perfectly on the first attempt.

Throughout the project, I had to troubleshoot:

* Virtual network configuration
* NAT vs. Internal Network connectivity
* Static IP addressing
* Windows Firewall behavior
* ICMP connectivity
* Service availability
* RDP connectivity
* Wazuh Agent registration
* Log collection
* Detection behavior

Each issue forced me to verify assumptions rather than simply follow instructions.

### Logs Need Context

A failed login event by itself does not necessarily indicate an attack.

Useful investigation context includes:

* Source IP
* Target account
* Timestamp
* Logon type
* Frequency
* Target system
* Surrounding authentication activity
* Related endpoint events

The context surrounding an event determines whether it represents normal behavior or something worth investigating.

### Detection Depends on Visibility

The Nmap test reinforced that detection depends on the telemetry available to the security tools.

If the environment does not collect the necessary data, the SIEM cannot reliably identify the activity.

### Alerts Require Investigation

An alert is not the investigation.

It is the starting point.

The analyst still needs to determine:

* What happened?
* When did it happen?
* Where did it originate?
* What system or account was affected?
* What evidence supports the conclusion?
* Is the activity malicious, suspicious, or legitimate?
* What is the potential business impact?
* What should happen next?

### Detection Improvement Is Iterative

Security monitoring is not simply about enabling a rule and assuming the problem is solved.

Detections should be tested against known activity, evaluated for useful signal, reviewed for false positives, and improved over time.

---

## 10. Final Project Outcome

I built an isolated SOC home lab using **VirtualBox, Kali Linux, Windows 11, Sysmon, and Wazuh**.

During the project, I:

* Built and troubleshot an isolated virtual network
* Configured static addressing and endpoint connectivity
* Troubleshot Windows Firewall behavior
* Configured Windows endpoint telemetry
* Connected the Windows endpoint to Wazuh
* Verified security event ingestion
* Generated controlled attack activity
* Investigated Windows authentication events
* Investigated Windows Security log-clearing activity
* Validated Wazuh detections
* Mapped simulated activity to MITRE ATT&CK
* Assessed potential production business impact
* Practiced risk-based alert triage
* Developed recommended response actions
* Identified visibility and detection gaps
* Practiced SOC-style documentation

The project gave me practical experience following security activity through the workflow:

```text
Build
  ↓
Troubleshoot
  ↓
Collect Telemetry
  ↓
Generate Activity
  ↓
Detect
  ↓
Investigate
  ↓
Assess Risk
  ↓
Recommend Response
  ↓
Identify Improvements
```

More importantly, the project helped me understand that operating a SIEM is only one part of SOC work.

The analyst must be able to connect an alert to the underlying evidence, determine what the activity means, evaluate its potential impact, communicate the findings clearly, and recommend what should happen next.

---

## Current Limitations

This is a personal home lab and does not replicate the scale or complexity of a production SOC environment.

The project did not include:

* A production Active Directory environment
* Enterprise EDR
* SOAR automation
* Multiple production endpoints
* A dedicated network IDS
* Large-scale log volume
* Production incident response procedures

These limitations also provide clear opportunities for future iterations of the lab.

---

## Future Improvements

Potential future improvements include:

* Custom Wazuh detection rules
* Detection tuning
* False-positive analysis
* Suricata network monitoring
* Windows Firewall telemetry
* Active Directory attack simulations
* Additional MITRE ATT&CK techniques
* Standardized incident reports
* Automated alert enrichment
* AI-assisted alert summarization with analyst validation

The goal of future iterations is to continue moving from **building security tools** toward **using security telemetry to investigate, understand, and respond to security problems**.
