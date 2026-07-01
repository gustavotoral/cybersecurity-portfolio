# Phase 1: Infrastructure Deployment & Network Topography

## Objective
To architect and configure an isolated, multi-node virtualized staging environment to simulate an enterprise local area network (LAN) for security monitoring and telemetry ingestion.

## Network Topology
The environment leverages a strict host-only network configuration to ensure zero exposure to the public internet while allowing seamless lateral communication and log aggregation between endpoints.



### Subnet Allocation
- **Network Mode:** VirtualBox Host-Only Adapter
- **Subnet Range:** `192.168.10.0/24`
- **Gateway/DHCP:** Managed via virtual hypervisor DHCP engine

### Node Registry
| Node Name | Operating System | Private IP Address | Primary Function |
| :--- | :--- | :--- | :--- |
| **Wazuh-Server** | Ubuntu Server 22.04 LTS | `192.168.10.x` | SIEM Indexer, Manager, & Dashboard |
| **Windows-Endpoint** | Windows 10 Enterprise | `192.168.10.10` | Target Node / Telemetry Source |
| **Kali-Attacker** | Kali Linux (Rolling) | `192.168.10.y` | Threat Emulation & Penetration Testing |

---

## Connectivity & Routing Verification

### 1. Layer 3 ICMP Verification
To validate that the virtual switch is properly handling communication across the `192.168.10.0/24` plane, bidirectional ICMP echo requests were completed. 
```cmd
# Executed from Windows-Endpoint to verify path to Wazuh-Server
ping 192.168.10.x