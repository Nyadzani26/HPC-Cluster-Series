# Local HPC Cluster Setup Guide
**User:** Gift Nemakonde  
**Project:** AI-Based Fault Prediction for HPC Systems (Mini-Lab)  
**Host:** Windows 11 (8GB RAM)  
**Hypervisor:** VMware Workstation  

---

## Phase 1: Infrastructure Setup

### 1. VM Creation (Do this in VMware)
**Download:** [Ubuntu Server 22.04 LTS (or 24.04) ISO](https://ubuntu.com/download/server)  
*Note: During installation, choose "Minimize" or "Minimal" install to save disk/RAM.*

#### VM 1: `headnode`
- **RAM:** 2048 MB (2 GB)
- **Processors:** 2
- **Hard Disk:** 20 GB (Single file preferred)
- **Network Adapter:** NAT (Simplest for internet access + static IP)
- **Hostname:** `headnode`
- **Username:** `hpcuser` (Keep it same across all nodes!)

#### VM 2: `node01`
- **RAM:** 1024 MB (1 GB)
- **Processors:** 1
- **Hard Disk:** 15 GB
- **Network Adapter:** NAT
- **Hostname:** `node01`
- **Username:** `hpcuser`

#### VM 3: `node02`
- **RAM:** 1024 MB (1 GB)
- **Processors:** 1
- **Hard Disk:** 15 GB
- **Network Adapter:** NAT
- **Hostname:** `node02`
- **Username:** `hpcuser`

---

## Phase 2: OS Configuration (To be done after VMs boot)

### 2. Network Configuration (Netplan)
We need static IPs so Slurm nodes can always find each other.
*We will edit `/etc/netplan/00-installer-config.yaml` on each node.*

**Subnet Assumption:** VMware NAT is usually `192.168.x.x`.  
*Action:* Check your Windows IP (VMnet8 adapter) or run `ip a` inside the VM first to confirm the subnet range before assigning static IPs.

### 3. Hosts File
We must map hostnames to IPs on ALL nodes (Head, Node01, Node02).
Content for `/etc/hosts`:
```
127.0.0.1 localhost
192.168.10.10 headnode
192.168.10.11 node01
192.168.10.12 node02
```

---

## Phase 3: HPC Stack Installation (Upcoming)
1. Install Munge (Authentication)
2. Install Slurm Controller (Head) & Client (Nodes)
3. Configure NFS Share
4. Configure Slurm (`slurm.conf`)
