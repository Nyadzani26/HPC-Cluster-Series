# Chapter 2: The Shared Reality – Identity and Storage

---

## 2.1. The Concept of "Single System Image" (SSI)

The ultimate goal of a cluster architect is to create an **illusion**. We want the user to believe they are logged into a single, massive computer, even though they are technically interacting with multiple distinct physical machines. This illusion is called a **Single System Image (SSI)**.

To achieve a true Single System Image, two fundamental operational truths must exist across all nodes in the fabric:

| Requirement | Description |
|---|---|
| **Identity Consistency** | "Who I am" on the Head Node must resolve to the exact same security profile on every Compute Node |
| **Data Consistency** | "My files" on the Head Node must be instantly visible and editable on every Compute Node at the exact same path |

---

## 2.2. Distributed Identity: UID Synchronization

In Linux, the operating system does not identify users by their text usernames (e.g., `ubuntu` or `gift`). It tracks ownership and permissions exclusively using a numerical index: the **User ID (UID)** and **Group ID (GID)**.

- When a file is created, its metadata is stamped with the current user's numerical ID (e.g., `Owner: 1000`).
- When a user or system service attempts to read that file, the Linux kernel cross-checks: *"Is your current executing UID 1000? If yes, grant access."*

---

### The HPC Challenge: Out-of-Sync Identifiers

If cluster nodes are configured manually and independently, user creation tools will simply increment the UID based on what is locally available. This quickly breaks the cluster.

**Consider this scenario:**

1. User account `alice` is created first on the `headnode`, receiving **UID 1000**.
2. On `compute-01`, a local administrator creates an account named `unwittinguser` first, which takes **UID 1000**.
3. The administrator then creates the `alice` account on `compute-01`. Because `1000` is taken, `alice` receives **UID 1001**.

---

### The Catastrophe

Because the filesystem is shared across the network:

1. Alice saves a compilation binary from the `headnode`. The file is written to disk stamped with `Owner: 1000`.
2. Alice launches a parallel processing job that executes on `compute-01`. The compute node runs her task under her local identity: **UID 1001**.
3. The compute node's OS checks the file, sees it belongs to **UID 1000** (which resolves locally to `unwittinguser`), and blocks her job with a fatal **Permission Denied** error.

---

### The Solution

In large-scale enterprise environments, supercomputers rely on centralized network directories (like **FreeIPA** or **LDAP**) to unify accounts. In our micro-cluster, we will use **Ansible automation** strictly as a lightweight identity manager to guarantee that names, UIDs, and GIDs match perfectly across all systems.

---

## 2.3. Distributed Authentication: MUNGE

How does the `headnode` trust that a control packet coming from `compute-01` is legitimate and hasn't been spoofed by an unauthorized device on the network?

Standard interactive SSH configurations work well for human users, but high-performance middleware services (like the **Slurm** scheduler) need to exchange validation signals thousands of times per second. They cannot pause processing loops to negotiate heavy RSA handshakes or ask for passwords.

**MUNGE (MUNGE Uid 'N' Gid Emporium)** is an authentication service designed specifically for high-throughput HPC environments:

```
[headnode]
  │  Generates munge.key
  │  Encodes payload → time-sensitive cryptographic credential
  ▼
[Network Fabric]
  │
  ▼
[compute-01]
  │  Holds identical munge.key
  │  Decodes credential → validates signature + timestamp
  ▼
[Trust Granted — Job Executes]
```

| Step | Action |
|---|---|
| **1** | Generate a single, highly secure Shared Secret Key (`munge.key`) on the control node |
| **2** | Copy this exact cryptographic key to every node allowed in the cluster |
| **3** | When Slurm transmits a request, the local MUNGE daemon encodes the payload inside a time-sensitive cryptographic credential wrapper signed by the key |
| **4** | The receiving node unwinds the wrapper. If the signature is validated and the internal timestamp aligns with the local clock, the cluster trusts the payload and executes the task instantly |

> **Note:** This is why Chrony time synchronization from Chapter 1 is not optional — MUNGE's timestamp validation will reject credentials if clocks drift too far apart.

---

## 2.4. Shared Storage: Network File System (NFS)

In an HPC environment, computational binaries are frequently executed across multiple systems simultaneously.

| Scenario | Reality |
|---|---|
| **Without Shared Storage** | A researcher manually `scp`s code files, inputs, and environment configurations to every compute node individually — every time a single line of source code changes |
| **With Shared Storage** | A single network-wide storage namespace is constructed. Scripts and runtimes in `/home` and `/shared` on the headnode are projected over the private network switch to all compute nodes |

---

### NFS Architecture

```
       ┌─────────────────────────────────┐
       │           headnode              │
       │  Physical Disk: /home /shared   │
       │  NFS Server — exports to subnet │
       └─────────────────────────────────┘
                        │
          [ Private Cluster Network ]
            10.100.0.0/24 fabric
                        │
       ┌────────────────┴───────────────┐
       │           compute-01           │
       │  /home  → mounted over network │
       │  /shared → mounted over network│
       │  Appears as local storage      │
       └────────────────────────────────┘
```

