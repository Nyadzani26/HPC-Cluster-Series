# Chapter 1: The Bedrock – Networking, Firewalls, and Time Synchronization

---

## 1.1. The Role of Networking in Distributed Systems

In a Distributed System, the network is not merely a utility; it is the **system bus**. Just as a CPU accesses RAM via a motherboard bus, compute nodes access shared data and receive instructions via the **network fabric**.

---

### The "Split-Brain" Problem

In standalone computing, if a cable is unplugged, the internet stops. In High Performance Computing (HPC), if the internal network fails, the cluster can suffer from a condition known as **"Split-Brain."**

This occurs when nodes lose contact with the controller and, assuming the controller is dead, may attempt to take unsafe actions or simply stall indefinitely. Therefore, **reliability and predictability** in the network layer are paramount.

---

### Latency vs. Bandwidth

| Concept | Definition | Real-World Analogy |
|---|---|---|
| **Bandwidth** | How *much* data you can send at once | Streaming a 4K movie |
| **Latency** | How *fast* a single message arrives | A ping in a video game |

HPC workloads are strictly **latency-sensitive**. When a simulation runs across 100 nodes, Node A often cannot proceed to Step 2 until it receives a result from Node B. If the network introduces a 10-millisecond delay, and this exchange happens 10,000 times a second, the supercomputer spends most of its time **waiting**, not computing.

---

### Name Resolution Strategies

Nodes must address each other by name (e.g., `compute-01`), not just by IP address. While the Domain Name System (DNS) works well for the internet, it introduces **query latency**.

| Approach | Method | Characteristics |
|---|---|---|
| **Enterprise** | Centralized DNS servers | Flexible, scalable, slight network overhead |
| **HPC** | Local lookup files (`/etc/hosts`) | Instant (microsecond) resolution, no round-trip |

By distributing a static list of names to every node, we ensure that address resolution happens **instantly** (in microseconds) without a network round-trip to an external DNS server.

---

## 1.2. The Necessity of Time Synchronization

Time is one of the most complex problems in computer science. In a single computer, the CPU clock is the **absolute truth**. In a distributed system, there is no absolute truth — only **relative time**.

---

### The Problem of Clock Drift

Every computer's hardware clock (Quartz crystal) vibrates at a slightly different frequency. Over the course of a day, two identical computers can drift apart by **seconds**.

#### Impact on Authentication
Security protocols (like **Munge**) issue tokens valid for a strict timeframe.

> **Example:** If Node A thinks it is `12:00:00` and Node B thinks it is `12:05:00`, a token valid for `"12:00:00 +/- 1 minute"` will be **rejected by Node B as expired**.

#### Impact on Logging
If a calculation creates an error on Node A *before* crashing Node B, but Node B's clock is slow, the logs might imply the crash happened **before** the error — making debugging impossible.

---

### NTP and Chrony

To solve this, we use the **Network Time Protocol (NTP)**. A master server connects to an atomic clock (**Stratum 0**), and downstream servers synchronize their clocks to it using sophisticated algorithms to account for network delay.

In modern Linux systems, **Chrony** is the preferred implementation because it synchronizes the system clock faster and with greater accuracy than older NTP daemons.

```
[Atomic Clock — Stratum 0]
        │
[Public NTP Pool — Stratum 1/2]
        │
[headnode — NTP Server — Stratum 10]
        │
[compute-01 — NTP Client]
```

---

## 1.3. Architecture of the CHPC Cluster Network

To replicate the CHPC Student Cluster Competition architecture (Sebowa Cloud environment) inside **VMware Workstation**, we must move away from flat networks. We segregate traffic using a **dual-homed Head Node topology**:

```
  [ Internet / VMware NAT ]
              │
          ens33 (DHCP — assigned by VMware)
              ▼
       ┌──────────────┐
       │   headnode   │ ──► [ Stateful nftables Firewall ]
       └──────────────┘
              ▲
          ens37 (static — 10.100.0.10)
              │
  [ Isolated Host-Only Virtual Switch ]
              │
              ├──► compute-01 (10.100.0.11)
```

