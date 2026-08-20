# Phase 1: Virtual Infrastructure & Isolated Network Architecture

## 1. Architectural Objective & Threat Model
The primary objective was to deploy a segregated, virtual security operations environment to safely conduct adversarial simulations, observe live telemetry generation, and validate SIEM alert pipelines without risking production networks or public exposure.

* **Network Containment:** Configured VirtualBox Internal Network (`soc-net`) to eliminate inadvertent outbound traffic or accidental internet routability.
* **Subnet Architecture:** `192.168.10.0/24` non-routable private address space.
* **Core Nodes:**
  * **Adversarial Platform:** Kali Linux (`192.168.10.x`) — Dedicated simulation node for reconnaissance, brute-force, and persistence techniques.
  * **Monitored Endpoint:** Windows 11 Enterprise (`192.168.10.x`) — Workstation configured for granular process, authentication, and network telemetry capture.
  * **Detection Engine:** Wazuh SIEM Server (`192.168.10.x`) — Centralized log ingestion, correlation engine, and alert dashboard.

---

## 2. Infrastructure Deployment & Hypervisor Configuration

To ensure strict network segmentation, all virtual interfaces were decoupled from default NAT/Bridged adapters and assigned to an isolated internal switch.

![VirtualBox Network Configuration](../assets/virtualbox_vm_settings_internal_network.png)
*Figure 1.1: VirtualBox hypervisor internal network boundary assignment.*

![VirtualBox VM Inventory](../assets/virtualbox_manager_vm_inventory_summary.png)
*Figure 1.2: Multi-node lab topology inventory.*

---

## 3. Connectivity Baseline & Network Troubleshooting

### Step 1: Endpoint IP & Subnet Verification
Verified static/DHCP assignments across both hosts to ensure mutual Layer 3 visibility on `192.168.10.0/24`.

* **Kali Linux:**
  ```bash
  ip a show eth0
  ```
  ![Kali IP Verification](../assets/kali_linux_ip_address_verification.png)

* **Windows 11 Endpoint:**
  ```powershell
  ipconfig /all
  ```
  ![Windows IP Verification](../assets/win11_ipconfig_network_verification.png)

---

### Step 2: Protocol Reachability & Host Firewall Triage

* **Initial Observation:** An ICMP ping sweep from Kali to the Windows 11 endpoint dropped 100% of packets despite valid Layer 3 addressing.

![ICMP Drop](../assets/kali_ping_failure_icmp_blocked.png)
*Figure 1.3: Host unreachable due to default ingress policy.*

* **Root Cause Analysis:** Default Windows Defender Firewall rules drop inbound ICMPv4 Echo Requests (`Protocol 1`) across all network profiles to prevent unauthorized network discovery.
* **Mitigation & Validation:** Rather than disabling host-based defenses entirely, an explicit inbound firewall rule was applied to permit ICMPv4 echo requests strictly from the internal subnet for health-checking and baseline validation.

```powershell
# Enable ICMPv4 Echo Request on Windows 11
netsh advfirewall firewall add rule name="Allow ICMPv4-In" protocol=icmpv4:8,any dir=in action=allow
```

![ICMP Success](../assets/kali_ping_success_after_firewall_adjustment.png)
*Figure 1.4: Verified bidirectional connectivity following least-privilege host firewall rule creation.*

---

## 4. Operational Readiness Summary

| Node | Hostname / OS | Role | Validation Status |
| :--- | :--- | :--- | :--- |
| **Adversary** | Kali Linux | Attack Simulation & Reconnaissance | `[OPERATIONAL]` |
| **Endpoint** | Windows 11 Pro | Monitored Host (Telemetry Target) | `[OPERATIONAL]` |
| **SIEM** | Wazuh Manager | Log Collection & Detection Engine | `[OPERATIONAL]` |

**Key Takeaway:** Establishing an isolated internal virtual network prevents unintentional payload spillover, while surgically configuring host-based firewall rules mirrors enterprise defense-in-depth principles without compromising the testing environment.
