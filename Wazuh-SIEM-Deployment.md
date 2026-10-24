# Enterprise Threat Detection Lab: Wazuh SIEM Deployment

This project demonstrates the implementation of an enterprise-grade security architecture built inside a compact 10-inch home lab rack hypervisor cluster. The lab features a centralized **Wazuh SIEM/XDR platform** ingesting telemetry across segmented corporate network boundaries, covering a **Windows Server Active Directory Domain Controller**, a **Windows 11 Client Host**, and an **OPNsense Perimeter Firewall running Suricata NIDS**.

## Architectural Overview
* **Hypervisor Matrix:** Proxmox VE (Type-1 Bare-Metal Hypervisor)
* **Network Gateway & Firewall:** OPNsense 
* **Intrusion Detection System:** Suricata (Signature-based Network NIDS)
* **Identity & Access Management:** Windows Server Active Directory Domain Services (AD DS)
* **SIEM Platform Architecture:** Wazuh Central Management Server Virtual Appliance

---

## Engineering Pivot: Moving from Script Failures to Proxmox Disk Import
When initially deploying the Wazuh SIEM platform inside my Proxmox environment, I followed the standard documentation scripted installation layout on a fresh Ubuntu Server instance. However, the all-in-one script and the step by step install consistently failed and froze during the heavy compilation phase of the **Wazuh Dashboard** engine due to hypervisor package registry loops and automated rollback triggers.

Instead of fighting broken package states, I conducted independent research and transitioned to a cleaner, more robust infrastructure engineering method: downloading the official **Wazuh OVA virtual appliance**, manually provisioning an unmapped Proxmox virtual machine skeleton, and **directly importing the raw VMDK disk schema via the Proxmox CLI shell utility**. This completely bypassed the brittle installation scripts, providing an uncorrupted, fully optimized, enterprise-ready SIEM platform.

---

## 🛠️ Step-by-Step Implementation Guide

### Step 1: Stage the Appliance on the Proxmox Host
1. Use Promox GUI shell to access the root administrative shell.
2. Navigated to the native Proxmox virtual template image storage directory and made a Wazuh directory to keep files contained in one directory:
   ```bash
   cd /var/lib/vz/
   mkdir wazuh && cd wazuh
   ```
3.While in the Wazuh directory I downloaded the official OVA archive package directly from the Wazuh distribution infrastructure:
   ```bash
   wget https://packages.wazuh.com/4.x/vm/wazuh-4.14.9.ova
   ```
3. Extracted the compressed archive using `tar` to uncover the underlying virtual configuration schema and disk image:
   ```bash
   tar -xvf wazuh-v4.14.9.ova
   ```