| Interface | Role |
|---|---|
| **ens33** | VMware-managed DHCP uplink — gives the headnode internet access |
| **ens37** | Static cluster fabric (`10.100.0.0/24`) — private communication between all nodes |

> **Key Design Principle:** Compute nodes sit **exclusively** on the cluster fabric (`ens37` side). They have no direct uplink and must route through the headnode via **Network Address Translation (NAT)** to reach the internet.

---

## 1.4. Laboratory: Configuring the Bedrock

---

### Prerequisites

| Requirement | Detail |
|---|---|
| **Hypervisor** | VMware Workstation |
| **OS** | Ubuntu Server 24.04 LTS (Minimal Install) on two VMs |
| **VM Hostnames** | `headnode` and `compute-01` |
| **headnode NICs** | Two network adapters: one NAT (Internet), one Host-Only (Cluster fabric) |
| **compute-01 NICs** | One network adapter: Host-Only only |

---

#### 🛠️ Configuring VMware Virtual Network Adapters
To configure the dual-homed network architecture, you must manually adjust the virtual hardware of your VMs:

1. **Power off or Suspend** both VMs in VMware Workstation.
2. **For `headnode` settings:**
   - Right-click the `headnode` VM and select **Settings** (or *Edit Virtual Machine Settings*).
   - In the hardware list, click the **Add...** button at the bottom.
   - Select **Network Adapter** from the wizard list and click **Finish**.
   - You will now see two network adapters:
     - **Network Adapter 1 (existing):** Keep configured as **NAT** (to provide public internet uplink).
     - **Network Adapter 2 (new):** Select it, and change its connection type on the right to **Host-only: A private network shared with the host** (for the isolated cluster fabric).
3. **For `compute-01` settings:**
   - Right-click the `compute-01` VM and select **Settings**.
   - Select its single **Network Adapter** and change its connection type to **Host-only** (so it exists solely on the private cluster switch).
4. Power both VMs back on.

---

### Step 1: Identifying Interface Names

Because VMware assigns random names to network cards, you must look up your specific device handles.

**On BOTH nodes, run:**

```bash
ip addr
```

Identify your interface names (e.g., `ens33`, `ens37`, `enp2s1`). Note them down — you will need them throughout this lab. Usually:
- `headnode`: `ens33` is NAT (external) and `ens37` is Host-only (internal).
- `compute-01`: `ens33` is Host-only (internal).

---

### Step 2: Configuring Fixed IP Identities (Netplan)

Server IP addresses must **never change**. We will hardcode static configurations using **Netplan**.

#### On `headnode`:

Open the Netplan configuration file:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Wipe the file and replace it with the following. `ens33` gets its address from VMware automatically; `ens37` is hardcoded as the cluster fabric interface:

```yaml
network:
    version: 2
    ethernets:
        ens33:
            dhcp4: true
        ens37:
            dhcp4: false
            addresses:
            - 10.100.0.10/24
```

#### On `compute-01`:

Open the Netplan configuration file:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

`compute-01` has only one NIC (`ens33`), connected to the Host-Only switch. Configure it with a static address and route all traffic through the headnode:

```yaml
network:
    version: 2
    ethernets:
        ens33:
            dhcp4: false
            addresses:
            - 10.100.0.11/24
            routes:
            - to: default
              via: 10.100.0.10
            nameservers:
                addresses: [8.8.8.8]
```

#### Apply Changes on Both Nodes:

```bash
sudo netplan apply
```

> **⚡ SSH ProxyJump Configuration (From your Windows Host):**
> Because `compute-01` resides on an isolated Host-Only network, your Windows PC cannot reach it directly. You must use the `headnode` as a jump host to reach it.
>
> Open **PowerShell** on your Windows host and run:
> ```powershell
> ssh -J ubuntu@<headnode_external_ip> ubuntu@10.100.0.11
> ```
> *(Replace `<headnode_external_ip>` with your actual `ens33` IP from the headnode, e.g. `192.168.116.130`).*
>
> Alternatively, for seamless access, open your Windows SSH configuration file (`notepad $HOME\.ssh\config`) and define:
> ```text
> Host headnode
>     HostName <headnode_external_ip>
>     User ubuntu
> 
> Host compute-01
>     HostName 10.100.0.11
>     User ubuntu
>     ProxyJump headnode
> ```
> Once saved, you can log directly into the compute node by running `ssh compute-01` from PowerShell!

