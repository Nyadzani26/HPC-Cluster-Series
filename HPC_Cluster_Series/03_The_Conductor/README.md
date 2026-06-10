# Chapter 3: The Conductor – Workload Management with Slurm

---

## 3.1. The Necessity of a Scheduler

In a standalone environment, running a program is simple: you execute it, and the operating system kernel (Linux) schedules your process on the CPU immediately.

In a High Performance Computing (HPC) environment, where thousands of users compete for limited resources (nodes, CPUs, RAM), a chaotic "free-for-all" approach would quickly lead to system collapse. Two users might run memory-intensive calculations on the same node simultaneously, causing both jobs to crash due to out-of-memory (OOM) errors.

---

### The Workload Manager

To solve resource contention, HPC systems employ a **Workload Manager** (commonly referred to as a scheduler). This middleware sits between the users and the hardware fabric:

```
[ Users (sbatch / srun) ]
           │
           ▼
┌───────────────────────────────────────┐
│     Slurm Workload Manager (Arbitration)│
└───────────────────────────────────────┘
           │
   ┌───────┼───────┐
   ▼       ▼       ▼
[Node 1] [Node 2] [Node 3] (Exclusive Hardware Allocation)
```

| Operational Pillar | Role |
|---|---|
| **Arbitration** | Decides which job executes next based on queue priority, fair-share algorithms, and resource availability |
| **Resource Allocation** | Grants exclusive access to a specific subset of hardware resources (nodes, cores, memory) for a strict time limit |
| **Launch & Monitor** | Propagates the user's executable to the allocated nodes, manages execution loops, streams output, and performs post-job cleanup |

---

## 3.2. Slurm Architecture

**Slurm (Simple Linux Utility for Resource Management)** is the dominant open-source scheduler in modern supercomputing. It utilizes a centralized controller/client architecture:

```
       ┌───────────────────────────────┐
       │           headnode            │
       │   [ slurmctld ] (Controller)  │
       └──────────────┬────────────────┘
                      │
        [ Private Cluster Network ]
         Ports 6817/6818 TCP traffic
                      │
       ┌──────────────┴────────────────┐
       │          compute-01           │
       │    [ slurmd ] (Compute)       │
       └───────────────────────────────┘
```

| Daemon | Location | Operational Profile |
|---|---|---|
| **`slurmctld`** (Controller) | Runs on **headnode** | The **"Brain."** Tracks node state, monitors the priority queues, and assigns tasks. All job submissions are managed by this daemon. |
| **`slurmd`** (Compute) | Runs on **compute-01** | The **"Hands."** Listens for instruction payloads from `slurmctld`, forks local processes, manages memory control groups, and returns log outputs. |
| **`slurmdbd`** (Database) | Runs on **headnode** | The **"Accountant."** Archives job histories, billing metrics, and user association details into a database. *(Disabled in our micro-cluster for simplicity)*. |

---

## 3.3. The Life of a Job

The scheduling loop follows a structured lifecycle to ensure predictable resource isolation:

```
[ sbatch script.sh ] ──► [ Queue (slurmctld) ] ──► [ Allocation Analysis ]
                                                            │
  ┌─────────────────────────────────────────────────────────┘
  ▼
[ Step Launch (slurmd) ] ──► [ Execution ] ──► [ Cleanup & Node Release ]
```

| Phase | Description |
|---|---|
| **1. Submission** | A user submits a batch script requesting specific parameters (e.g., `#SBATCH --nodes=1 --time=00:30:00`). |
| **2. Queuing** | The Controller places the job into the queue, sorting it by partition rules and priority. |
| **3. Allocation** | Once the requested resources become idle, the Controller claims them exclusively for this job. |
| **4. Step Launch** | The Controller signals the execution daemon (`slurmd`) on the target node to run the script. |
| **5. Cleanup** | When the job finishes or hits its time limit, `slurmd` terminates orphan processes, purges temp space, and alerts the Controller that the node is `IDLE` and ready for reuse. |

---

## 3.4. Laboratory: Configuring the Conductor

---

### Step 1: Install Slurm Packages

**On BOTH Nodes:**

Install the standard Slurm Workload Manager packages:

```bash
sudo apt update && sudo apt install slurm-wlm -y
```

---

### Step 2: System Specs Discovery

Slurm configuration must reflect the exact hardware profiles of the compute nodes. Discrepancies between your configuration file and physical hardware will cause compute daemons to fail on startup.

**On `compute-01`:**

Query the local hardware configuration exactly as Slurm sees it:

```bash
slurmd -C
```

> **📝 Note Down the Output:** The command will print a line containing `CPUs=... Board=... Sockets=... CoresPerSocket=... ThreadsPerCore=... RealMemory=...`. You will use these exact parameters in the configuration file.

---

### Step 3: Create the Slurm Configuration

The configuration file `/etc/slurm/slurm.conf` **must be identical across all nodes**. We will author it on the `headnode` and share it across the network.

**On `headnode`:**

Open a new configuration file:

```bash
sudo nano /etc/slurm/slurm.conf
```

Paste the following blueprint. Modify the `NodeName` properties (CPUs, RealMemory) at the bottom to match your exact output from Step 2:

