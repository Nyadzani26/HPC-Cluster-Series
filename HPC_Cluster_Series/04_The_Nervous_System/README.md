# Chapter 4: The Nervous System – Observability and Telemetry

## 4.1. From "Monitoring" to "Observability"

In traditional System Administration, we often spoke of "Monitoring." This typically meant binary checks: "Is the server on?" or "Is the website reachable?" This approach is reactive. It tells you *that* something is broken, but generally fails to explain *why*.

In High Performance Computing (HPC), we strive for **Observability**. Observability is a property of a system that allows you to deduce its internal state purely by analyzing its external outputs (telemetry). We do not just want to know if a Compute Node is "Up"; we want to know if it is suffering from thermal throttling, if its I/O buffers are saturated, or if it is experiencing micro-stalls due to network congestion.

To achieve this, we cannot rely on human operators refreshing terminal windows using `top` or `htop`. We must implement an automated telemetry pipeline that continuously aggregates thousands of data points per second from every corner of the cluster, storing them for historical analysis.

---

## 4.2. The Time-Series Database Paradigm

Standard relational databases (like SQL) are designed for transactional data — bank records, user inventories, or customer lists. They are optimized to update existing rows and guarantee consistency. This model is fundamentally ill-suited for telemetry data.

In telemetry, we are dealing with **Time-Series Data**:
1.  **Immutable:** Once a measurement is taken (e.g., "CPU Load at 12:00:01 was 90%"), it never changes. We never update old records.
2.  **High Volume:** A cluster might generate millions of metrics per minute.
3.  **Temporal Queries:** We almost always ask questions based on time (e.g., "What was the average load *between 10:00 and 11:00*?").

Therefore, we use a specialized **Time-Series Database (TSDB)**. For this project, we utilize **Prometheus**. Prometheus is optimized for write-heavy workloads and uses sophisticated compression algorithms (Delta-of-Delta encoding) to store massive amounts of timestamped data efficiently.

---

## 4.3. The Pull Model Architecture

There are two schools of thought on how to collect data from a fleet of servers:

### **The Legacy "Push" Model**
In older systems (like Nagios or Graphite), agents running on the servers would actively send ("push") data to the central server.
*   **The Flaw:** If you scale your cluster from 100 nodes to 10,000 nodes, suddenly 10,000 agents are flooding your central server simultaneously, often causing a Denial of Service (DoS) attack on your own monitoring infrastructure.

### **The Modern "Pull" Model (Prometheus)**
In this architecture, the agents (Exporters) are passive. They do not send data anywhere. They simply run a tiny web server that exposes their current metrics as text (e.g., on port 9100).
*   **The Advantage:** The central Prometheus server decides when to collect data. It iterates through its list of targets and "scrapes" (downloads) the data from them one by one. This prevents the central server from being overwhelmed and allows the monitoring system to scale linearly with the cluster size.

---

## 4.4. The Exporter Pattern

Prometheus itself does not know how to talk to a Linux Kernel, or a MySQL database, or a Slurm Scheduler. It only knows how to read text over HTTP. To bridge this gap, we use **Exporters**.

An Exporter is a small translation software.
1.  It queries the underlying system (e.g., reading `/proc/meminfo` from the Linux Kernel).
2.  It translates that raw system data into the standardized Prometheus exposition format (human-readable text).
3.  It serves that text on an HTTP port.

In this laboratory, we will deploy the **Node Exporter**. This is the industry-standard agent for Hardware Telemetry. It exposes thousands of low-level OS metrics, including CPU context switches, interrupt rates, disk I/O milliseconds, and network packet drops.

---

## 4.5. Why Docker?

In this tutorial, we deploy the entire monitoring stack using **Docker containers** rather than installing packages directly onto the host operating system. This is the approach used by the CHPC competition environment and reflects modern production practice. There are several key reasons:

*   **Isolation:** Each service (Prometheus, Grafana, Node Exporter) runs in its own container with its own dependencies. There is no risk of conflicting libraries or version clashes with other software on the host.
*   **Portability:** The entire stack is defined in a single `docker-compose.yml` file. You can tear it down and bring it back up on any machine running Docker with a single command.
*   **Reproducibility:** The exact same configuration runs identically on every node. There is no "it works on my machine" problem.
*   **Easy Updates:** Updating a service is as simple as pulling a new image version and restarting the container.

---

## 4.6. The Architecture of Our Stack

