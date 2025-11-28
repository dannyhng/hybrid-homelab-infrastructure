# Enterprise Hybrid Homelab Infrastructure

![Status](https://img.shields.io/badge/Status-Active-success)
![Infrastructure](https://img.shields.io/badge/Infrastructure-Proxmox%20VE%20%7C%20Bare%20Metal-orange)
![Network](https://img.shields.io/badge/Network-pfSense%20%7C%20Tailscale-blue)
![Automation](https://img.shields.io/badge/IAC-Ansible-red)
![Monitoring](https://img.shields.io/badge/Observability-Prometheus%20%7C%20Grafana-green)

## Project Overview

This repository documents the architecture and automation of a hybrid network environment. The design separates **Critical Infrastructure** (Virtual Routing/Storage) from **Compute Resources** (Bare Metal Docker Host) to ensure network stability during high-load operations.

The environment utilizes a **Zero-Trust (ZTNA)** approach for remote access, replacing legacy port-forwarding with SD-WAN overlays.

---

## Architecture Topology
```mermaid
graph TD
    %% --- External Network ---
    Internet((Internet)) -->|WAN IP| Modem[ISP Modem]
    
    %% --- Host A: Proxmox VE ---
    subgraph Host_A [Host A: Proxmox VE - Infrastructure]
        direction TB
        vmbr0[vmbr0 - WAN Bridge]
        vmbr1[vmbr1 - LAN Bridge]
        
        subgraph VMs [Critical Virtual Machines]
            pfSense[pfSense Firewall<br/>192.168.1.1]
            TrueNAS[TrueNAS Scale<br/>192.168.1.30]
            AdGuard[AdGuard DNS<br/>192.168.1.10]
            PBS[Proxmox Backup<br/>192.168.1.40]
        end
    end

    %% --- Host B: Bare Metal ---
    subgraph Host_B [Host B: Ubuntu Bare Metal - Compute]
        direction TB
        Docker[Docker Engine<br/>192.168.1.20]
        
        subgraph Containers [App Stack]
            Prom[Prometheus]
            Graf[Grafana]
            NPM[Nginx Proxy Mgr]
        end
    end

    %% --- Connections ---
    Modem --> vmbr0
    vmbr0 -->|Pass-through| pfSense
    pfSense -->|LAN 192.168.1.x| vmbr1
    
    vmbr1 -->|Virtual Switch| TrueNAS
    vmbr1 -->|Virtual Switch| AdGuard
    vmbr1 -->|Virtual Switch| PBS
    
    vmbr1 ==>|Physical Eth Cable| Docker
    Docker --> Prom
    Docker --> Graf
    Docker --> NPM
    
    %% --- Styling ---
    classDef physical fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    classDef virtual fill:#fff9c4,stroke:#fbc02d,stroke-width:2px;
    class Host_A,Host_B physical;
    class pfSense,TrueNAS,AdGuard,PBS,Docker virtual;

```
# System Architecture

### 1. Infrastructure Layer (Hybrid)

The lab utilizes a split-resource model to prevent compute exhaustion from impacting network routing.
* **Host A (Dell OptiPlex 7070):** Runs **Proxmox VE**. Hosts critical "always-on" services (Router, DNS, NAS).

* **Host B (Physical PC):** Runs **Ubuntu Server 24.04**. Dedicated bare-metal host for Docker containers.


### 2. Networking & Security

* **Edge Firewall:** pfSense acting as the primary gateway (`192.168.1.1`), managing VLANs and strictly separating lab traffic from the ISP network.

* **Remote Access:** Tailscale installed directly on pfSense as a Subnet Router. This allows secure access to internal IPs (`192.168.1.x`) without exposing ports to the WAN.

* **DNS:** AdGuard Home handles local DNS resolution (`app.homelab.local`) and network-wide content filtering.

### 3. Application & Proxy
* **Reverse Proxy**: Nginx Proxy Manager (NPM) handles SSL termination and routes traffic based on subdomains.

* **Proxmox Proxy:** Configured with Websocket support to allow secure access to the Proxmox console via standard HTTPS (443)

### 4. Observability Stack
Monitoring is deployed via Docker Compose on the bare-metal host.

  * **Node Exporter:** Configured with `--net=host` to scrape raw hardware metrics from the Ubuntu kernel.

  * **Prometheus:** Time-series database scraping targets at 15s intervals.

  * **Grafana:** Visualizes metrics using standard Linux Host dashboards (ID 1860).

### 5. Storage & Backups
* **TrueNAS Scale:** Virtualized with physical disk passthrough (via serial number mapping).

* **ZFS Mirror:** Two-disk mirror (RAID-1) configuration for data redundancy.

* **Backup:** Proxmox Backup Server (PBS) performs incremental nightly snapshots of all critical VMs.

### 6. Automation
Infrastructure management is handled via **Ansible**.
* **Inventory:** Defines connection methods for both physical (SSH) and virtual (API/Root) hosts.
* **Playbooks:**
    * `site.yml`: Routine system maintenance and package updates.
    * `deploy_monitoring.yml`: Automates the deployment of the Docker observability stack.
  
---
## How to Run
### 1. Prerequisite Setup
Ensure your control node (laptop/desktop) has **Ansible** and **Git** installed.
```bash
# Update and install Ansible (Ubuntu/Debian)
sudo apt update && sudo apt install -y ansible git
```

### 2. Clone the Repository
```bash
git clone [https://github.com/dannyhng/hybrid-homelab-infrastructure.git](https://github.com/dannyhng/hybrid-homelab-infrastructure.git)
cd hybrid-homelab-infrastructure
```

### 3. Run Automation (Ansible)
Edit `ansible/inventory/inventory.ini` to match your IP addresses, then run the maintenance playbook:
```bash
ansible-playbook -i ansible/inventory/inventory.ini ansible/playbooks/site.yml -K
```

### 4. Deploy Containers
Deploy the monitoring stack:
```bash
cd docker/monitoring
docker-compose up -d
```