![Proxmox host terminal displaying successful extraction of the raw virtual machine disk configuration schema](https://github.com/k3rn3l-p4n1k-l4b/Enterprise-Threat-Detection-Lab/blob/main/screenshots/1Wazuhfilesdownloadedandextracted.png)

_Figure 1.1: Proxmox host terminal displaying successful extraction of the raw virtual machine disk configuration schema._

---

### Step 2: Provision the Virtual Machine Skeleton
1. Logged into the Proxmox Web UI console dashboard.
2. Clicked **Create VM** and configured the custom bare hardware profile specs:
   * **Name:** `Wazuh-Lab`
   * **OS Tab:** Selected *Do not use any media* (bypassing ISO boot targets)
   * **System Tab:** Set Machine architecture type to `Q35`
   * **Disks Tab:** Wiped out and deleted the default hard drive disk entry completely
   * **CPU Tab:** Allocated `4 Cores` for high-throughput log evaluation rules
   * **Memory Tab:** Allocated `8192 MB` (8GB RAM) to satisfy database indexing overhead
   * **Network Tab:** Linked the network bridge interface to the internal lab VLAN leg managed by the OPNsense perimeter firewall and assigned it my VLAN tag for my A.D lab subnet. 
1. Noted the newly generated Proxmox hardware workspace tracker **VM ID: 101**.

![](https://github.com/k3rn3l-p4n1k-l4b/Enterprise-Threat-Detection-Lab/blob/main/screenshots/2VMCreation.png)
---

### Step 3: Import and Map the Virtual Disk via CLI
1. Returned to the Proxmox GUI terminal interface window and naivgated back to the Wazuh directory where files were unpacked.
2. Ran the `qm importdisk` utility to ingest the extracted virtual disks directly into Proxmox's local storage engine, mapping it to the empty VM container:
   ```bash
   qm importdisk 101 wazuh-4.14.9-disk-1.vmdk local-lvm --format raw
   ```

![](https://github.com/k3rn3l-p4n1k-l4b/Enterprise-Threat-Detection-Lab/blob/main/screenshots/3IngestingharddisktoVM.png)
![](https://github.com/k3rn3l-p4n1k-l4b/Enterprise-Threat-Detection-Lab/blob/main/screenshots/3aingestion%20complete.png)

_Figure 3.1: Hypervisor command logs completing the storage block parsing and successfully registering the imported disk structures._

---

### Step 4: Attach the Storage Volume and Configure Boot Order
1. Inside the Proxmox Web UI interface, navigated to the **Wazuh VM (101)** dashboard and opened the **Hardware** tab.
2. Selected the newly mapped data structure row labeled **Unused Disk 0**, clicked **Edit/Add**, and attached it as a system boot drive.
3. Switched to the **Options** tab and double-clicked **Boot Order**.
4. Disabled the standard optical/network boot interfaces, checked the newly attached virtual disk device row, and dragged it to the primary top-slot boot priority.

![](https://github.com/k3rn3l-p4n1k-l4b/Enterprise-Threat-Detection-Lab/blob/main/screenshots/4attachingharddisktoVM.png)
![](https://github.com/k3rn3l-p4n1k-l4b/Enterprise-Threat-Detection-Lab/blob/main/screenshots/5bootorder.png)

_Figure 4.1: Proxmox systems parameters page illustrating custom boot preference assignment targeting the imported array._

---

### Step 5: Initialize and Lock Down Infrastructure Network Settings
1. Powered on the virtual machine and opened the Proxmox **Console** panel.
2. Logged into the appliance command-line workspace using default credentials (`wazuh_user` / `wazuh`).
3. Conducted a system evaluation to verify the active network management subsystem. The forensic output confirmed that **`systemd-networkd`** is the active core engine, rendering legacy network configurations (`ifcfg-eth0`) obsolete:
   ```bash
   systemctl status systemd-networkd
   ```
4. To ensure networking persistence across compute reboots, purged the conflicting legacy configurations and automated cloud-init overlays that routinely force DHCP resets:
   ```bash
   sudo rm -f /etc/systemd/network/10-cloud-init-eth0.network
   sudo rm -f /etc/sysconfig/network-scripts/ifcfg-eth0
   ```
5. Provisioned a native, permanent configuration profile targeted directly at the active network manager:
   ```bash
   sudo nano /etc/systemd/network/10-eth0.network
   ```
6. Enforced an immutable manual interface schema by injecting the exact parameters matching the lab's OPNsense boundary configurations:
   ```text
   [Match]
   Name=eth0

   [Network]
   Address=xx.xx.xx.xx/xx
   Gateway=xx.xx.xx.xx
   DNS=xx.xx.xx.xx
   ```
   *Note: Press `Ctrl+O` then `Enter` to write the output, and `Ctrl+X` to exit the nano editor.*

7. Applied strict system permissions, bound the static service layer, cleared the dynamic caching layers, and initialized a hard reset to verify permanent retention across system reboots:
   ```bash
   sudo chmod 644 /etc/systemd/network/10-eth0.network
   sudo systemctl restart systemd-networkd
   sudo cloud-init clean
   sudo reboot
   ```
8. Post-reboot, executed network diagnostics via `ip a` to confirm the interface permanently retains the static pointer (`xx.xx.xx.xx./xx`) and routes successfully through the lab firewall leg.

![](https://github.com/k3rn3l-p4n1k-l4b/Enterprise-Threat-Detection-Lab/blob/main/screenshots/6loginwithdefaultcreds.png)
![](https://github.com/k3rn3l-p4n1k-l4b/Enterprise-Threat-Detection-Lab/blob/main/screenshots/7NetconfigScreenshot.png)

_Figure 5.1: System console output verifying that the systemd-networkd static network parameters successfully persist across standard OS reboots.

---
### Step 6: Verify SIEM Service Availability
1. Launched an enterprise web browser from the domain-joined internal testing host.
2. Navigated to the assigned network static interface pointer destination: `https://xx.xx.xx.xx`.
3. Validated authentication capabilities by logging into the central single-pane interface panel with default security manager parameters (`admin` / `admin`).

![](https://github.com/k3rn3l-p4n1k-l4b/Enterprise-Threat-Detection-Lab/blob/main/screenshots/8awazuhdashboard.png)
_Figure 6.1: Active, fully stable centralized Wazuh dashboard engine actively monitoring infrastructure boundaries in flash bang appearance setup._

---

## Phase 1.5: Critical Post-Deployment Security & Account Hardening

### ⚠️ Operational Security (OpSec) Note on the Master Admin Password
During the initial initialization of the Wazuh virtual appliance, the system auto-generates a complex, high-entropy master password for the root `admin` account. It is critical that this terminal output is **immediately screenshotted or documented in a secure password vault**. 

Because this account serves as the absolute "break-glass" master identity for the entire SIEM architecture, losing it requires a complete redeployment of the indexer cluster database.

### Best Practice: Creating a Personalized Operational Administrator Account
To follow enterprise hardening standards and ensure a forensic audit trail, the default master `admin` account should be reserved strictly for emergencies. For daily deployment tasks, network modifications, and agent onboarding, you should provision a personalized administrator account with a secure, memorable password. 

#### Execution Steps:
1. Log into the Wazuh Web UI using the master `admin` credentials.
2. Navigate to the top-left menu and click **OpenSearch Plugins** -> **Security** (or click the gear icon for **Dashboard Management** -> **Security** depending on your specific version layout).
3. Select **Internal Users** from the sidebar, then click **Create internal user**.
4. Configure your custom account parameters:
   * **Username:** Input your personal identifier (e.g., `ca7_admin`).
   * **Password / Repeat Password:** Input a secure, memorable passphrase that complies with enterprise complexity requirements.
5. Under the **Attributes / Roles** section, assign this new user the **`all_access`** and **`security_manager`** backend roles. This grants full administrative capabilities without relying on the shared master account.
6. Click **Submit** to finalize the account creation.
7. Log out of the master account, clear your browser session, and log back in using your new personal administrator account to verify proper permission mapping.

![](https://github.com/k3rn3l-p4n1k-l4b/Enterprise-Threat-Detection-Lab/blob/main/screenshots/9auserperms.png)
![](https://github.com/k3rn3l-p4n1k-l4b/Enterprise-Threat-Detection-Lab/blob/main/screenshots/9newusercreation.png)

_Figure 7.1: Wazuh internal user management console showing the successful provisioning of a secondary, personalized administrator profile with advanced role-based access control (RBAC) maps._