Before we begin, understand the layout of what we are building:

```
+--------------------------+        +---------------------------+        +---------------------------+
|       headnode           |        |       compute-01          |        |       compute-02          |
|  (10.100.0.10)           |        |  (10.100.0.11)            |        |  (10.100.0.12)            |
|                          |        |                           |        |                           |
|  [node-exporter :9100]   |        |  [node-exporter :9100]    |        |  [node-exporter :9100]    |
|  [prometheus    :9090]   |        |                           |        |                           |
|  [grafana       :3000]   |        |                           |        |                           |
+-----------+--------------+        +---------------------------+        +---------------------------+
            |                                    ^                                    ^
            |   scrapes metrics every 15 seconds |                                    |
            +------------------------------------+------------------------------------+
```

*   **Node Exporter** runs on **every node** (headnode, compute-01, compute-02). It reads the host system's metrics and serves them on port `9100`.
*   **Prometheus** runs **only on the headnode**. Every 15 seconds, it connects to each node's port `9100` and downloads ("scrapes") the current metrics.
*   **Grafana** runs **only on the headnode**. It queries Prometheus and displays beautiful, interactive dashboards on port `3000`.

All three services run with `network_mode: "host"`. This means they share the headnode's actual network interface rather than a private Docker virtual network. This is the most reliable approach for a cluster environment because it avoids any internal Docker DNS or routing issues when connecting to compute nodes on the cluster's private network.

---

# 4.7. Laboratory: Turning on the Lights

### **Prerequisites: Install Docker on ALL Nodes**

Docker must be installed on every node you want to monitor. In our cluster, that means **headnode**, **compute-01**, and **compute-02**.

Run the following commands on **each node** one by one. SSH into each node before running:

```bash
# Step 1: Install required dependencies for adding an external repository
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common

# Step 2: Add Docker's official GPG signing key so apt can verify downloaded packages
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Step 3: Add the official Docker repository to apt's source list
# lsb_release -cs returns your Ubuntu version codename (e.g., "jammy" for 22.04)
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list

# Step 4: Refresh the package list so apt sees the new Docker repository, then install
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Step 5: Start the Docker service and configure it to auto-start on boot
sudo systemctl start docker
sudo systemctl enable docker
```

**Verify Docker is working correctly** by running the test container:
```bash
sudo docker run hello-world
```
You should see a "Hello from Docker!" message. This confirms Docker can pull and run images successfully.

> **Note:** If you previously installed `prometheus-node-exporter` via `apt`, disable it on all nodes before continuing. The Docker container needs port `9100` to be free:
> ```bash
> sudo systemctl stop prometheus-node-exporter
> sudo systemctl disable prometheus-node-exporter
> ```

---

### **Step 1: Run Node Exporter on the Compute Nodes**

Node Exporter is the "sensor" — it reads every metric the Linux kernel exposes about the hardware and serves it as text over HTTP on port `9100`. We need it running on every compute node so that Prometheus (on the headnode) can collect their metrics.

Run this on **`compute-01`** and **`compute-02`** (SSH into each one):

```bash
sudo docker run -d \
  --name=node-exporter \
  --restart=always \
  --net="host" \
  --pid="host" \
  -v "/:/host:ro,rslave" \
  prom/node-exporter \
  --path.rootfs=/host
```

**Breaking down the flags:**
*   `--name=node-exporter` — Give the container a fixed name so we can refer to it later.
*   `--restart=always` — Docker will automatically restart this container if the node reboots.
*   `--net="host"` — Use the host's network namespace directly. This means the container listens on the actual node's IP address, not a Docker virtual IP. This is important so that Prometheus on the headnode can reach it.
*   `--pid="host"` — Give the container visibility into all processes running on the host. Without this, Node Exporter would only see processes inside the container, not the real system workloads.
*   `-v "/:/host:ro,rslave"` — Mount the host's entire root filesystem into the container at `/host` in read-only mode (`ro`). This is how Node Exporter reads the real `/proc`, `/sys`, and `/etc` from the host rather than the container's virtual view.
*   `--path.rootfs=/host` — Tell Node Exporter to look at `/host` as its root filesystem (matching the volume mount above).

**Verify Node Exporter is running on each compute node:**
```bash
sudo docker ps
curl localhost:9100/metrics | head -n 10
```
You should see lines like `node_cpu_seconds_total` and `node_memory_MemFree_bytes`. This is the raw telemetry data that Prometheus will collect.

