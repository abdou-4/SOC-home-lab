# 🛡️ SOC Engineering Lab – Phased Implementation

> A virtual Security Operations Center (SOC) lab that simulates a real‑world enterprise environment.  
> Built to practice network segmentation, threat detection, alert triage, and incident response using industry‑standard open‑source tools.  

---

## 📖 Table of Contents

- [Overview](#overview)
- [Phase 1 – Foundation (Completed)](#phase-1--foundation)
  - [Architecture & Network Segmentation](#architecture--network-segmentation)
  - [VM Components & Resource Constraints](#vm-components--resource-constraints)
  - [Network Zones & IP Assignment](#network-zones--ip-assignment)
  - [VirtualBox Network Configuration](#virtualbox-network-configuration)
  - [Initial Setup & Challenges](#initial-setup--challenges)
- [Phase 2 – Operations (Planned)](#phase-2--operations-planned)
- [How to Replicate](#how-to-replicate)
- [Lessons Learned](#lessons-learned)
- [Future Roadmap](#future-roadmap)
- [License & Acknowledgements](#license--acknowledgements)

---

## Overview

This project builds a complete SOC environment on a single physical host (16 GB RAM, 500 GB external HDD).  
The lab emulates a small enterprise with:

- **Firewall & network segmentation** (IPFire)  
- **Network Detection & Response (NDR)** (Security Onion)  
- **Endpoint Detection & Response (EDR)** (Wazuh)  
- **Centralized logging & alerting**  
- **A dedicated management workstation** (jumpbox)  
- **Simulated users, servers, and an attacker**

The lab is divided into two phases to ensure a solid foundation before adding detection engineering and automation.

---

## Phase 1 – Foundation

### Architecture & Network Segmentation

IPFire acts as the perimeter firewall, dividing the internal network into four zones:

| Zone        | Purpose                                      | Subnet          | IPFire Interface |
|-------------|----------------------------------------------|-----------------|------------------|
| **RED**     | WAN / untrusted (attacker)                   | 10.0.2.0/24     | RED              |
| **GREEN**   | Management zone (monitoring tools) | 192.168.30.0/24 | GREEN            |
| **BLUE**    | Server zone (victim servers)                 | 192.168.10.0/24 | BLUE             |
| **ORANGE**  | User zone (end‑user workstations)            | 192.168.20.0/24 | ORANGE           |

- **GREEN** has full outbound access (allows updates, management).  
- **BLUE** and **ORANGE** require explicit firewall rules to reach RED.  
- **RED** is isolated: IPFire blocks all inbound traffic from the attacker network.

### VM Components & Resource Constraints

All VMs run in VirtualBox on a host with **16 GB RAM** and **500 GB external HDD**.  
Because of limited resources, VMs are not always running simultaneously; a selective power‑on strategy is used.

| VM                       | Role                                      | vCPU | RAM (GB) | Storage (GB) | Zone      | IP Address   |
|--------------------------|-------------------------------------------|------|----------|--------------|-----------|--------------|
| **IPFire**               | Firewall / Router                         | 1    | 1        | 10           | –         | (DHCP on RED) |
| **Security Onion**       | NDR (network monitoring)                  | 4    | 8        | 200          | GREEN     | 192.168.30.2 |
| **Wazuh OVA**            | EDR / SIEM                                | 4    | 8        | 50           | GREEN     | 192.168.30.3 |
| **Ubuntu Desktop (GUI)** | Management workstation                    | 2    | 4        | 25           | GREEN     | 192.168.30.4 |
| **Windows Server 2019**  | Victim server (AD)                        | 2    | 2        | 40           | BLUE      | 192.168.10.2 |
| **Ubuntu Server**        | Victim Linux server                       | 1    | 2        | 25           | BLUE      | 192.168.10.3 |
| **Windows 10 LTSC**      | End‑user workstation                      | 2    | 2        | 40           | ORANGE    | 192.168.20.2 |
| **Parrot OS**            | Attacker (external)                       | 2    | 4        | 40           | RED       | (DHCP)       |

> **Note:** All internal VMs use static IPs; the management workstation acts as the only way to access monitoring dashboards (no direct internet access for servers), used RAM could be decrease only after installation 

### Network Zones & IP Assignment

| Zone    | Network Name (VirtualBox) | Subnet       | Gateway (IPFire) | VMs                                                                 |
|---------|---------------------------|--------------|------------------|---------------------------------------------------------------------|
| RED     | `nat-wan` (NAT Network)   | 10.0.2.0/24  | 10.0.2.1         | Parrot OS (attacker)                                                |
| BLUE    | `zone-server` (Internal)  | 192.168.10.0/24 | 192.168.10.1     | Windows Server, Ubuntu Server (victims)                             |
| ORANGE  | `zone-user` (Internal)    | 192.168.20.0/24 | 192.168.20.1     | Windows 10 (user)                                                   |
| GREEN   | `zone-monitoring` (Internal)| 192.168.30.0/24 | 192.168.30.1     | Security Onion (mgmt), Wazuh, Ubuntu Desktop           |

**Security Onion Monitoring Interface:**  
In addition to its management interface on GREEN, Security Onion has a **second adapter** attached to `zone-server` (BLUE) with **promiscuous mode** enabled. This allows it to capture all traffic on the server network without an IP address.

### VirtualBox Network Configuration

#### IPFire (4 adapters)

| Adapter | Attached To        | Network Name      | Purpose           | IPFire Zone |
|---------|--------------------|-------------------|-------------------|-------------|
| 1       | NAT Network        | `nat-wan`         | WAN link (attacker)| RED         |
| 2       | Internal Network   | `zone-server`     | Server zone       | BLUE        |
| 3       | Internal Network   | `zone-user`       | User zone         | ORANGE      |
| 4       | Internal Network   | `zone-monitoring` | Management zone   | GREEN       |

#### Security Onion (2 adapters)

| Adapter | Attached To        | Network Name      | IP Assignment       | Promiscuous Mode |
|---------|--------------------|-------------------|---------------------|------------------|
| 1       | Internal Network   | `zone-monitoring` | Static 192.168.30.2 | Disabled         |
| 2       | Internal Network   | `zone-server`     | None (monitor‑only) | **Allow All**    |

All other VMs have a single adapter attached to their respective internal network.

### Initial Setup & Challenges

During the build, several typical enterprise‑grade issues were encountered and resolved:

| Challenge                                         | Solution                                                                                     |
|---------------------------------------------------|----------------------------------------------------------------------------------------------|
| The external HDD change name depending on the I/O port & Virtualbox not lunching VMs due to KVM issue  | Wrote a bash file `/starter` that mount the HDD using the UUID and turn off intel KVM|
| Ubuntu netplan configuration lost after install in all VMs   | Manually wrote `/etc/netplan/50-cloud-init.yaml` with static IP, gateway, and nameservers.      |
| BLUE zones had no internet access          | Added all the blue MAC addresses in IPFire: **Access to Blue** |
| Wazuh manual installation repeatedly failed       | Switched to the official Wazuh OVA (pre‑configured).                                         |
| Parrot OS live environment hung on boot           | remove the ISO file from the VM storage.                                              |
| DNS resolution broken on Ubuntu                   | Set `/etc/resolv.conf` manually and disabled systemd‑resolved.                               |
| Systemd service timeouts                          | Increased `DefaultTimeoutStartSec=600` in `/etc/systemd/system.conf`.                        |

- [starter bash script](/starter)
- [netplan config file](/50-cloud-init.yaml)

---

## Phase 2 – Operations (Planned)

The foundation is now stable. The next phase will focus on **detection engineering, automation, and incident response**.

### Planned Activities

- **Complete agent deployment**  
  Ensure all endpoints have Wazuh agents installed and reporting.

- **Centralize all logs into Security Onion**  
  Configure Security Onion to ingest Wazuh alerts (via Elastic integration) and firewall logs from IPFire (syslog).

- **Detection engineering**  
  Write custom detection rules (Sigma / Elastic) mapped to MITRE ATT&CK, e.g.:
  - Privileged group changes
  - LSASS credential dumping
  - PowerShell obfuscation patterns

- **Normal traffic generation**  
  Use scripts or tools like LogGen to simulate benign user activity, reducing alert fatigue and creating realistic noise.

- **Adversary simulation**  
  Run Atomic Red Team or Caldera on victim machines to validate detection coverage.

- **SOAR automation**  
  Deploy **n8n** (or Shuffle) to create playbooks:
  - **Phishing email analysis** – automatically scan links/attachments with VirusTotal, URLScan, and a local AI model.
  - **Automated IP blocking** – when Suricata detects a known malicious C2 domain, n8n adds a firewall rule on IPFire.

- **Incident response documentation**  
  Simulate a ransomware attack, document the full investigation timeline, and produce a post‑mortem report.

- **Enhance dashboards**  
  Create Kibana dashboards showing:
  - Top triggered alerts
  - MITRE ATT&CK tactic breakdown
  - Geomap of incoming attacks (from firewall logs)

---

## How to Replicate

1. **Hardware requirements**  
   - Minimum: 16 GB RAM, 500 GB storage, CPU with virtualization support.  
   - All VMs can be run on a single host; use selective power‑on to stay within limits.

2. **Software**  
   - VirtualBox (or VMware)  
   - ISOs / OVAs: IPFire, Security Onion, Wazuh OVA, Windows Server/10 evaluation, Ubuntu Server/Desktop, Parrot OS.

3. **Network configuration**  
   - Create the three internal networks (`zone-server`, `zone-user`, `zone-monitoring`) and one NAT network (`nat-wan`) in VirtualBox.  
   - Assign static IPs as per the tables above.  
   - Configure IPFire zones and firewall rules accordingly.

4. **Installation steps**  
   - Follow the sequence: IPFire → Security Onion → Wazuh OVA → other VMs.  
   - Refer to the [Initial Setup & Challenges](#initial-setup--challenges) section for common pitfalls.

5. **Verification**  
   - Ensure all internal VMs can ping the gateway (IPFire) and, where permitted, the internet.  
   - Check that Security Onion’s monitoring interface captures packets (use `tcpdump` on the sensor).  
   - Confirm Wazuh agents are active on all endpoints.

---

## Lessons Learned

- **Resource planning is critical.** A single host with 16 GB RAM is sufficient, but not all VMs can run simultaneously. Plan ahead.  
- **Document network configuration early.** A clear zone and IP scheme avoids confusion later.  
- **Use pre‑built appliances when possible.** The Wazuh OVA saved hours of troubleshooting compared to manual installation.  
- **Promiscuous mode** is essential for monitoring traffic that does not flow directly to the sensor.  
- **Explicit firewall rules** for BLUE/ORANGE zones are often overlooked but mandatory for internet access.

---

## Future Roadmap

- Add a **second physical host** to increase capacity and simulate distributed infrastructure.  
- Integrate **MISP** for threat intelligence sharing.  
- Implement **Elasticsearch‑based anomaly detection** using machine learning (if resources permit).  
- Extend automation to include **Slack/Teams notifications** for critical alerts.

---

## License & Acknowledgements

This project is built using open‑source tools and is intended for educational purposes only.  
- **IPFire** – GNU GPL  
- **Security Onion** – Various open‑source licenses  
- **Wazuh** – GPLv2  
- **VirtualBox** – GPLv2  

Special thanks to the SOC community for inspiration and troubleshooting tips.

---

*Last updated: March 2026*  
*Phase 1 completed – Phase 2 in planning.*
