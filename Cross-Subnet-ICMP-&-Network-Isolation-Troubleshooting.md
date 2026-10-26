# Active Directory Lab: Cross-Subnet ICMP & Network Isolation Troubleshooting

## 1. Project Overview & Objective

This project covers the deployment, isolation, and troubleshooting of a multi-VLAN Active Directory environment in my home lab.

The goal was to establish a secure network topology using an OPNsense firewall, a TP-Link SG105 managed switch, and a TP-Link Wireless Access Point (WAP) broadcasting separate SSIDs mapped to individual VLANs.

A dedicated **Attack Simulation VLAN** was provisioned alongside the **Enterprise Active Directory Production VLAN** to simulate adversarial activity and test defensive capabilities.

---

## 2. The Troubleshooting Challenge

### Phase 1: Managed Switch Trunking & Lockouts

During the initial configuration of the TP-Link WAP through a direct Ethernet connection, the management interface was assigned to VLAN 10 (Admin).

This immediately caused a lockout because the laptop's network interface lacked the required 802.1Q tagging configuration to communicate with the newly tagged management interface.

**Resolution**

The switch interface (Port 4) connecting to the WAP was reconfigured as a hybrid/trunk port.

- **VLAN 10:** Configured as Untagged with a Port VLAN ID (PVID) of 10, allowing untagged management frames from the WAP to enter the Admin network correctly.
- **Client Wi-Fi VLANs:** VLANs 20, 30, and others were configured as Tagged on Port 4, allowing the WAP to segregate multiple SSIDs.

![](https://github.com/k3rn3l-p4n1k-l4b/Enterprise-Threat-Detection-Lab/blob/main/screenshots/Active-Directory-Config-Changes/1Tplinkpvidsettings.png)
![](https://github.com/k3rn3l-p4n1k-l4b/Enterprise-Threat-Detection-Lab/blob/main/screenshots/Active-Directory-Config-Changes/2tplinkportrules.png)

### Phase 2: Inter-VLAN Routing & Perimeter Defense Log Verification

Once the trunk lines were operating correctly, Wi-Fi clients successfully obtained DHCP addresses from their respective pools.

However, a routine post-deployment security scan revealed that clients on the Standard Wi-Fi VLAN (VLAN 20) could successfully ping the Admin Management interface on VLAN 10.

**Root Cause**

OPNsense routes traffic between internal interfaces unless firewall rules explicitly restrict that communication.

**Resolution**

A strict Block rule was placed at the top of the VLAN 20 interface ruleset to drop traffic destined for the Admin subnet.

Packet logging was enabled on the rule to maintain visibility for blue-team operations, ensuring that unauthorized lateral movement attempts generated telemetry in the system logs instead of passing silently.

![](https://github.com/k3rn3l-p4n1k-l4b/Enterprise-Threat-Detection-Lab/blob/main/screenshots/Active-Directory-Config-Changes/3OPNSENSEFirewallrules.png)

### Phase 3: The Host-Level Firewall Disconnect

_Why the Domain Controller Responded but Windows 11 Dropped Packets_

After establishing network perimeter blocks, testing shifted to the Attack Simulation VLAN.

During routing validation, a specific anomaly emerged:

- The Attack workstation could successfully ping the Windows Server Active Directory Domain Controller (DC).
- The Attack workstation could **not** ping individual Windows 11 workstations on the same enterprise subnet.

Because the Attack workstation had internet access and could successfully ping the Domain Controller, the intermediate Layer 2 switch tagging and Layer 3 OPNsense routing were functioning correctly. The issue appeared to be occurring at the host level.

**Diagnostic Explanation**

1. **Network Profile Mismatch:** The Domain Controller operated under the Domain network profile, while the Windows 11 workstations could be using a Public or Private profile with more restrictive inbound firewall rules.
2. **Built-in Scope Constraints:** The Windows 11 File and Printer Sharing (Echo Request - ICMPv4-In) firewall rule can be restricted to the Local Subnet. When a packet arrives from the Attack VLAN, its source falls outside that scope and the workstation drops the request.

---

## 3. Engineering Resolution: Scalable Active Directory GPO Implementation

To standardize network visibility across the lab without manually modifying every endpoint, a centralized Group Policy Object (GPO) was configured on the Domain Controller.

### Step 1: GPO Initialization

1. Opened Group Policy Management using `gpmc.msc`.
2. Created a new GPO named `Lab-Firewall-Allow-Ping`.
3. Linked it directly to the Organizational Unit (OU) containing the Windows 11 computer objects.

![](https://github.com/k3rn3l-p4n1k-l4b/Enterprise-Threat-Detection-Lab/blob/main/screenshots/Active-Directory-Config-Changes/4OUPolicyreationUnderWorkstationOUpng.png)

### Step 2: Policy Configuration

Navigated to:

```
Computer Configuration
└── Policies
    └── Windows Settings
        └── Security Settings
            └── Windows Defender Firewall with Advanced Security
```
### Step 3: Rule Provisioning

Created a predefined inbound rule targeting **File and Printer Sharing (Echo Request - ICMPv4-In)** and configured the action to **Allow the connection**.

![](https://github.com/k3rn3l-p4n1k-l4b/Enterprise-Threat-Detection-Lab/blob/main/screenshots/Active-Directory-Config-Changes/6IPV4ruleedit.png)

### Step 4: Scope Modification

Opened the rule's properties and navigated to the **Scope** tab.

Under Remote IP Address, changed the scope from `Local Subnet` to `Any IP Address`, or alternatively, the specific Attack VLAN subnet.

This allows ICMP echo requests from the designated remote subnet while making cross-subnet testing possible.

![](https://github.com/k3rn3l-p4n1k-l4b/Enterprise-Threat-Detection-Lab/blob/main/screenshots/Active-Directory-Config-Changes/7ruleactivatedconfirmation.png)
![](https://github.com/k3rn3l-p4n1k-l4b/Enterprise-Threat-Detection-Lab/blob/main/screenshots/Active-Directory-Config-Changes/7aipv4ruleedit.png)

### Step 5: Endpoint Enforcement

On the Windows 11 workstation, opened an administrative Command Prompt and executed a policy refresh:


```
gpupdate /force
```

![](https://github.com/k3rn3l-p4n1k-l4b/Enterprise-Threat-Detection-Lab/blob/main/screenshots/Active-Directory-Config-Changes/8gpupdateconfirmation.png)

---

## 4. Verification & Key Takeaways

Following GPO deployment, the Attack Simulation workstation successfully established ICMP echo replies with both the Domain Controller and the Windows 11 endpoints.

This troubleshooting exercise highlighted an important principle in enterprise security architecture:

> **Key Takeaway**
>
> Network-level accessibility through switches and routers is only half the battle. True connectivity requires aligning Layer 2/3 routing policies with host-level firewalls, network profiles, and centralized Group Policy management.

![](https://github.com/k3rn3l-p4n1k-l4b/Enterprise-Threat-Detection-Lab/blob/main/screenshots/Active-Directory-Config-Changes/9pingconfirmedworkingfromattack2.png)