| Role | Node | Behaviour |
|---|---|---|
| **Server** | `headnode` | Manages physical drives and exports directories to the cluster subnet with explicit access controls |
| **Client** | `compute-01` | Mounts the remote network directory over its own local folder structure — all I/O is seamlessly marshalled across the network cable |

---

## 2.5. Laboratory: Unifying the Cluster Identity and Storage Layers

---

### Step 1: Manual MUNGE Security Configuration

We will manually install MUNGE on both nodes, generate our shared cluster token on the control plane, and distribute it securely.

---

#### On `headnode`:

**Install the core service packages:**

```bash
sudo apt update && sudo apt install munge libmunge-dev -y
```

**Generate the cluster secret key:**

```bash
sudo mungekey --create
```

**Tighten permissions** — MUNGE enforces strict security constraints and will completely refuse to launch if its token or configuration files are exposed to other system accounts:

```bash
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 400 /etc/munge/munge.key
```

**Enable and start the authentication daemon:**

```bash
sudo systemctl enable munge
sudo systemctl start munge
```

---

#### On `compute-01`:

**Install the matching client software:**

```bash
sudo apt update && sudo apt install munge libmunge-dev -y
```

**Stop the default daemon while configuring the network credentials:**

```bash
sudo systemctl stop munge
```

---

#### Deploying the Key — From `headnode`:

Because `/etc/munge` is a protected system directory owned by the root account, a raw `scp` command will fail due to write permissions on the destination. Instead, pipe the binary key file through SSH and use the `tee` utility to write it securely on the client node:

```bash
cat /etc/munge/munge.key | ssh -t ubuntu@compute-01 "sudo tee /etc/munge/munge.key > /dev/null"
```

---

#### Securing and Starting the Client Daemon — On `compute-01`:

**Lock down the newly arrived key file to prevent daemon initialization failure:**

```bash
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 400 /etc/munge/munge.key
```

**Start the authenticated processing loop:**

```bash
sudo systemctl enable munge
sudo systemctl start munge
```

---

#### Cryptographic Verification Test — From `headnode`:

Verify that your cluster keys match perfectly and that your Chrony time synchronization is working by encoding a live token on the control plane and decoding it instantly on the execution node:

```bash
munge -n | ssh compute-01 unmunge
```

> **✅ Success Output:** The command will output metadata showing `STATUS: Success (0)` along with matching encoding and decoding timestamps.

| Failure Message | Cause | Fix |
|---|---|---|
| `Invalid credential` | Key files do not match | Re-run the piped SSH key deployment step |
| `Expired credential` | Chrony clock sync has broken | Re-check your nftables UDP port 123 rules from Chapter 1 |

---

### Step 2: Manual NFS Shared Storage Deployment

We will configure the `headnode` to export vital storage locations and mount them permanently on the compute node.

---

#### On `headnode` — The Storage Server:

**Install the kernel space NFS engine:**

```bash
sudo apt install nfs-kernel-server -y
```

**Create an administrative cluster partition and set its user ownership:**

```bash
sudo mkdir -p /shared
sudo chown ubuntu:ubuntu /shared
```

**Define the export rule policies** to expose `/shared` and `/home` to your private network block:

```bash
sudo nano /etc/exports
```

Append these two exact lines to the bottom of the file:

```
/shared 10.100.0.0/24(rw,sync,no_subtree_check,no_root_squash)
/home   10.100.0.0/24(rw,sync,no_subtree_check,no_root_squash)
```

| Export Option | Meaning |
|---|---|
| `rw` | Grants read and write access to the volume over the network switch |
| `sync` | Forces the server to commit changes to physical storage before replying to client requests, preventing data corruption |
| `no_subtree_check` | Eliminates path validation overhead on sub-files, improving network I/O performance |
| `no_root_squash` | Permits administrative tasks initiated by root on a compute node to retain root privileges on the storage volume |

**Open TCP port 2049 inside the stateful firewall** to allow incoming mount requests:

```bash
sudo nano /etc/nftables/hn.nft
```

Locate your `hn_tcp_chain` and insert the NFS permission rule:

```
chain hn_tcp_chain {
        tcp dport 22 accept
        tcp dport 2049 accept
}
```

**Reload the firewall and update the active kernel storage tables:**

```bash
sudo nft -f /etc/nftables/hn.nft
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
```

---

#### On `compute-01` — The Storage Client:

**Install the common client utilities package:**

```bash
sudo apt install nfs-common -y
```

**Create the matching local mounting target folder:**

```bash
sudo mkdir -p /shared
sudo chown ubuntu:ubuntu /shared
```

> **⚠️ Mount Trajectory Correction:** You cannot safely mount a network storage pool over `/home` while your terminal session is actively sitting inside it. Change your directory to the system root before mounting:

```bash
cd /
sudo mount headnode:/shared /shared
sudo mount headnode:/home /home
```

**Refresh your shell context** to break out of the pre-mount directory memory cache and enter the live network directory space:

```bash
cd /home/ubuntu
```

**Run the storage validation utility** to check active mounts:

```bash
df -h -t nfs4
```