---

### Step 3: Configuring the Head Node Gateway Router (NAT & IP Forwarding)

Because `compute-01` is on an isolated network, it currently **cannot access the internet** to install packages. We must turn the headnode into a **router**.

**On `headnode`:**

Enable IP forwarding in the Linux kernel:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Make this permanent across reboots:

```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
```

---

### Step 4: Deploying the Stateful nftables Firewall

Ubuntu uses `ufw` by default, but CHPC utilizes **nftables** for unified packet filtering and high-performance NAT handling. We will disable `ufw` and deploy an advanced nftables framework.

---

#### On BOTH Nodes — Disable default UFW:

```bash
sudo ufw disable
```

---

#### On `headnode` — Configure Gateway Firewall:

**Install the userspace utility:**

```bash
sudo apt update && sudo apt install nftables -y
```

**Flush any default rulesets:**

```bash
sudo nft flush ruleset
```

**Create the master table container:**

```bash
sudo nft add table inet hn_table
```

**Add the base infrastructure chains with default policies:**

```bash
sudo nft add chain inet hn_table hn_input '{ type filter hook input priority 0 ; policy accept ; }'
sudo nft add chain inet hn_table hn_forward '{ type filter hook forward priority 0 ; policy accept ; }'
sudo nft add chain inet hn_table hn_output '{ type filter hook output priority 0 ; policy accept ; }'
```

**Create auxiliary sub-chains to organize traffic:**

```bash
sudo nft add chain inet hn_table hn_tcp_chain
sudo nft add chain inet hn_table hn_udp_chain
```

**Implement stateful packet tracking rules:**

```bash
sudo nft add rule inet hn_table hn_input ct state related,established accept
sudo nft add rule inet hn_table hn_input ct state invalid drop
```

**Permit the internal loopback interface, ICMP (ping), and IGMP traffic:**

```bash
sudo nft add rule inet hn_table hn_input iif lo accept
sudo nft add rule inet hn_table hn_input meta l4proto icmp accept
sudo nft add rule inet hn_table hn_input ip protocol igmp accept
```

**Divert new TCP and UDP packets to their respective sorting chains:**

```bash
sudo nft add rule inet hn_table hn_input meta l4proto udp ct state new jump hn_udp_chain
sudo nft add rule inet hn_table hn_input 'meta l4proto tcp tcp flags & (fin|syn|rst|ack) == syn ct state new jump hn_tcp_chain'
```

**Authorize incoming SSH traffic to the headnode:**

```bash
sudo nft add rule inet hn_table hn_tcp_chain tcp dport 22 accept
```

**Reject all unhandled traffic matching other protocols:**

```bash
sudo nft add rule inet hn_table hn_input meta l4proto udp reject
sudo nft add rule inet hn_table hn_input meta l4proto tcp reject with tcp reset
sudo nft add rule inet hn_table hn_input counter reject with icmpx port-unreachable
```

**Configure IP Masquerading (NAT):**

Create the routing hook to rewrite outgoing packets from `compute-01` as they leave through `ens33` (the VMware DHCP uplink):

```bash
sudo nft add table inet my_nat
sudo nft add chain inet my_nat my_masquerade '{ type nat hook postrouting priority srcnat ; }'
sudo nft add rule inet my_nat my_masquerade oifname "ens33" masquerade
```

**Save your operational rulesets into the configurations directory:**

```bash
sudo mkdir -p /etc/nftables
sudo nft -s list ruleset | sudo tee /etc/nftables/hn.nft
```

**Set the input and forward baseline policies to `drop` inside the file for production security:**

