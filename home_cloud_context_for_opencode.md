# Project Context: Hyper-Converged Home Cloud & Compute Cluster

## 1. Executive Summary

This document provides full architectural context for **OpenCode** (CLI AI Agent) to assist in developing, configuring, and automating a hyper-converged private cloud cluster.

The system repurposes standard hardware (a Dell laptop control plane and old Android phones flashed with native Linux edge nodes) into a scalable private cloud storage system (Nextcloud) and edge compute cluster (K3s).

## 2. Bare-Metal Target Architecture (Production)

### A. Control Plane (The Brain)

* **Hardware:** Dell Inspiron 14" (16GB RAM, 512GB SSD, built-in battery acting as UPS).
* **OS:** Ubuntu Server 24.04 LTS (Headless).
* **Role:** K3s Kubernetes Master Node, PostgreSQL database host, Redis cache host, Nextcloud server, **Private DHCP Server (`dnsmasq`)**, and Tier 1 (Hot) fast storage.

### B. Edge Nodes (The Workers & Storage Blocks)

* **Hardware:** Repurposed Android Phones (~8GB RAM, ARM CPUs, internal flash storage).
* **OS:** **Ubuntu Touch** (or PostmarketOS / Native Linux).
* **Role:** K3s Kubernetes Worker Nodes and Tier 2 (Cold/Bulk) external storage mounts.
* **Network Client:** Automatically requests an IP address via DHCP upon USB cable connection.
* **Safety:** Software charge limit (60%) or physical battery bypass to prevent thermal degradation/swelling during 24/7 operation.

### C. Physical & Network Backbone

* **Power & Data Hub:** Powered USB 3.0 Data Hub supplying wall power to phones while routing data to the Dell via USB.
* **Dual Network Topology:**
  * **Public Local Subnet (`192.168.1.X`):** Wi-Fi/Ethernet bridge connecting the Dell to the home router for client device access (phones, desktop browsers, smart TVs).
  * **Private Edge Subnet (`10.0.0.X`):** Air-gapped **RNDIS (Ethernet-over-USB)** connection.
    * **Dell Laptop Interface:** Static IP `10.0.0.1`.
    * **Edge Nodes Interface:** Dynamic IP assignment via Dell's internal DHCP server (range: `10.0.0.10` - `10.0.0.250`).
    * **Hostname Resolution:** Dynamic DNS / mDNS / local hosts auto-registration via DHCP lease hooks or `dnsmasq`.

## 3. Staging Environment Architecture (Oracle VirtualBox Lab)

Before deploying to bare-metal hardware, the entire stack is developed and validated inside **Oracle VirtualBox**. The phones are simulated as dedicated Virtual Machines running Linux to accurately mirror the DHCP network and SSHFS storage pooling.

### VM Cluster Topology

1. **`master-vm` (Simulating the Dell Laptop):**
   * **OS:** Ubuntu Server 24.04 LTS (Headless).
   * **Specs:** 4GB RAM, 2 vCPUs, 25GB Disk.
   * **Network Adapter 1:** Bridged Adapter (`192.168.1.X` - for access from host browser).
   * **Network Adapter 2:** Internal Network (`10.0.0.1` static - simulating the RNDIS USB master interface with `dnsmasq` DHCP service active).