---

### **Step 2: Set Up the Monitoring Stack on the Headnode**

All configuration for Prometheus and Grafana lives in `/opt/monitoring_stack` on the headnode. We will create three files:
1.  `docker-compose.yml` — defines all three containers and how they run
2.  `prometheus.yml` — tells Prometheus which nodes to scrape
3.  `prometheus-datasource.yaml` — tells Grafana where to find Prometheus automatically

**On `headnode`**, run:

```bash
# Create the directory that holds all our configuration files
sudo mkdir -p /opt/monitoring_stack
cd /opt/monitoring_stack
```

---

#### **File 1: `docker-compose.yml`**

This is the master configuration file. Docker Compose reads it and manages all three containers as a single coordinated stack.

```bash
sudo tee /opt/monitoring_stack/docker-compose.yml > /dev/null << 'EOF'
version: '3'
services:

  # ── Node Exporter ───────────────────────────────────────────────────────────
  # Runs on the headnode itself to collect headnode metrics.
  # Uses host networking and pid namespace so it can see the real hardware.
  node-exporter:
    image: prom/node-exporter
    container_name: node-exporter
    restart: always
    network_mode: "host"
    pid: "host"
    volumes:
      - "/:/host:ro,rslave"
    command:
      - '--path.rootfs=/host'

  # ── Prometheus ───────────────────────────────────────────────────────────────
  # The time-series database. Scrapes metrics from all node-exporters every 15s.
  # Uses host networking so it can reach compute nodes on the cluster network.
  prometheus:
    image: prom/prometheus
    container_name: prometheus
    restart: always
    network_mode: "host"
    volumes:
      - /opt/monitoring_stack/prometheus.yml:/etc/prometheus/prometheus.yml

  # ── Grafana ──────────────────────────────────────────────────────────────────
  # The visualization layer. Reads data from Prometheus and draws dashboards.
  # Uses host networking so it can reach Prometheus (also on host network).
  grafana:
    image: grafana/grafana
    container_name: grafana
    restart: always
    network_mode: "host"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin123
    volumes:
      - /opt/monitoring_stack/prometheus-datasource.yaml:/etc/grafana/provisioning/datasources/prometheus-datasource.yaml
EOF
```

> **Why `network_mode: "host"` for all services?**
> By default, Docker gives each container a private virtual IP on an internal bridge network. Containers on this bridge can talk to each other using their service names as hostnames (e.g., `http://prometheus:9090`). However, those internal names only work *within* the Docker bridge — they cannot reach external hosts like `compute-01` or `compute-02` on the cluster's private network. By setting `network_mode: "host"`, all containers share the headnode's actual network interface. This means Prometheus can directly dial `10.100.0.11:9100` to scrape `compute-01`, and Grafana can reach Prometheus at `http://127.0.0.1:9090` — no Docker DNS magic required.

---

#### **File 2: `prometheus.yml`**

This tells Prometheus what to scrape and where to find it. Replace the IP addresses with your actual node IPs.

```bash
sudo tee /opt/monitoring_stack/prometheus.yml > /dev/null << 'EOF'
global:
  # How often Prometheus collects (scrapes) metrics from each target
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node-exporter'
    static_configs:
      - targets:
        # Headnode: use 127.0.0.1 (localhost) because node-exporter is on the same host
        - '127.0.0.1:9100'
        # Compute nodes: use their actual private IP addresses
        - '10.100.0.11:9100'   # compute-01
        - '10.100.0.12:9100'   # compute-02
EOF
```

> **Why `127.0.0.1` for the headnode instead of its IP?**
> Since Prometheus is running in `network_mode: "host"` and Node Exporter is also running in `network_mode: "host"` on the same machine, they both share the headnode's localhost. Using `127.0.0.1:9100` is the most direct and reliable way to reach the local Node Exporter — it avoids any routing through the physical network interface.

> **Finding your node IPs:** Check `/etc/hosts` on your headnode with `cat /etc/hosts`. It will list the private IP addresses for all nodes in your cluster. Use those values in the targets list above.

---

#### **File 3: `prometheus-datasource.yaml`**

This file is automatically read by Grafana at startup and pre-configures the Prometheus data source. Without it, you would have to manually add the data source through the Grafana UI every time you rebuild the stack.

