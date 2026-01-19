# Chapter 1: The Bedrock – Networking and Time Synchronization

## 1.1. The Role of Networking in Distributed Systems

In a Distributed System, the network is not merely a utility; it is the **system bus**. Just as a CPU accesses RAM via a motherboard bus, compute nodes access shared data and receive instructions via the network fabric.

### **The "Split-Brain" Problem**
In standalone computing, if a cable is unplugged, the internet stops. In High Performance Computing (HPC), if the internal network fails, the cluster can suffer from a condition known as "Split-Brain." This occurs when nodes lose contact with the controller and, assuming the controller is dead, may attempt to take unsafe actions or simply stall indefinitely. Therefore, reliability and predictability in the network layer are paramount.

### **Latency vs. Bandwidth**
*   **Bandwidth** is how *much* data you can send at once (e.g., streaming a 4K movie).
*   **Latency** is how *fast* a single message arrives (e.g., a ping in a video game).
HPC workloads are strictly **latency-sensitive**. When a simulation runs across 100 nodes, Node A often cannot proceed to Step 2 until it receives a result from Node B. If the network introduces a 10-millisecond delay, and this exchange happens 10,000 times a second, the supercomputer spends most of its time waiting, not computing.

### **Name Resolution Strategies**
Nodes must address each other by name (e.g., `compute-01`), not just by IP address. While the Domain Name System (DNS) works well for the internet, it introduces latency.
*   **Enterprise approach:** Centralized DNS servers.
*   **HPC approach:** Local lookup files (`/etc/hosts`). by distributing a static list of names to every node, we ensure that address resolution happens instantly (microseconds) without a network round-trip to a DNS server.

---

## 1.2. The Necessity of Time Synchronization

Time is one of the complex problems in computer science. In a single computer, the CPU clock is the absolute truth. In a distributed system, there is no absolute truth—only relative time.

### **The Problem of Clock Drift**
Every computer's hardware clock (Quartz crystal) vibrates at a slightly different frequency. Over the course of a day, two identical computers can drift apart by seconds.
*   **Impact on Authentication:** Security protocols (like Kerberos or Munge) issue tokens valid for a specific timeframe. If Node A thinks it is 12:00:00 and Node B thinks it is 12:05:00, a token valid for "12:00:00 +/- 1 minute" will be rejected by Node B as "expired."
*   **Impact on Logging:** If a calculation creates an error on Node A before crashing Node B, but Node B's clock is slow, the logs might imply the crash happened *before* the error, making debugging impossible.

### **NTP and Chrony**
To solve this, we use the Network Time Protocol (NTP). A master server connects to an atomic clock (Stratum 0), and downstream servers synchronize their clocks to it using sophisticated algorithms to account for network delay. In modern Linux systems, **Chrony** is the preferred implementation because it synchronizes the system clock faster and with greater accuracy than older NTP daemons.

---

## 1.3. Architecture of the Cluster Network

For a secure and stable cluster, we typically segregate traffic into two networks:
1.  **Public Network:** For administrative access (You SSH-ing into the Head Node).
2.  **Private Cluster Network:** An isolated network where nodes talk to each other. This prevents outside traffic from interfering with sensitive computations.

In our implementation, we will simulate this using a **Private Virtual LAN (NAT Subnet)**.

---

# 1.4. Laboratory: Configuring the Bedrock

In this lab, we will configure the foundational networking and time services on our Ubuntu nodes.

### **Prerequisites**
*   **Hypervisor:** VMware Workstation
*   **OS:** Ubuntu Server 24.04 LTS installed on two VMs.
*   **VM Names:** `headnode` and `compute-01`.

### **Step 1: Configuring Static IP Addresses (Netplan)**
We cannot rely on DHCP (Dynamic IP assignment) because server IPs must never change. We will use **Netplan**, the standard network configuration utility in Ubuntu.

**On ANY Node (Repeat for both):**
1.  Open the Netplan configuration file:
    ```bash
    sudo nano /etc/netplan/50-cloud-init.yaml
    ```
2.  Your interface name (e.g., `ens33`) may vary. Check it with `ip addr`.
3.  Configure the static IP:

**For `headnode`:**
```yaml
network:
    ethernets:
        ens33:
            dhcp4: false
            addresses:
            - 192.168.116.10/24
            routes:
            - to: default
              via: 192.168.116.2
            nameservers:
                addresses: [8.8.8.8, 8.8.4.4]
    version: 2
```

**For `compute-01`:**
Change the address to `192.168.116.11/24`, keep everything else the same.

4.  Apply the changes:
    ```bash
    sudo netplan apply
    ```

### **Step 2: Configuring Fast Name Resolution (Hosts File)**
We will hardcode the cluster's "phonebook" to ensure zero-latency lookups.

**On BOTH nodes:**
1.  Open the file:
    ```bash
    sudo nano /etc/hosts
    ```
2.  Add these lines at the bottom:
    ```
    192.168.116.10 headnode
    192.168.116.11 compute-01
    ```
3.  **Verification:** From `compute-01`, try to ping the headnode by name:
    ```bash
    ping -c 4 headnode
    ```

### **Step 3: Disabling the Firewall**
In a production environment, we would craft strict firewall rules. However, inside a secured, private cluster fabric, internal firewalls add latency and complexity. Since our cluster network is isolated, we disable the internal firewall to allow unrestricted communication between Slurm, NFS, and MPI services.

**On BOTH nodes:**
```bash
sudo ufw disable
```

### **Step 4: Synchronizing Time (Chrony)**
We will install Chrony to ensure our nodes are perfectly synced.

**On BOTH nodes:**
1.  Install Chrony:
    ```bash
    sudo apt update
    sudo apt install chrony -y
    ```
2.  Verify synchronization:
    ```bash
    chronyc tracking
    ```
    *Look for "System time" offset. It should be very small (e.g., < 0.001 seconds).*

---

### **Summary**
You have now laid the foundation. You have two servers with fixed identities (IPs), they know each other's names (Hosts), and they exist in the exact same moment in time (Chrony). The bedrock is solid.

**Next Chapter:** We will build the **Shared Identity and Storage** layer, making these two separate machines behave as one.
