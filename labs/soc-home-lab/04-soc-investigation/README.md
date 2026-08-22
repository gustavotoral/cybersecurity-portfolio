# Phase 4: SOC Investigation, Triage & Risk Assessment

## Objective

The previous phases focused on building the lab, establishing telemetry, and generating controlled security activity.

This phase focuses on what happens **after a detection appears**:

> **What happened, what evidence supports that conclusion, how serious could it be, and what should happen next?**

I used the events generated during the lab to practice a basic SOC investigation workflow involving evidence correlation, timeline analysis, triage, risk assessment, and recommended response actions.

---

## 1. Investigation Workflow

For each security event, I followed the same general process:

```text
Alert / Event
      ↓
Validate the activity
      ↓
Identify source and target
      ↓
Review supporting evidence
      ↓
Reconstruct the event sequence
      ↓
Determine whether the activity is expected
      ↓
Assess risk and potential impact
      ↓
Classify the activity
      ↓
Recommend next steps
```

The main lesson was that a detection is not the conclusion.

It tells the analyst that something happened. The investigation determines **what it means**.

---

# 2. Investigation: RDP Password Guessing

## What Happened

Kali Linux generated repeated RDP authentication attempts against the Windows 11 endpoint.

Windows recorded the activity as failed authentication events.

### Evidence

| Evidence       | Finding                                 |
| -------------- | --------------------------------------- |
| Windows Event  | 4625 — Failed Logon                     |
| Logon Type     | 10 — Remote Interactive                 |
| Source IP      | `192.168.10.20`                         |
| Target Account | `soc_analyst`                           |
| Target         | Windows 11 endpoint                     |
| Service        | RDP / TCP 3389                          |
| Source System  | Kali Linux                              |
| Activity       | Repeated failed authentication attempts |

The source IP matched the Kali Linux system used for the controlled simulation, allowing me to connect the Windows authentication evidence back to the originating host.

The same authentication failures were also visible in Wazuh, confirming that the endpoint telemetry reached the SIEM.

---

## Timeline

```text
Kali Linux
    ↓
Repeated RDP authentication attempts
    ↓
Windows receives failed logons
    ↓
Windows generates Event ID 4625
    ↓
Wazuh Agent collects the events
    ↓
Events reach the Wazuh Manager
    ↓
Authentication failures become visible in Wazuh
    ↓
Analyst reviews the evidence
```

This helped me understand that a SIEM event is the result of an underlying telemetry chain rather than an isolated dashboard entry.

---

## MITRE ATT&CK

**T1110.001 — Password Guessing**

The simulated activity represented repeated password attempts against a remote authentication service.

---

## Analyst Assessment

**Classification:** True Positive — Malicious Behavior Simulated in Lab
**Confidence:** High

### Reasoning

The evidence showed:

* Repeated authentication failures
* Remote Interactive logon attempts
* The same target account
* A consistent source IP
* Activity originating from the known Kali test system
* An intentionally generated password-guessing pattern

A single failed login would not be enough to classify activity as malicious.

Possible benign explanations could include:

* A user repeatedly entering the wrong password
* Help desk troubleshooting
* An administrator testing connectivity
* A service or application using an outdated credential

In a production environment, I would verify whether the source, account, and activity were authorized before escalating.

---

## Risk Assessment

### Lab Risk: Low

The activity was intentionally generated against dedicated systems inside an isolated VirtualBox network.

There was no production business impact.

### Potential Production Risk: High

If similar activity occurred against an exposed production RDP service, successful credential compromise could provide unauthorized remote access to an endpoint.

Potential consequences could include:

* Unauthorized system access
* Credential theft
* Data exposure
* Malware or ransomware deployment
* Privilege escalation
* Lateral movement
* Business disruption

Actual risk would depend on factors such as internet exposure, MFA, account privileges, successful authentication, and the endpoint's access to other systems.

---

## Recommended Response

If this occurred in production, I would:

1. Validate the source IP and targeted account.
2. Determine whether any authentication attempt succeeded.
3. Review surrounding authentication activity and other systems targeted by the source.
4. Check the account's privileges and look for suspicious activity following the login attempts.
5. Restrict or block the source if confirmed malicious.
6. Reset or disable the account if compromise is suspected.
7. Escalate or isolate the endpoint if additional evidence indicates compromise.

Long-term controls could include MFA, restricted RDP exposure, VPN-based remote access, strong password policies, account lockout controls, and monitoring for repeated authentication failures.

---

# 3. Investigation: Windows Security Log Clearing

## What Happened

I simulated clearing the Windows Security event log using:

```powershell
wevtutil cl Security
```

Windows generated:

**Event ID 1102 — The audit log was cleared**

Wazuh received the event and generated **Rule 63103**.

### Evidence

| Evidence      | Finding                              |
| ------------- | ------------------------------------ |
| Target        | Windows 11 endpoint                  |
| Action        | Windows Security log cleared         |
| Windows Event | 1102                                 |
| Wazuh Rule    | 63103                                |
| MITRE ATT&CK  | T1070.001 — Clear Windows Event Logs |

---

## MITRE ATT&CK

**T1070.001 — Clear Windows Event Logs**

The technique is associated with **Defense Evasion** because an attacker may attempt to remove evidence of previous activity.

---

## Analyst Assessment

**Classification:** True Positive — Suspicious Defense Evasion Behavior Simulated in Lab
**Confidence:** High that the log was cleared; compromise itself is not established.

