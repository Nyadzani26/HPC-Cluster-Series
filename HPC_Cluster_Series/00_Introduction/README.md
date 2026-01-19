# Chapter 0: Introduction to High Performance Computing Architecture

## 1. Introduction

High Performance Computing (HPC) refers to the practice of aggregating computing power to solve complex problems that are too large or time-intensive for standard workstations. While traditional enterprise computing focuses on transactions (e.g., database queries, web serving), HPC focuses on **computational throughput**—the ability to perform trillions of floating-point operations per second (FLOPS) to simulate physical phenomena, train machine learning models, or process massive datasets.

This guide is designed for IT students to transition from single-server administration to distributed system architecture. We will move beyond the concept of a "computer" as a standalone entity and treat it as a component within a larger, unified computational resource.

---

## 2. Scaling Paradigms: Up vs. Out

To understand why we build clusters, we must first understand the limits of hardware.

### **Vertical Scaling (Scale-Up)**
Classically, if a computation was too slow, the solution was to buy a faster computer. This involves adding more CPUs, more RAM, or faster storage to a single server chassis.
*   **Limitation:** This approach hits a physical ceiling. There is a limit to how many transistors can fit on a chip (Moore's Law limits) and how much heat a single chassis can dissipate. Furthermore, the cost of specialized "super-servers" grows exponentially, not linearly.

### **Horizontal Scaling (Scale-Out)**
HPC utilizes horizontal scaling. Instead of using one massive processor, we connect hundreds or thousands of commodity processors using a high-speed network.
*   **The Beowulf Cluster:** This is the standard architecture for modern HPC. It consists of identical, commercial off-the-shelf (COTS) computers connected via a private local area network, running open-source software to coordinate tasks.
*   **The Challenge:** Software does not automatically run faster on a cluster. Applications must be rewritten to utilize **Distributed Memory** parallelism (typically using MPI - Message Passing Interface), explicitly passing data between nodes over the network.

---

## 3. The Anatomy of an HPC Cluster

A cluster is not just a bunch of computers wired together; it is a strictly tiered architecture with specific roles.

### **3.1. The Head Node (Control Plane)**
The Head Node (often called the Login Node or Controller) is the administrative gateway to the cluster.
*   **Function:** It hosts the critical middleware services: the job scheduler daemon, the cluster management tools, and the shared file system server.
*   **User Interaction:** Users SSH into this node to write scripts, compile code, and submit jobs. *Users never log into compute nodes directly.*
*   **Hardware Profile:** High reliability, moderate CPU, high I/O throughput (for serving files).

### **3.2. The Compute Nodes (Data Plane)**
These are the workhorses of the cluster.
*   **Function:** They execute the actual computational payloads. They sit idle until the scheduler assigns them a job. Once a job is assigned, they dedicate their resources entirely to that task.
*   **Hardware Profile:** High CPU core count, large memory capacity. Often stripped of unnecessary peripherals (video cards, audio) to conserve power for computation.

### **3.3. The Interconnect (Fabric)**
The network switch and cabling that connects the nodes.
*   **Function:** In standard IT, latency (delay) of a few milliseconds is acceptable. In HPC, latency is the enemy. If a simulation spans 100 nodes, they must synchronize their data thousands of times per second.
*   **Technology:** While we will use standard Ethernet (TCP/IP) for this learning lab, production clusters use specialized low-latency protocols like InfiniBand or Omni-Path.

### **3.4. Shared Storage (Single Name Space)**
For a cluster to function as a single system, a file created on the Head Node must immediately be visible to all Compute Nodes at the exact same path.
*   **Technology:** We will implement **NFS (Network File System)** to export the `/home` directory from the Head Node to the Compute Nodes. This ensures that when a user compiles a program `main.c` on the Head Node, the executable `a.out` is instantly available to the Compute Nodes for execution.

---

## 4. The Middleware Stack

A cluster requires a specific stack of software "middleware" to unify the hardware. In this project, we will configure the standard open-source HPC stack:

1.  **Time Synchronization (Chrony/NTP):**
    Distributed systems rely heavily on timestamps. If Node A thinks it is 12:00:00 and Node B thinks it is 12:00:05, authentication tokens will fail, and logs will be impossible to correlate. We require sub-millisecond precision.

2.  **Authentication (Munge):**
    In a cluster, thousands of messages fly between nodes ("Start job", "Job finished", "Status update"). Validating a password for every message is too slow. **Munge** is a specialized authentication service that allows nodes to verify each other's identity instantly using a shared cryptographic key.

3.  **Workload Manager (Slurm):**
    The **Simple Linux Utility for Resource Management (Slurm)** is the industry standard scheduler (used by ~60% of the TOP500 supercomputers).
    *   It manages the **Queue**: Users submit jobs, and Slurm orders them by priority.
    *   It manages **Resources**: It knows which nodes are busy and which are free.
    *   It launches **Tasks**: It executes the user's binary on the allocated nodes.

4.  **Observability (Prometheus & Grafana):**
    Because we cannot manually check the `htop` of 100 nodes, we implement a telemetry stack. Agents (Exporters) running on every node "scrape" metrics (CPU load, memory usage, job status) and send them to a central database for visualization.

---

## 5. Deployment Overview

In the modules that follow, we will build a fully functional **Micro-Cluster** that mimics a real production environment.

**The Specification:**
*   **Hypervisor:** VMware Workstation (representing our physical datacenter).
*   **OS:** Ubuntu Server 24.04 LTS (Minimal Install).
*   **Network:** A private, static IPv4 subnet (`192.168.116.x`).
    *   `192.168.116.10`: **head-node**
    *   `192.168.116.11`: **compute-01**

By the end of this course, you will not simply have "installed software"; you will have architected a distributed system capable of parallel processing, monitored by a professional telemetry stack.
