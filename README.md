# Enterprise-Threat-Detection-Lab
### Proxmox Active Directory & SIEM Monitoring Lab

This repository documents the setup, configuration, and architecture of my local Active Directory (AD) security lab hosted on Proxmox VE. The goal of this project is to build a realistic corporate environment to test enterprise administrative controls, practice threat hunting, and analyze security telemetry using an open-source SIEM. 

I am actively building on top of this environment to simulate real-world attacks and build detection rules. 

### Core Infrastructure

The entire laboratory runs on a dedicated Proxmox VE node isolated behind a custom virtual network (pfSense/pns_net). 

- **Domain Controller:** Windows Server 2022 (Hostname: DC01) 
    - Active Directory Domain Services (AD DS) 
    - DNS / DHCP roles configured 
- **Endpoints:** Windows 11 Enterprise (Hostname: WIN11-LAB01) 
    - Domain-joined to the testing environment 
    - Used for generating standard user activity and attack behaviors 
- **SIEM / Telemetry:** Wazuh Server (Ubuntu 24.04 LTS) 
    - Centralized log collection, file integrity monitoring, and alerting 

### Current State & Capabilities

- Active Directory forest up and running with custom Group Policy Objects (GPOs). 
- Core network isolation implemented to mimic an enterprise DMZ and internal corporate network. 
- Centralized logging pipeline configured to ship host-level telemetry to the SIEM. 

### Upcoming Roadmap

- [ ] Deploy an Ubuntu server running Apache/Nginx to serve as a target web server. 
- [ ] Implement Sysmon logging across all Windows nodes with a unified configuration file (e.g., SwiftOnSecurity). 
- [ ] Configure Windows Event Forwarding (WEF) to a centralized collector. 
- [ ] Run simulation tools (like Atomic Red Team) to validate alert pipelines. 

### How to Spin Up (Local Reference)

1. **Network:** Verify the Proxmox SDN or Linux Bridge settings match the isolated VLAN IDs. 
2. **Domain Controller:** Boot the DC, ensure the static IP allocation holds, and verify AD DS services are active. 
3. **Endpoints:** Boot client machines and confirm they can successfully resolve the DC domain name and authenticate. 
4. **Monitoring:** Access the SIEM dashboard via your management browser to view live infrastructure metrics.
