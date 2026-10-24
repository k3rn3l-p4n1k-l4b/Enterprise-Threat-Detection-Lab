### Step-by-Step Guide: Deploying Wazuh Agents in Proxmox AD Lab

This guide details how to install and register the Wazuh agent on both the Windows Server Domain Controller and the Windows 11 client machine. 

### Prerequisites

- A running Wazuh Server (Note down its static IP address, e.g., `10.0.0.50`). 
- Administrator access to both your Domain Controller and Windows 11 VM. 

---

### Part 1: Domain Controller Deployment (Windows Server)

### Step 1: Download the MSI Installer

1. Log into your Domain Controller via the Proxmox console or RDP. 
2. Open a browser and download the official Windows Wazuh Agent installer. 

![](https://github.com/k3rn3l-p4n1k-l4b/Enterprise-Threat-Detection-Lab/blob/main/screenshots/1downloadingtheagent.png) 

### Step 2: Install via PowerShell (Command Line Method)

Using PowerShell ensures that the agent points to your manager IP address immediately upon installation. 

1. Open **PowerShell as an Administrator**. 
2. Navigate to the directory you downloaded the MSI file to(Usually Downloads)
3. Run the following command (replace `YOUR_WAZUH_MANAGER_IP` with your actual server IP): 

```
msiexec.exe /i wazuh-agent-4.x.x.msi /q WAZUH_MANAGER="YOUR_WAZUH_MANAGER_IP" WAZUH_REGISTRATION_SERVER="YOUR_WAZUH_MANAGER_IP"
```


![](https://github.com/k3rn3l-p4n1k-l4b/Enterprise-Threat-Detection-Lab/blob/main/screenshots/3powershellinstallcommand.png) 

### Step 3: Verify and Start the Service

1. Type `services.msc` into the Windows Run dialog (`Win + R`) and press Enter. 
2. Scroll down to find the **Wazuh** service. 
3. Ensure the Status is **Running** and the Startup Type is set to **Automatic**. 

![](https://github.com/k3rn3l-p4n1k-l4b/Enterprise-Threat-Detection-Lab/blob/main/screenshots/4servicesmscverification.png) 

---

### Part 2: Windows 11 Lab Machine Deployment

### Step 1: Install Using the Dashboard Deployment Command
1. Open your management browser and log into the **Wazuh Web Dashboard**.
2. Navigate to **Wazuh Menu -> Agents -> Deploy new agent**.
3. Select **Windows** as the OS, enter your Wazuh Server IP address, and give the agent a name (e.g., `WIN11-LAB01`).
4. Copy the generated PowerShell command from the bottom of the page.
5. Open **PowerShell as an Administrator** on your Windows 11 machine, paste the command, and press Enter.
6. Verify the installation says it completed successfully, then run: `net start wazuh`.

![](https://github.com/k3rn3l-p4n1k-l4b/Enterprise-Threat-Detection-Lab/blob/main/screenshots/5webguiagentdeploy1.png)
![](https://github.com/k3rn3l-p4n1k-l4b/Enterprise-Threat-Detection-Lab/blob/main/screenshots/5awebguiagentdeploy.png)
![](https://github.com/k3rn3l-p4n1k-l4b/Enterprise-Threat-Detection-Lab/blob/main/screenshots/5bwebguiagentdeploy.png)

### Step 2: Permissions Bypass & Configuration Fix (If Agent Doesn't Appear)
If Windows UAC permissions locked the automated script out from writing your Server IP to the configuration file, perform the Desktop edit bypass:

1. Open File Explorer and navigate to `C:\Program Files (x86)\ossec-agent\`.
2. Right-click the **`ossec.conf`** file and select **Copy**.
3. Go to your **Desktop**, right-click an empty area, and select **Paste**.
4. Double-click the desktop copy of `ossec.conf` and open it with **Notepad**.
5. Locate the `<client>` block and manually type your Wazuh Server IP address into the `<address>` tags:
   ```xml
   <client>
     <server>
       <address>YOUR_WAZUH_SERVER_IP</address>
       <port>1514</port>
       <protocol>tcp</protocol>
     </server>
   </client>
   ```
6. Save the file and close Notepad.
7. Copy the modified `ossec.conf` from your Desktop and paste it back into `C:\Program Files (x86)\ossec-agent\`. 
8. Click **Continue** when prompted to grant administrator permissions, and choose **Replace the file in the destination**.
9. Open **PowerShell as an Administrator** and restart the service to apply changes:
   ```powershell
   Restart-Service -Name Wazuh
   ```

---
### Step 3: Allow Through Local Windows Firewall
If you have local firewall profiles enabled inside your lab, verify that outbound traffic on port `1514` (Agent communication) and port `1515` (Enrollment) is allowed. 

1. Open **Windows Defender Firewall with Advanced Security**. 
2. Confirm outbound rules do not block communication to your SIEM subnet. 

---

### Part 3: Verifying Agent Connectivity on the SIEM Dashboard

1. Open your management machine's web browser and navigate to your Wazuh Dashboard URL (`https://<WAZUH_SERVER_IP>`). 
2. Log in with your security credentials. 
3. Click on the **Wazuh Menu -> Agents**. 
4. Check the agent list to verify that both your Domain Controller and Windows 11 machine appear with an **Active** status indicator. 

![](https://github.com/k3rn3l-p4n1k-l4b/Enterprise-Threat-Detection-Lab/blob/main/screenshots/6bothmachinesinagentslist.png)
