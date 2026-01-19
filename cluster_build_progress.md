# HPC Cluster Build Progress Tracker
**Project:** Mini-HPC Learning Cluster  
**Builder:** Gift Nemakonde  
**Started:** 2025-12-16  

---

## Infrastructure Setup Status

### Phase 0: VM Creation
- [ ] headnode VM created (2GB RAM, 2 vCPU)
- [ ] node01 VM created (1.5GB RAM, 1-2 vCPU)
- [ ] Ubuntu Server installed on both VMs
- [ ] Both VMs booted successfully

---

## Phase 1: Network Foundation
- [ ] 1. Static IP Configuration (Netplan)
- [ ] 2. Hosts File Configuration
- [ ] 3. Firewall Disabled (UFW)
- [ ] 4. Network Connectivity Verified

---

## Phase 2: System Foundation
- [ ] 5. User Account + UID Synchronization
- [ ] 6. Time Synchronization (Chrony)
- [ ] 7. Build Essentials Installed

---

## Phase 3: HPC Core Services
- [ ] 8. Munge Installation & Configuration
- [ ] 9. NFS Server (headnode) + Client (node01)
- [ ] 10. Slurm Installation & Configuration

---

## Phase 4: Parallel Computing
- [ ] 11. OpenMPI Installation
- [ ] 12. Test Job Execution
- [ ] 13. MPI "Hello World" Test

---

## Phase 5: Monitoring Stack (Later)
- [ ] Prometheus Installation
- [ ] Node Exporter on all nodes
- [ ] Grafana Installation
- [ ] Dashboard Configuration

---

## Notes & Issues
*This section will track any errors encountered and how we resolved them*

---

**Last Updated:** Waiting for VM installation to complete
