# Phase 2: Windows Telemetry & Wazuh Log Ingestion

## What I Built

In this phase, I configured the Windows 11 endpoint to generate detailed security telemetry and forward it to Wazuh.

My goal was simple:

> **Generate activity on Windows → collect the events → send them to Wazuh → verify that I could see them in the SIEM.**

The basic pipeline was:

```text
Windows 11
   ↓
Sysmon
   ↓
Wazuh Agent
   ↓
Wazuh Manager
   ↓
Wazuh Dashboard
```

This gave me a way to see what was happening on the Windows endpoint from a central location.

---

## 1. Install and Verify Sysmon

Windows already generates security logs, but I wanted more detail about processes and network activity.

I installed Microsoft's Sysmon to capture additional endpoint telemetry.

```powershell
.\Sysmon64.exe -i sysmonconfig.xml -accepteula
```

After installation, I verified that the Sysmon service was running:

```powershell
Get-Service -Name "Sysmon64"
```

**Sysmon Installation**

[View Screenshot](https://github.com/gustavotoral/cybersecurity-portfolio/blob/main/labs/soc-home-lab/assets/win11_sysmon_installation_success.png)

**Sysmon Service Running**

[View Screenshot](https://github.com/gustavotoral/cybersecurity-portfolio/blob/main/labs/soc-home-lab/assets/win11_powershell_verify_sysmon64_running.png)

### What I was looking for

I wanted to confirm that Windows was actually generating Sysmon events before trying to send them to Wazuh.

---

## 2. Install and Configure the Wazuh Agent

Next, I installed the Wazuh agent on the Windows endpoint and connected it to my Wazuh server.

```powershell
.\wazuh-agent-4.x.x.msi /q WAZUH_MANAGER="192.168.10.x" WAZUH_REGISTRATION_SERVER="192.168.10.x"

Start-Service -Name "Wazuh"
```

**Wazuh Agent Installation**

[View Screenshot](https://github.com/gustavotoral/cybersecurity-portfolio/blob/main/labs/soc-home-lab/assets/win11_powershell_wazuh_agent_installation.png)

I then configured the agent to collect events from the Sysmon operational log:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

The goal was to create a path from the Windows endpoint to the Wazuh server.

---

## 3. Verify the Wazuh Agent

After configuring the agent, I checked the Wazuh dashboard to confirm that the Windows endpoint was registered and communicating with the Wazuh server.

**Active Wazuh Agent**

[View Screenshot](https://github.com/gustavotoral/cybersecurity-portfolio/blob/main/labs/soc-home-lab/assets/wazuh_dashboard_active_agent_count.png)

This confirmed that the agent was connected.

---

## 4. Generate and Verify Windows Telemetry

Once the pipeline was working, I generated activity on the Windows endpoint and checked whether Sysmon recorded it.

### Process Creation — Sysmon Event ID 1

I ran commands on the Windows machine and verified that Sysmon recorded the process creation.

For example, I used `nslookup` and checked the resulting event.

The event showed information such as:

* Process name
* Process path
* Parent process
* User
* Command-line arguments

**Sysmon Event ID 1**

[View Screenshot](https://github.com/gustavotoral/cybersecurity-portfolio/blob/main/labs/soc-home-lab/assets/sysmon_event_1_process_creation_nslookup.png)

This helped me understand what information an analyst can use when investigating process activity.

---

### Network Connections — Sysmon Event ID 3

I also verified that Sysmon was recording network connections.

The event included information such as:

* Source and destination IP addresses
* Destination port
* Protocol
* Process responsible for the connection

**Sysmon Event ID 3**

[View Screenshot](https://github.com/gustavotoral/cybersecurity-portfolio/blob/main/labs/soc-home-lab/assets/sysmon_event_3_network_connection_detected.png)

This gave me visibility into which processes were creating network connections.

---

## 5. Verify the Full Pipeline

At this point, I had verified each part of the telemetry pipeline:

| Stage           | What I Verified               | Status   |
| --------------- | ----------------------------- | -------- |
| Windows         | Endpoint generating activity  | Verified |
| Sysmon          | Security events being created | Verified |
| Wazuh Agent     | Agent connected to manager    | Verified |
| Wazuh Manager   | Events being received         | Verified |
| Wazuh Dashboard | Events available for analysis | Verified |

The important part was not just installing each tool. I wanted to verify that **an action on the Windows endpoint resulted in an event that could be investigated from Wazuh.**

---

## What I Learned

This phase helped me understand the basic flow of endpoint telemetry.

Before building this lab, I mostly thought of a SIEM as a place where alerts appear. This helped me understand what happens before the alert exists:

**User/Process Activity → Windows Event → Sysmon → Wazuh Agent → Wazuh Manager → Analyst**

I also learned that troubleshooting telemetry requires checking each part of the pipeline instead of assuming the SIEM is the problem.

---

## Next Phase

With the telemetry pipeline working, I moved to the next phase: generating security events from the Kali Linux machine and investigating how those events appeared in Wazuh.