```bash
sudo tee /opt/monitoring_stack/prometheus-datasource.yaml > /dev/null << 'EOF'
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    # Since Grafana and Prometheus both use host networking,
    # Grafana reaches Prometheus at localhost on port 9090.
    url: http://127.0.0.1:9090
EOF
```

---

### **Step 3: Start the Stack**

Now bring the full monitoring stack up. Docker Compose will pull the images (first time only) and start all three containers:

```bash
cd /opt/monitoring_stack
sudo docker compose up -d
```

The `-d` flag runs the containers in the background ("detached" mode) so they keep running after you close your terminal.

**Verify all containers are running:**
```bash
sudo docker ps
```

You should see three containers: `prometheus`, `grafana`, and `node-exporter`. All should show status `Up`.

---

### **Step 4: Verify Prometheus Can Reach All Nodes**

Before opening Grafana, confirm that Prometheus has successfully scraped all three node exporters:

```bash
curl localhost:9090/api/v1/targets
```

Look for the `"health"` field in the JSON response. Each target should show `"health":"up"`:
- `127.0.0.1:9100` — headnode ✅
- `10.100.0.11:9100` — compute-01 ✅
- `10.100.0.12:9100` — compute-02 ✅

If a target shows `"health":"down"`, check:
1.  Is Node Exporter running on that node? (`sudo docker ps` on the compute node)
2.  Is port `9100` reachable from the headnode? (`curl 10.100.0.11:9100/metrics`)
3.  Is there a firewall blocking port `9100`? (`sudo ufw status`)

---

### **Step 5: Access Grafana and Import the Dashboard**

Grafana runs on port `3000`. Since your cluster is headless (no GUI), you will access it from your **local laptop** via an SSH tunnel.

**On your local laptop terminal**, open the tunnel:
```bash
ssh -L 3000:localhost:3000 ubuntu@<headnode-ip>
```

Now open your browser and go to: **`http://localhost:3000`**

**Login credentials:**
*   Username: `admin`
*   Password: `admin123`

**The Prometheus data source is already configured** (from the provisioning file we created). You can verify it by going to **Connections > Data Sources** — you should see Prometheus listed with a green checkmark after clicking **Save & Test**.

**Import the Node Exporter Full Dashboard:**
1.  In Grafana, click the **+** icon or go to **Dashboards > New > Import**.
2.  Enter dashboard ID: **`1860`** and click **Load**.
3.  Select **Prometheus** from the data source dropdown.
4.  Click **Import**.

You will now see a comprehensive dashboard showing CPU usage, memory, disk I/O, network traffic, and much more — for each node in your cluster. Use the **Instance** dropdown at the top of the dashboard to switch between `headnode`, `compute-01`, and `compute-02`.

---

### **Step 6: The Stress Test**

A monitoring system is only proven when there is something to observe. Let's generate a synthetic load on one of the compute nodes and watch it appear on the Grafana dashboard in real time.

**Install the stress tool on `compute-01`** (and `compute-02` if desired):
```bash
sudo apt install stress-ng -y
```

**Run a CPU stress job via Slurm:**
```bash
sbatch --wrap="stress-ng --cpu $(nproc) --timeout 60"
```

This command tells Slurm to run a job that saturates **all CPU cores** (`$(nproc)` expands to the number of cores) for exactly **60 seconds**.

**Observe in Grafana:**
Switch to the `compute-01` instance on the Node Exporter Full dashboard. Within 15 seconds (one scrape interval), you will see the CPU graph spike to 100%. After 60 seconds, it will drop back to idle. 

You have just observed a workload on your cluster in real time. This is exactly what competition judges and system administrators use to identify bottlenecks, rogue jobs, and failing hardware.

---

## 4.8. Useful Management Commands

Once the stack is running, these commands will be your everyday tools:

```bash
# Check the status of all containers
sudo docker ps

# View live logs from Prometheus (useful for debugging scrape errors)
sudo docker logs prometheus -f

# View live logs from Grafana
sudo docker logs grafana -f

# Restart a single service after editing its config
sudo docker compose restart prometheus

# Stop the entire monitoring stack (containers are removed but images are kept)
sudo docker compose down

# Start the stack back up
sudo docker compose up -d

# Check what Prometheus sees as its targets and their health
curl localhost:9090/api/v1/targets
```

---

**You have now successfully wired your cluster with a production-grade observability stack. The heartbeat of every node is visible in real time.**
