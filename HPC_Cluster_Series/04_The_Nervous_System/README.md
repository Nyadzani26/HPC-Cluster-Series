# Chapter 4: The Nervous System – Observability and Telemetry

## 4.1. From "Monitoring" to "Observability"

In traditional System Administration, we often spoke of "Monitoring." This typically meant binary checks: "Is the server on?" or "Is the website reachable?" This approach is reactive. It tells you *that* something is broken, but generally fails to explain *why*.

In High Performance Computing (HPC), we strive for **Observability**. Observability is a property of a system that allows you to deduce its internal state purely by analyzing its external outputs (telemetry). We do not just want to know if a Compute Node is "Up"; we want to know if it is suffering from thermal throttling, if its I/O buffers are saturated, or if it is experiencing micro-stalls due to network congestion.

To achieve this, we cannot rely on human operators refreshing terminal windows using `top` or `htop`. We must implement an automated telemetry pipeline that continuously aggregates thousands of data points per second from every corner of the cluster, storing them for historical analysis.

---

## 4.2. The Time-Series Database Paradigm

Standard relational databases (like SQL) are designed for transactional data—bank records, user inventories, or customer lists. They are optimized to update existing rows and guarantee consistency. This model is fundamentally ill-suited for telemetry data.

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

# 4.5. Laboratory: Turning on the Lights

In this lab, we will deploy the full Observability Stack: **Node Exporter** (the sensor), **Prometheus** (the brain), and **Grafana** (the face).

### **Step 1: Installing the Sensor (Node Exporter)**

We must install this on **EVERY** node in the cluster, because we want to monitor every machine.

**On `headnode` AND `compute-01`:**
```bash
sudo apt update
sudo apt install prometheus-node-exporter -y
```

**Verification:**
The exporter should immediately start a web server on port 9100. Text it using `curl`:
```bash
curl localhost:9100/metrics | head -n 10
```
*You should see lines of text beginning with `node_cpu_seconds` or `node_vmstat`. This is the raw data.*

---

### **Step 2: Installing the Brain (Prometheus)**

We install this **ONLY** on the `headnode`.

1.  **Install:**
    ```bash
    sudo apt install prometheus -y
    ```
2.  **Configure:**
    We need to tell Prometheus where our nodes are.
    ```bash
    sudo nano /etc/prometheus/prometheus.yml
    ```
    Scroll to the `scrape_configs` section and add your nodes:
    ```yaml
      - job_name: 'node_exporter'
        scrape_interval: 15s
        static_configs:
          - targets: ['headnode:9100', 'compute-01:9100']
    ```
3.  **Restart:**
    ```bash
    sudo systemctl restart prometheus
    ```

**Verification:**
Open your web browser on your Windows machine and navigate to: `http://192.168.116.10:9090/targets`. You should see both endpoints listed with a status of **UP**.

---

### **Step 3: Installing the Face (Grafana)**

Prometheus collects the data, but it is terrible at displaying it. Grafana is a visualization engine that queries Prometheus and draws beautiful, interactive dashboards.

**On `headnode`:**
1.  **Install Dependencies:**
    ```bash
    sudo apt install -y software-properties-common
    sudo wget -q -O /usr/share/keyrings/grafana.key https://apt.grafana.com/gpg.key
    echo "deb [signed-by=/usr/share/keyrings/grafana.key] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
    sudo apt update
    sudo apt install grafana -y
    ```
2.  **Start the Service:**
    ```bash
    sudo systemctl enable grafana-server
    sudo systemctl start grafana-server
    ```
3.  **Access:**
    Open `http://192.168.116.10:3000` in your browser. Default login is `admin` / `admin`.

---

### **Step 4: Connecting the Dots**

1.  **Add Data Source:**
    *   In Grafana, go to **Connections > Data Sources**.
    *   Select **Prometheus**.
    *   URL: `http://localhost:9090`.
    *   Click **Save & Test**.

2.  **Import Dashboard:**
    *   Building graphs from scratch is hard. We will import a community standard.
    *   Go to **Dashboards > Import**.
    *   Enter Dashboard ID: `1860` (Node Exporter Full).
    *   Select your Prometheus data source and click **Import**.

### **Step 5: The Stress Test**

A monitoring system is useless if there is nothing to monitor. Let’s generate a synthetic failure signal.

1.  **Install Stress Tool (on `compute-01`):**
    ```bash
    sudo apt install stress-ng -y
    ```
2.  **Run a Stress Job:**
    Submit a job that eats 100% of the CPU:
    ```bash
    sbatch --wrap="stress-ng --cpu 1 --timeout 60"
    ```
3.  **Observe:**
    Look at your Grafana dashboard. You will see the CPU graph spike to 100% (or higher, showing "System Load") for exactly 60 seconds, then drop.

**You have now successfully visualized the heartbeat of your cluster.**
