# Phase 2: Telemetry Ingestion & Endpoint Monitoring

## 1. Objective (What & Why)
The objective of this phase was to establish deep visibility on the Windows 11 endpoint and centralize log collection. I deployed Sysmon to capture advanced host-level telemetry and configured a Wazuh agent to securely ship these logs to a central SIEM management console. This setup ensures that malicious activity cannot happen in a vacuum.

---

## 2. Environment Setup (The Configuration)
* **Telemetry Engine:** Sysmon (System Monitor) with a customized configuration file to filter out noise and focus on high-fidelity security events.
* **SIEM Platform:** Wazuh (Open-source Security Monitoring Platform).
* **Log Pipeline:** Windows Event Logs + Sysmon Logs ➡️ Wazuh Agent ➡️ Wazuh Manager.

---

## 3. Verification (The Proof)
To confirm telemetry was actively flowing and the endpoint was successfully monitored, I verified the agent status and local event logging:

### SIEM Agent Connectivity
Below is the verified connection status showing the Windows 11 workstation actively checking into the centralized Wazuh management console:

![Wazuh Agent Summary](../assets/wazuh_agents_summary.png)

### Host-Level Logging Verification
I verified that Sysmon was successfully hooks-installed and generating granular event IDs on the local endpoint:

* **Event ID 1 (Process Creation):** Confirming execution monitoring is active.
![Sysmon Event 1 Process Creation](../assets/sysmon_event_1_process_creation.png)

* **Event ID 3 (Network Connection):** Confirming network sockets opened by processes are being logged.
![Sysmon Event 3 Network Connection](../assets/sysmon_event_3_network_connection.png)