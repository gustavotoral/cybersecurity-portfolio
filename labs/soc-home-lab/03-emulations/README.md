# Phase 3: Attack Emulation & Detection Triage

## Objective
To simulate an automated brute-force attack against an enterprise asset over Remote Desktop Protocol (RDP) and verify SIEM alert correlation rules.

## Attack Vector
- **Attacker Node:** Kali Linux (`192.168.10.20`)
- **Target Node:** Windows 10 Endpoint (`192.168.10.10`)
- **Tooling Used:** Hydra (FreeRDP module)

## Detection & Triage Workflow
During testing, initial connection failures were isolated to a disabled OS-level listening port (3389). Following targeted troubleshooting and system remediation, the port was successfully opened to a listening state.

An automated brute-force wave was executed. The SIEM (Wazuh) successfully ingested raw Windows Security logs (**Event ID 4625: An account failed to log on**).

### The Correlation Signature
While individual logon failures were logged, Wazuh's correlation engine successfully triggered **Rule 60122: Multiple Windows Logon Failure** once the velocity threshold was breached.

- **Identified Attacker IP:** `192.168.10.20`
- **Logon Type:** 10 (RemoteInteractive / RDP)
- **SIEM Action:** Escalated to high-priority security alert.
