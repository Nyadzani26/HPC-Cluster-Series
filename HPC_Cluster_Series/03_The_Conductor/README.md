# Chapter 3: The Conductor – Workload Management with Slurm

## 3.1. The Necessity of a Scheduler

In a standalone environment, if you want to run a program, you simply execute it. The Operating System kernel (Linux) schedules your process on the CPU immediately.

In a High Performance Computing (HPC) environment, where thousands of users compete for limited resources (Nodes, CPUs, RAM), this chaotic "free-for-all" approach would result in system collapse. Two users might try to use the same node simultaneously, causing both jobs to crash due to out-of-memory errors.

### **The Workload Manager**
To solve this, HPC systems use a **Workload Manager** (or Scheduler). This middleware sits between the user and the hardware.
1.  **Arbitration:** It decides who runs next based on priority policies (Fair Share).
2.  **Resource Allocation:** It grants exclusive access to specific hardware resources for a strictly defined period.
3.  **Launch & Monitor:** It propagates the user's executable to the allocated nodes, starts it, monitors its health, and cleans up afterwards.

---

## 3.2. Slurm Architecture

**Slurm (Simple Linux Utility for Resource Management)** is the dominant open-source scheduler in modern supercomputing. It follows a centralized server/client architecture:

### **1. The Controller Daemon (`slurmctld`)**
*   **Location:** Runs *only* on the Head Node.
*   **Role:** The "Brain." It maintains the state of the entire cluster (which nodes are valid, which are busy, which jobs are pending). All `sbatch` requests go here.

### **2. The Compute Daemon (`slurmd`)**
*   **Location:** Runs on *every* Compute Node.
*   **Role:** The "Hands." It waits for orders from the Controller. When ordered, it executes a job step, manages local process IDs, and streams stdout/stderr back to the user.
*   *Note: `slurmd` is designed to be lightweight and fault-tolerant. If the Controller crashes, the compute daemon keeps running the current job.*

### **3. The Database Daemon (`slurmdbd`)**
*   **Location:** Usually on the Head Node.
*   **Role:** The "Accountant." It records historical usage (accounting) to a database. *For our micro-cluster, we will disable accounting enforcement to simplify configuration.*

---

## 3.3. The Life of a Job

When a user submits a job, the following sequence occurs:

1.  **Submission (`sbatch`):** The user submits a script requesting "2 Nodes for 1 Hour."
2.  **Queuing:** `slurmctld` places the job in the Priority Queue.
3.  **Scheduling:** When resources become free, `slurmctld` selects the job.
4.  **Allocation:** `slurmctld` assigns specific nodes (e.g., `compute-01`, `compute-02`) to the job.
5.  **Step Launch:** `slurmctld` contacts `slurmd` on those nodes to launch the task.
6.  **Termination:** Once the time limit is reached or the script finishes, `slurmd` kills all processes, cleans the RAM, and reports "IDLE" back to the Controller.

---

# 3.4. Laboratory: Configuring the Conductor

In this lab, we will install Slurm and configure a basic "debug" partition.

### **Step 1: Installation**

**On BOTH Nodes:**
Install the Slurm packages (and Munge if not already active):
```bash
sudo apt update
sudo apt install slurm-wlm -y
```

### **Step 2: Configuration (`slurm.conf`)**

The configuration MUST be identical on all nodes. We will write it on the `headnode` and deploy it via NFS.

**On `headnode`:**
1.  Identify your hardware specs:
    ```bash
    slurmd -C
    ```
    *Copy the output (CPUs, RealMemory, Sockets, etc.). You need this for the config file.*

2.  Create the config file:
    ```bash
    sudo nano /etc/slurm/slurm.conf
    ```

3.  **The Manual Configuration:**
    Paste the following (adjust `RealMemory` to match your `slurmd -C` output, usually slightly less than VM total RAM):

    ```ini
    # --- IDENTITY ---
    ClusterName=mini-hpc
    SlurmctldHost=headnode
    
    # --- LOGGING & STATE ---
    SlurmUser=slurm
    SlurmdSpoolDir=/var/lib/slurm/slurmd
    StateSaveLocation=/var/lib/slurm/slurmctld
    SlurmctldLogFile=/var/log/slurm/slurmctld.log
    SlurmdLogFile=/var/log/slurm/slurmd.log
    
    # --- SCHEDULING ---
    SchedulerType=sched/backfill
    SelectType=select/cons_tres
    SelectTypeParameters=CR_Core
    
    # --- AUTHENTICATION ---
    # We rely on Munge
    AuthType=auth/munge
    CryptoType=crypto/munge
    
    # --- ACCOUNTING ---
    # Disable strict accounting for this lab
    AccountingStorageType=accounting_storage/none
    
    # --- NODES ---
    # Define our worker. Replace metrics with your 'slurmd -C' output
    NodeName=compute-01 CPUs=1 RealMemory=1900 State=UNKNOWN
    
    # --- PARTITIONS ---
    # Create a default queue called 'debug'
    PartitionName=debug Nodes=compute-01 Default=YES MaxTime=INFINITE State=UP
    ```

4.  **Distribute the Config:**
    Copy this file to the shared storage so the compute node gets it too:
    ```bash
    sudo cp /etc/slurm/slurm.conf /shared/
    ```

### **Step 3: Service Start-Up**

**On `headnode` (The Controller):**
```bash
sudo systemctl enable slurmctld
sudo systemctl start slurmctld
sudo systemctl status slurmctld
```
*It should be Active (Running).*

**On `compute-01` (The Worker):**
1.  Copy the config file to the local directory:
    ```bash
    sudo cp /shared/slurm.conf /etc/slurm/
    ```
2.  Start the daemon:
    ```bash
    sudo systemctl enable slurmd
    sudo systemctl start slurmd
    sudo systemctl status slurmd
    ```

### **Step 4: Verification**

Back on the **headnode**, check the status of your cluster:
```bash
sinfo
```
*   **Expected Output:**
    *   `PARTITION`: debug
    *   `AVAIL`: up
    *   `NODES`: 1
    *   `STATE`: idle
    *   `NODELIST`: compute-01

If state is `idle`, your cluster is ALIVE.

### **Step 5: Your First Job**

Create a simple test script called `test.sh`:
```bash
#!/bin/bash
#SBATCH --job-name=test
#SBATCH --output=result.txt
#SBATCH --nodes=1

echo "I am running on: $(hostname)"
date
```

**Submit it:**
```bash
sbatch test.sh
```

**Check the queue:**
```bash
squeue
```

**Check the result:**
```bash
cat result.txt
```
If the file says "I am running on: compute-01", you have successfully built a Supercomputer.