```bash
sudo nano /etc/nftables/hn.nft
```

Modify the policy values on lines 4 and 5 from `accept` to `drop`:

```
type filter hook input priority 0; policy drop;
type filter hook forward priority 0; policy drop;
```

**Bind this ruleset to the system service configuration. Open `/etc/nftables.conf`:**

```bash
sudo nano /etc/nftables.conf
```

Ensure the contents match exactly:

```
flush ruleset
include "/etc/nftables/hn.nft"
```

**Restart and enable the persistence daemon:**

```bash
sudo systemctl stop nftables
sudo systemctl start nftables
sudo systemctl enable nftables
```

---

#### On `compute-01` — Verification:

Test your NAT routing layer. Your isolated compute node should now successfully resolve and ping external addresses through the headnode:

```bash
ping -c 4 8.8.8.8
```

---

### Step 5: Configuring Fast Name Resolution (Hosts File)

We hardcode the cluster's internal **phonebook** to eliminate name resolution delays.

**On BOTH nodes:**

Open the file:

```bash
sudo nano /etc/hosts
```

Append these exact records to the bottom:

```
10.100.0.10 headnode
10.100.0.11 compute-01
```

Test connectivity by hostname from `compute-01`:

```bash
ping -c 4 headnode
```

---

### Step 6: Synchronizing Time (Chrony & Firewall Integration)

We will configure the headnode as an **authoritative time server**, open the port in nftables, and bind `compute-01` to it.

---

#### On `headnode` — NTP Server Configuration:

**Install the time daemon:**

```bash
sudo apt install chrony -y
```

**Modify the configuration to allow the private fabric and force isolated accuracy:**

```bash
sudo nano /etc/chrony/chrony.conf
```

Add these lines at the **bottom** of the file:

```
allow 10.100.0.0/24
local stratum 10
```

**Restart the service:**

```bash
sudo systemctl restart chrony
```

**Open the Firewall Port:**

Open your master firewall ruleset:

```bash
sudo nano /etc/nftables/hn.nft
```

Locate your empty `hn_udp_chain` definition and add the UDP port 123 permission:

```
chain hn_udp_chain {
        udp dport 123 accept
}
```

**Reload the firewall configuration:**

```bash
sudo nft -f /etc/nftables/hn.nft
```

---

#### On `compute-01` — NTP Client Configuration:

**Install the time daemon:**

```bash
sudo apt install chrony -y
```

**Open the configuration file:**

```bash
sudo nano /etc/chrony/chrony.conf
```

Comment out (add a `#` in front of) all default `pool` or `server` lines.

Append your headnode as the absolute time authority using the `iburst` boot optimization:

```
server headnode iburst
```

**Restart the service:**

```bash
sudo systemctl restart chrony
```

---

### Step 7: Verifying the Time Sync Bedrock

**On `compute-01`**, force an immediate clock stepping adjustment and read your synchronization status:

```bash
sudo chronyc makestep
sudo chronyc sources
```

> **✅ Success Indicator:** The record for `headnode` or `10.100.0.10` must display the `^*` prefix symbol within a few seconds, proving active, accurate synchronization.

**On `headnode`**, verify that the compute node is recorded as a tracking customer:

```bash
sudo chronyc clients
```

---

## 1.5. Summary

You have successfully laid down a robust, production-grade bedrock. You have achieved:

| Achievement | Description |
|---|---|
| **Static Network Identities** | Both servers have fixed, predictable IP addresses |
| **Stateful Firewall** | nftables handles automatic packet processing with a drop-default policy |
| **NAT Routing** | Cluster isolation with internet access routed through the headnode |
| **Zero-Latency Name Lookups** | Local `/etc/hosts` phonebook eliminates DNS round-trips |
| **Sub-Millisecond Time Sync** | Chrony ensures all nodes share a single source of truth for time |

> **Next Chapter:** We will build the **Shared Reality** layer — configuring shared storage and a unified identity system, making these two separate machines behave as one cohesive cluster.
