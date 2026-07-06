# Phase 1: Infrastructure Deployment

## 1. Objective (What & Why)
I built an isolated, multi-node virtual lab network using VirtualBox to safely simulate an enterprise network without exposing the environment to the public internet. This ensures malware or attack simulations stay contained.

* **Subnet Range:** `192.168.10.0/24`
* **Network Mode:** VirtualBox Host-Only Adapter

---

## 2. Environment Setup (The Configuration)
The lab consists of two primary virtual machines deployed within the isolated subnet:
1. **Windows 11 Enterprise:** Serving as the target workstation for log collection.
2. **Kali Linux:** Serving as the controlled adversarial platform to launch simulations.

---

## 3. Verification (The Proof)
To confirm the environment was properly isolated and both endpoints had line-of-sight to each other, I verified the IP configurations:

### Hypervisor Network Bounds
![VirtualBox Network Settings](../assets/virtualbox_manager_settings.png)

### Kali Linux Subnet Verification
![Kali Linux IP Address Verification](../assets/kali_linux_ip_address_verification.png)