```ini
# --- IDENTITY ---
ClusterName=mini-hpc
SlurmctldHost=headnode

# --- CONTROL DAEMONS & PORTS ---
SlurmctldPort=6817
SlurmdPort=6818
SlurmUser=slurm
StateSaveLocation=/var/lib/slurm/slurmctld
SlurmdSpoolDir=/var/lib/slurm/slurmd

# --- LOGGING ---
SlurmctldLogFile=/var/log/slurm/slurmctld.log
SlurmdLogFile=/var/log/slurm/slurmd.log

# --- SCHEDULING ---
SchedulerType=sched/backfill
SelectType=select/cons_tres
SelectTypeParameters=CR_Core

# --- AUTHENTICATION ---
AuthType=auth/munge
CryptoType=crypto/munge

# --- ACCOUNTING ---
AccountingStorageType=accounting_storage/none

# --- NODES ---
# Replace CPUs & RealMemory metrics with your 'slurmd -C' output
NodeName=compute-01 NodeAddr=10.100.0.11 CPUs=1 RealMemory=1900 State=UNKNOWN

# --- PARTITIONS ---
PartitionName=debug Nodes=compute-01 Default=YES MaxTime=INFINITE State=UP
```

---

### Step 4: Authorizing Slurm in the Gateway Firewall

For `compute-01` to communicate with the controller, we must open TCP ports `6817` and `6818` inside the firewall.

**On `headnode`:**

Open your master firewall ruleset:

```bash
sudo nano /etc/nftables/hn.nft
```

Append the ports to the `hn_tcp_chain` rules:

```
chain hn_tcp_chain {
        tcp dport 22 accept
        tcp dport 2049 accept
        tcp dport 6817-6818 accept
}
```

Reload the ruleset to apply changes:

```bash
sudo nft -f /etc/nftables/hn.nft
```

---

### Step 5: Distribute and Launch Services

**On `headnode` (The Controller):**

Ensure the state directory exists and is owned by Slurm:

```bash
sudo mkdir -p /var/lib/slurm/slurmctld /var/log/slurm
sudo chown -R slurm:slurm /var/lib/slurm/ /var/log/slurm/
```

Enable and start the Controller service:

```bash
sudo systemctl enable slurmctld
sudo systemctl start slurmctld
```

Expose the configuration file to the shared NFS mount:

```bash
sudo cp /etc/slurm/slurm.conf /shared/
```

---

**On `compute-01` (The Worker):**

Copy the synchronized config file from the shared directory:

```bash
sudo cp /shared/slurm.conf /etc/slurm/
```

Ensure the state directory exists and permissions match:

```bash
sudo mkdir -p /var/lib/slurm/slurmd /var/log/slurm
sudo chown -R slurm:slurm /var/lib/slurm/ /var/log/slurm/
```

Enable and start the Compute service:

```bash
sudo systemctl enable slurmd
sudo systemctl start slurmd
```

---

### Step 6: Cluster Verification

**On `headnode`:**

Query the cluster state to confirm `compute-01` has registered:

```bash
sinfo
```

> **✅ Success Indicator:** The output should show the `debug` partition as `up` with `compute-01` listed in the `idle` state.

```
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
debug*       up   infinite      1   idle compute-01
```

#### Troubleshooting Node States

If your node state is listed as `down` or `drain`, check the logs (`/var/log/slurm/slurmctld.log`). You can manually force the node back into service by running:

```bash
sudo scontrol update nodename=compute-01 state=resume
```

---

### Step 7: Executing Your First Batch Job

Now we will test resource orchestration by submitting a script that executes asynchronously on the compute node.

**On `headnode`:**

Create a test batch script named `test.sh` inside your home folder:

```bash
nano ~/test.sh
```

Add the following execution script:

```bash
#!/bin/bash
#SBATCH --job-name=cluster_test
#SBATCH --output=result.txt
#SBATCH --nodes=1
#SBATCH --partition=debug

echo "Task is executing on host: $(hostname)"
echo "Current execution directory: $(pwd)"
echo "Execution timestamp: $(date)"
```

Submit the script to the scheduler queue:

```bash
sbatch ~/test.sh
```

Monitor the execution queue:

```bash
squeue
```

Once the queue clears, inspect the output file written to the shared filesystem:

```bash
cat ~/result.txt
```

> **✅ Success Criteria:** The output file `result.txt` should print:
> - Host: `compute-01` (proving the scheduler marshalled the job to the compute node).
> - Path: `/home/ubuntu` (proving execution occurred on the shared storage layer).

---

## 3.5. Summary

You have successfully configured the **Conductor** of your cluster. Your nodes are no longer independent systems acting in isolation; they are fully coordinated under a central orchestrator.

| Achievement | Impact |
|---|---|
| **Central Scheduler Daemon** | `slurmctld` listens to job allocations on the control plane |
| **Active Execution Daemon** | `slurmd` actively executes workloads on `compute-01` |
| **Secure Networking** | nftables firewall policies permit scheduler traffic on ports `6817/6818` |
| **Successful Orchestration** | Asynchronous job execution verified via `sbatch` |

> **Next Chapter:** We will install the **Nervous System** — compiling Message Passing Interface (MPI) libraries to allow parallel computations to scale across all cores and machines simultaneously.