The event confirms that the Security log was cleared.

It does **not**, by itself, prove that an attacker compromised the endpoint.

In production, I would want to determine:

* Which account performed the action?
* Was it authorized?
* What command or process initiated it?
* What occurred immediately before the log was cleared?
* Were there other authentication, process, or network alerts associated with the host?

An important lesson from the test was:

> **An attempt to remove evidence can itself create evidence.**

---

## Risk Assessment

### Lab Risk: Low

The action was intentionally performed inside the isolated test environment.

### Potential Production Risk: High

Clearing security logs can remove valuable forensic evidence and make incident investigation more difficult.

Potential impact includes:

* Loss of local forensic evidence
* Reduced visibility into previous activity
* Difficulty reconstructing an attack timeline
* Difficulty identifying affected accounts or processes
* Increased difficulty determining the scope of an incident
* Malicious activity remaining undetected for longer

---

## Recommended Response

If this occurred in production, I would:

1. Identify the account responsible for clearing the log.
2. Determine whether the action was authorized.
3. Review activity immediately preceding the event.
4. Examine other telemetry sources for authentication, process, and network activity.
5. Preserve remaining evidence.
6. Escalate if the action cannot be explained by legitimate administration.

This also reinforced why centralized logging matters: deleting a local log should not remove the organization's only copy of security evidence.

---

# 4. Detection Gap: Nmap Reconnaissance

One of the most useful lessons from the lab came from activity that **did not generate the alert I expected**.

I ran Nmap reconnaissance from Kali against the Windows endpoint, but the scans did not automatically produce a corresponding Wazuh detection.

Initially, I expected the scan itself to appear in the SIEM.

Investigating why it did not helped me understand an important principle:

> **A SIEM can only analyze the telemetry available to it.**

The lab primarily relied on:

* Windows Security events
* Sysmon telemetry
* Wazuh Agent collection
* Wazuh correlation

That provided useful **endpoint visibility**, but it was not equivalent to having a dedicated network IDS inspecting traffic on the network.

This helped me distinguish between:

**Endpoint telemetry**

```text
Windows Events
Sysmon
Processes
Authentication
Host activity
```

and:

**Network telemetry**

```text
Packets
Connection attempts
Port scanning
Network reconnaissance
Traffic patterns
```

A future version of the lab could add Windows Firewall / Windows Filtering Platform auditing or a network IDS such as Suricata to provide greater visibility into reconnaissance activity.

---

# 5. Supporting Process Telemetry

The lab also included process-creation validation using Sysmon.

I generated controlled activity from PowerShell and verified that process information could be captured and analyzed, including:

* Executable name
* Process path
* Command-line activity
* User context
* Parent process

For example, I executed `whoami.exe` from PowerShell and observed the parent-child relationship:

```text
powershell.exe
      ↓
whoami.exe
```

This demonstrated how process telemetry can help an analyst determine **what executed and what launched it**.

The activity itself was benign and intentionally generated for telemetry validation. I did not treat `whoami.exe` alone as malicious.

Future testing could expand this into suspicious PowerShell and command-line behavior.

---

# 6. Key Lessons

This project changed how I think about security monitoring.

### Alerts Need Context

An event or alert is the starting point, not the final answer.

Source, account, timestamp, frequency, process information, surrounding activity, and expected behavior all contribute to the investigation.

### Risk Is Contextual

The same activity can have very different risk depending on where it occurs.

The simulated RDP attempts presented **low actual risk inside the isolated lab**, but similar activity against an exposed production system could represent high risk.

### Detection Depends on Visibility

The Nmap test demonstrated that security tools cannot reliably detect activity when the required telemetry is not being collected.

### Troubleshooting Is Part of Security Work

Throughout the project I had to troubleshoot networking, firewall behavior, service availability, telemetry collection, Wazuh connectivity, and detection expectations.

The repeated workflow was:

```text
Observe the problem
      ↓
Verify assumptions
      ↓
Identify the cause
      ↓
Make a targeted change
      ↓
Validate the result
```

---

# Phase 4 Summary

In this phase, I practiced moving from **detection to analyst decision-making**.

I:

* Correlated endpoint evidence with simulated attack activity
* Reconstructed event sequences
* Distinguished individual events from suspicious patterns
* Practiced true-positive and false-positive reasoning
* Mapped activity to MITRE ATT&CK
* Distinguished lab risk from potential production risk
* Considered business impact
* Developed investigation and response recommendations
* Identified a network visibility gap through Nmap testing
* Analyzed parent-child process telemetry
* Recognized the limits of the current lab environment

The overall workflow became:

```text
Detection
    ↓
Evidence
    ↓
Investigation
    ↓
Context
    ↓
Risk
    ↓
Decision
    ↓
Response
```

## Final Takeaway

The most valuable part of this project was not simply getting Wazuh to display security events.

It was learning to connect those events back to the underlying activity, determine what the evidence supports, recognize what the environment could not see, assess why the activity could matter in a real organization, and explain what I would investigate next.

---

## Future Improvements

This project primarily demonstrated **telemetry validation and SOC investigation**, not full detection engineering.

Future iterations could include:

* Custom Wazuh detection rules and tuning
* Windows Firewall / Windows Filtering Platform telemetry
* Suricata network monitoring
* Suspicious PowerShell and command-line simulations
* Account and privilege-change monitoring
* Active Directory attack simulations
* Additional false-positive testing

These are future iterations rather than requirements for the current project.
