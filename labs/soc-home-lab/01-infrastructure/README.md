# Phase 1: Virtual Infrastructure & Isolated Network

## What I Built

I built an isolated virtual network in VirtualBox for my home SOC lab. The goal was to safely generate security events, collect logs, and test detection without exposing the lab to my home network or the internet.

The lab contains three virtual machines:

* **Kali Linux** — used to simulate attacks and reconnaissance
* **Windows 11** — monitored endpoint where security events are generated
* **Ubuntu Server / Wazuh** — collects and analyzes security logs

### Lab Network

* **Network:** `soc-net`
* **Subnet:** `192.168.10.0/24`
* **Hypervisor:** VirtualBox

---

## 1. Build the Virtual Network

I configured the three VMs to communicate through a VirtualBox **Internal Network** called `soc-net`.

This keeps the lab isolated from my normal network while still allowing the virtual machines to communicate with each other.

**VirtualBox Network Configuration**

[View Screenshot](https://github.com/gustavotoral/cybersecurity-portfolio/blob/main/labs/soc-home-lab/assets/virtualbox_vm_settings_internal_network.png)

**VM Inventory**

[View Screenshot](https://github.com/gustavotoral/cybersecurity-portfolio/blob/main/labs/soc-home-lab/assets/virtualbox_manager_vm_inventory_summary.png)

---

## 2. Verify Network Connectivity

Before moving on to the security monitoring portion of the lab, I needed to make sure the machines could communicate.

### Check IP Addresses

On Kali Linux, I used:

```bash
ip a show eth0
```

On Windows, I used:

```powershell
ipconfig /all
```

I verified that the systems were receiving addresses on the `192.168.10.0/24` network.

**Kali IP Verification**

[View Screenshot](https://github.com/gustavotoral/cybersecurity-portfolio/blob/main/labs/soc-home-lab/assets/kali_linux_ip_address_verification.png)

**Windows IP Verification**

[View Screenshot](https://github.com/gustavotoral/cybersecurity-portfolio/blob/main/labs/soc-home-lab/assets/win11_ipconfig_network_verification.png)

---

## 3. Troubleshooting: Windows Firewall Blocking Ping

My first connectivity test failed.

Kali could not successfully ping the Windows endpoint, even though both systems were on the same subnet.

**Initial Result**

[View Screenshot](https://github.com/gustavotoral/cybersecurity-portfolio/blob/main/labs/soc-home-lab/assets/kali_ping_failure_icmp_blocked.png)

### What I Found

Windows Defender Firewall was blocking inbound ICMP traffic.

Instead of disabling the firewall, I created an inbound rule to allow ICMPv4 traffic for testing.

```powershell
netsh advfirewall firewall add rule name="Allow ICMPv4-In" protocol=icmpv4:8,any dir=in action=allow
```

I then tested connectivity again from Kali.

**Successful Ping**

[View Screenshot](https://github.com/gustavotoral/cybersecurity-portfolio/blob/main/labs/soc-home-lab/assets/kali_ping_success_after_firewall_adjustment.png)

### What I Learned

This was a useful reminder that having two systems on the same subnet does not automatically mean they can communicate.

I had to check:

1. IP addressing
2. Network configuration
3. Host firewall rules

This is the same basic troubleshooting process I would use when investigating connectivity problems on a real endpoint.

---

## 4. Lab Status

| System         | Role                    | Status      |
| -------------- | ----------------------- | ----------- |
| Kali Linux     | Attack simulation       | Operational |
| Windows 11     | Monitored endpoint      | Operational |
| Ubuntu / Wazuh | SIEM and log collection | Operational |

At this point, the virtual infrastructure was ready for the next phase: **collecting endpoint telemetry and sending it to Wazuh.**

---

## Key Takeaways

* Built an isolated VirtualBox network for security testing.
* Configured three virtual machines to communicate on the same subnet.
* Verified IP addressing and connectivity using Linux and Windows command-line tools.
* Troubleshot a Windows firewall issue that prevented ICMP connectivity.
* Created a targeted firewall rule instead of disabling the firewall.
* Established the foundation for the Wazuh monitoring environment.

**Next Phase:** Configure Windows telemetry and verify that security events are being collected by Wazuh.
