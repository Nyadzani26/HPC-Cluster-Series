# Chapter 2: The Shared Reality – Identity and Storage

## 2.1. The Concept of "Single System Image" (SSI)

The ultimate goal of a cluster architect is to create an illusion. We want the user to believe they are logged into a single, massive computer, even though they are technically interacting with hundreds of distinct machines. This illusion is called a **Single System Image (SSI)**.

To achieve SSI, two fundamental truths must exist across all nodes:
1.  **Identity Consistency:** "Who I am" on the Head Node must be exactly "Who I am" on every Compute Node.
2.  **Data Consistency:** "My files" on the Head Node must be exactly "My files" on every Compute Node.

---

## 2.2. Distributed Identity: UID Synchronization

In Linux, the operating system does not know you as "John" or "Sarah." It knows you as a number: the **User ID (UID)**.
*   When you create a file, the file is stamped with `Owner: 1000`.
*   When you try to read a file, the OS checks: "Is your current UID 1000? If yes, access granted."

### **The HPC Challenge**
If you create a user `researcher` on the Head Node, Linux might assign them `UID 1001`.
If you create a user `researcher` on Compute Node 1, Linux might assign them `UID 1002`.

**The Catastrophe:**
1.  Researcher (UID 1001) creates a file on shared storage. The file says `Owner: 1001`.
2.  Researcher logs into Compute Node 1 (UID 1002).
3.  Researcher tries to read their own file.
4.  Compute OS says: "This file is owned by 1001. You are 1002. Access Denied."

**The Solution:**
In large clusters, we use centralized identity servers (LDAP/FreeIPA) to force synchronization. In our micro-cluster, we rely on the default `ubuntu` user (always UID 1000) or check manually to ensure UIDs match perfectly across all nodes.

---

## 2.3. Distributed Authentication: Munge

How does the Head Node trust that a message coming from `compute-01` is legitimate? It could be a hacker on the network spoofing the IP address.

Standard SSH keys work for users, but system services (like Slurm) need to talk to each other thousands of times per second. They cannot stop to type passwords or verify bulky RSA handshakes.

**MUNGE (MUNGE Uid 'N' Gid Emporium)**
Munge is an authentication service specifically designed for HPC.
1.  We generate a **Shared Secret Key** (`/etc/munge/munge.key`).
2.  We copy this key physically to every node in the cluster.
3.  When Slurm sends a message ("Run this job at UID 1000"), Munge wraps that message in a specialized cryptographic envelope signed with the key.
4.  The receiver unwraps it. If the signature matches, it accepts the message instantly and trusts the UID inside it implicitly.

---

## 2.4. Shared Storage: Network File System (NFS)

In a distributed computation, a program is often launched on 50 nodes simultaneously.
*   **Without Shared Storage:** You would have to `scp` (copy) your program to all 50 nodes manually every time you changed a line of code.
*   **With Shared Storage:** You put your code in `/home`. That directory is physically on the Head Node, but "mounted" over the network to all Compute Nodes.

**NFS Architecture:**
1.  **Server (`headnode`):** "Exports" a directory (e.g., `/home` or `/shared`) to the network.
2.  **Client (`compute-01`):** "Mounts" that remote directory. To the OS, it looks like a local hard drive, but all reads/writes travel over the network cable.

---

# 2.5. Laboratory: Unifying the Cluster

In this lab, we will implement Munge and NFS to bind our two nodes into a single coherent system.

### **Step 1: Identity Check**
We need to verify that our user (`ubuntu`) has the same UID on both nodes.

**On BOTH nodes:**
```bash
id ubuntu
```
*Result: Ensure `uid=1000(ubuntu)` matches on both.*

### **Step 2: Configuring Munge (The Secret Handshake)**

**On `headnode` (Generate the Key):**
1.  Install Munge:
    ```bash
    sudo apt install munge libmunge-dev -y
    ```
2.  Create a key (if one wasn't auto-generated):
    ```bash
    sudo mungekey --create
    ```
3.  Set permissions (Munge is paranoid about security; it will refuse to start if permissions are wrong):
    ```bash
    sudo chown munge:munge /etc/munge/munge.key
    sudo chmod 400 /etc/munge/munge.key
    ```
4.  Start the service:
    ```bash
    sudo systemctl enable munge
    sudo systemctl start munge
    ```

**On `compute-01` (Receive the Key):**
1.  Install Munge:
    ```bash
    sudo apt install munge libmunge-dev -y
    ```
2.  **Transfer the Key:** *Securely copy the key file from `headnode` to `compute-01` at `/etc/munge/munge.key`.* (You can use `scp` or copy-paste the base64 content if SSH isn't set up).
3.  Set identical permissions and start the service.

**Test Authentication:**
From `headnode`, ask `compute-01` to verify a credential:
```bash
munge -n | ssh compute-01 unmunge
```
*Success Output: `STATUS: Success (0)`*

### **Step 3: Configuring NFS (Shared Storage)**

**On `headnode` (The Server):**
1.  Install the server:
    ```bash
    sudo apt install nfs-kernel-server -y
    ```
2.  Create a shared directory:
    ```bash
    sudo mkdir -p /shared
    sudo chown ubuntu:ubuntu /shared
    ```
3.  Configure the export file:
    ```bash
    sudo nano /etc/exports
    ```
    Add this line:
    ```
    /shared 192.168.116.0/24(rw,sync,no_subtree_check,no_root_squash)
    /home   192.168.116.0/24(rw,sync,no_subtree_check,no_root_squash)
    ```
    *(Note: We share `/home` so users have the same environment everywhere)*.
4.  Apply exports:
    ```bash
    sudo exportfs -a
    sudo systemctl restart nfs-kernel-server
    ```

**On `compute-01` (The Client):**
1.  Install the client:
    ```bash
    sudo apt install nfs-common -y
    ```
2.  Mount the directories:
    ```bash
    sudo mount headnode:/shared /shared
    sudo mount headnode:/home /home
    ```
3.  **Make it Permanent:** Add to `/etc/fstab` so it mounts on boot.

### **Step 4: The "Touch" Test**
To verify SSI is working:
1.  On `headnode`: `touch /shared/cluster_test.txt`
2.  On `compute-01`: `ls -l /shared/cluster_test.txt`
3.  If you see the file immediately, congratulations: You have broken the barrier between machines.