> **✅ Success Indicator:** Both `/home` and `/shared` paths show up active with matching storage metrics from the `headnode`.

---

#### Enforcing Permanent Mount Resilience:

To prevent the storage points from dropping during cluster reboots, add them to the system filesystem table on `compute-01`:

```bash
sudo nano /etc/fstab
```

Append these exact mounts at the very bottom of the file:

```
headnode:/home    /home    nfs    defaults,timeo=900,retrans=5,_netdev    0    0
headnode:/shared  /shared  nfs    defaults,timeo=900,retrans=5,_netdev    0    0
```

> **Key Option:** `_netdev` instructs the operating system to delay mounting until the cluster's network configuration is fully up — without this, the system may hang on boot trying to mount a network share before the interface is ready.

---

### Step 3: Exploiting NFS for Passwordless SSH Keys

Now that your home directory is projected across the entire cluster network, we can take advantage of a powerful HPC configuration technique. Setting up SSH keys inside your shared home folder **once** instantly distributes passwordless access across all compute nodes.

---

#### On `headnode`:

**Generate an optimized cryptographic keypair** using the ED25519 standard. Press Enter to accept all default paths and leave the passphrase empty:

```bash
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519
```

**Navigate into your secure SSH metadata folder:**

```bash
cd ~/.ssh
```

**Append your new public key to the list of authorized cluster access tokens:**

```bash
cat id_ed25519.pub >> authorized_keys
```

**Set tight local file permissions:**

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

---

#### On `compute-01` — Verification:

Because your home folder is shared, `compute-01` automatically sees the updated configuration. Test your passwordless access by initiating an SSH call from the compute node back to the headnode, or from the headnode directly into the compute node:

```bash
ssh compute-01
```

> **✅ Success Criteria:** The shell prompt drops you directly into the remote command line **without prompting for a password**. Type `exit` to return to your previous session.

---

### Step 4: Introducing Ansible Infrastructure Identity Automation

To ensure user accounts created for your team maintain perfect UID/GID synchronization, we will install **Ansible** on the control plane and use it exclusively for user lifecycle provisioning.

---

#### On `headnode` Only — Install Ansible:

Modern Ubuntu versions handle Python 3 natively. Install the automation platform along with the core environment interpreter mapping package:

```bash
sudo apt update && sudo apt install python3 python-is-python3 ansible -y
```

---

#### Build Your Cluster Control Inventory:

Open a new inventory mapping file in your home folder:

```bash
nano ~/inventory
```

Construct the inventory infrastructure records using INI syntax:

```ini
[head]
headnode

[compute]
compute-01
```

**Test your controller-to-client automated network connection** using the system ping module:

```bash
ansible -i ~/inventory compute -m ping
```

> **✅ Success Indicator:** The framework responds with a green `"ping": "pong"` confirmation message.

---

#### Automating User Management with an Ansible Playbook:

We will write a playbook to automate creating a new synchronized cluster user account and removing any temporary configurations.

**Create a structured playbooks folder and open a new task file:**

```bash
mkdir -p ~/playbooks
nano ~/playbooks/create_sudo_users.yml
```

Paste the following configuration script exactly. Note that we use the `sudo` group for administrative privileges to align with Ubuntu systems:

```yaml
---
- name: Cluster Infrastructure Identity Synchronization
  hosts: all
  become: true
  vars:
    add_sudo_user: gift
    del_user: unwittinguser

  tasks:
    - name: Ensure synchronized sudo user is present across all nodes
      ansible.builtin.user:
        name: "{{ add_sudo_user }}"
        state: present
        groups: "sudo"
        append: true
        shell: /bin/bash
        create_home: true

    - name: Purge unsynchronized temporary user profiles safely
      ansible.builtin.user:
        name: "{{ del_user }}"
        state: absent
        remove: true
```

**Execute your automated provisioning playbook across the cluster:**

```bash
ansible-playbook -i ~/inventory ~/playbooks/create_sudo_users.yml
```

---

#### Post-Execution Verification Checklist:

Run a quick identification lookup across both nodes to verify that your new team user was deployed with perfectly synchronized, identical identifiers:

```bash
# Check locally on headnode
id gift

# Check remotely on compute-01 via your passwordless SSH fabric
ssh compute-01 "id gift"
```

> **✅ Success Criteria:** Both commands report the **exact same** numerical user and group metrics (e.g., `uid=1001(gift) gid=1001(gift)`).

---

## 2.6. Summary

You have achieved a functioning **Single System Image (SSI)** framework. Your cluster nodes now:

| Achievement | Description |
|---|---|
| **MUNGE Authentication** | Nodes share cryptographic envelopes enabling instant, trusted inter-service communication |
| **NFS Shared Storage** | Unified network filesystem with permanent mounts surviving reboots via `/etc/fstab` |
| **Passwordless SSH** | Shared SSH keys in the NFS home directory enable frictionless node access |
| **Ansible Identity Sync** | Automated playbooks guarantee identical UID/GID profiles across every node |

> **Next Chapter:** We will deploy the **Conductor** — the Slurm Workload Manager — turning our unified cluster into a fully operational job scheduling and resource management system.