2. **`edge-node-01-vm` (Simulating Phone #1 running Ubuntu Touch/Linux):**
   * **OS:** Ubuntu / Alpine Linux Server.
   * **Specs:** 1GB RAM, 1 vCPU, 10GB Disk.
   * **Network Adapter 1:** Internal Network (Set to DHCP - receives `10.0.0.X` IP from `master-vm`).
   * **Storage Export:** Runs OpenSSH server, exposing `/mnt/storage` for SSHFS mounting by `master-vm`.

3. **`edge-node-02-vm` (Simulating Phone #2 running Ubuntu Touch/Linux):**
   * **OS:** Ubuntu / Alpine Linux Server.
   * **Specs:** 1GB RAM, 1 vCPU, 10GB Disk.
   * **Network Adapter 1:** Internal Network (Set to DHCP - receives `10.0.0.X` IP from `master-vm`).
   * **Storage Export:** Exposes `/mnt/storage` over SSHFS.

## 4. Software Stack & Data Pipeline

```text
[ Client Browser / Mobile App ]
             │ (HTTP/HTTPS - 192.168.1.X)
             ▼
 ┌─────────────────────────────────────────────────────────────┐
 │ CONTROL PLANE (master-vm / Dell Laptop) - IP: 10.0.0.1      │
 │                                                             │
 │   ┌─────────────────┐       ┌───────────────────────────┐   │
 │   │  Nginx Proxy    │       │  DHCP Server (dnsmasq)    │   │
 │   └────────┬────────┘       │  (Manages 10.0.0.X Lease) │   │
 │            ▼                └─────────────┬─────────────┘   │
 │   ┌─────────────────┐                     │                 │
 │   │ Nextcloud App   │ ◄───► PostgreSQL & Redis              │
 │   └────────┬────────┘                                       │
 │            │                                                │
 │            ▼                                                │
 │   ┌─────────────────┐                                       │
 │   │ SSHFS Mount Point│                                       │
 │   └────────┬────────┘                                       │
 └────────────┼────────────────────────────────────────────────┘
              │ (RNDIS / SSHFS Tunnel - Dynamic DHCP Subnet)
              ├──────────────────────────┐
              ▼ (DHCP Lease)             ▼ (DHCP Lease)
 ┌──────────────────────────┐ ┌──────────────────────────┐
 │ EDGE NODE #1             │ │ EDGE NODE #2             │
 │ (Ubuntu Touch VM/Phone)  │ │ (Ubuntu Touch VM/Phone)  │
 │ IP: Dynamic (10.0.0.X)   │ │ IP: Dynamic (10.0.0.X)   │
 │ Storage: Tier 2 Flash    │ │ Storage: Tier 2 Flash    │
 │ Compute: K3s Worker      │ │ Compute: K3s Worker      │
 └──────────────────────────┘ └──────────────────────────┘
```

## 5. Directives for OpenCode CLI Agent

When assisting with this repository, **OpenCode** must follow these software engineering standards:

1. **Staging First:** All Docker Compose files, environment configurations, and shell automation scripts must work seamlessly inside the multi-VM VirtualBox test environment before bare-metal migration.

2. **DHCP & Dynamic Discovery:** Because edge nodes receive dynamic IPs from the server (`10.0.0.1`), automation scripts must discover active edge nodes using hostname resolution, ARP table scanning, or DHCP lease parsing (`/var/lib/misc/dnsmasq.leases`) rather than hardcoded IPs.

3. **Environment Separation:** Use `.env` files for all variable substitution (ports, database credentials, base subnet ranges, mount paths). Never hardcode secrets in compose files or scripts.

4. **Idempotent Automation:** Shell scripts (e.g., SSHFS mounting, automated Docker setup) must be idempotent. They should safely check current state (e.g., `if mountpoint -q ...`) before attempting modifications.

5. **Clean Abstractions:** Maintain separate directory structures:
   * `/docker` - Docker Compose configuration files and proxy rules.
   * `/scripts` - System provisioning, DHCP setup (`dnsmasq`), SSHFS auto-mounting, and network discovery scripts.
   * `/k3s` - Kubernetes manifests (reserved for production cluster migration).

6. **No GUI Assumptions:** All scripts and configurations must execute headlessly over SSH / CLI without requiring desktop environments.

## 6. Immediate Execution Tasks for OpenCode

Upon reading this context document, OpenCode is ready to assist with:

* **Task 1:** Generating `dnsmasq.conf` and setup script for the Dell server (`10.0.0.1`) to serve DHCP leases and handle local DNS for edge nodes.
* **Task 2:** Writing an automated node-discovery & mount script (`mount_nodes.sh`) that reads active DHCP leases and mounts edge storage via SSHFS dynamically.
* **Task 3:** Generating the primary production-grade `docker-compose.yml` for Nextcloud, PostgreSQL 16, and Redis 7.
* **Task 4:** Creating the `.env.example` file with configuration variables.
* **Task 5:** Providing Nginx reverse proxy configs for SSL and clean domain routing.