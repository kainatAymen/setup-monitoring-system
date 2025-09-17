Setup Monitoring System

````markdown
# Setup Monitoring System

This guide explains how to set up **Prometheus**, **Grafana**, **Node Exporter**, and **cAdvisor** on Ubuntu,  
then configure **alerting rules** and **email notifications** via Grafana.

---

## 🖥️ Step 1: One-Shot Installation Script

This single script installs **Prometheus + Node Exporter + Grafana + cAdvisor** in one go.

```bash
#!/bin/bash
set -e

# ===== Update & Install Prerequisites =====
sudo apt update && sudo apt upgrade -y
sudo apt install -y wget tar curl apt-transport-https software-properties-common docker.io

# ===== Prometheus =====
PROM_VERSION="2.55.1"
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v${PROM_VERSION}/prometheus-${PROM_VERSION}.linux-amd64.tar.gz
tar xvf prometheus-${PROM_VERSION}.linux-amd64.tar.gz
sudo mkdir -p /opt/prometheus
sudo cp -r prometheus-${PROM_VERSION}.linux-amd64/* /opt/prometheus/
sudo useradd --no-create-home --shell /bin/false prometheus || true
sudo chown -R prometheus:prometheus /opt/prometheus
(cd /opt/prometheus && nohup ./prometheus --config.file=/opt/prometheus/prometheus.yml > /var/log/prometheus.log 2>&1 &)

# ===== Node Exporter =====
NODE_EXP_VER="1.8.2"
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v${NODE_EXP_VER}/node_exporter-${NODE_EXP_VER}.linux-amd64.tar.gz
tar xvf node_exporter-${NODE_EXP_VER}.linux-amd64.tar.gz
sudo mkdir -p /opt/node_exporter
sudo cp node_exporter-${NODE_EXP_VER}.linux-amd64/node_exporter /opt/node_exporter/
sudo chown root:root /opt/node_exporter/node_exporter
(/opt/node_exporter/node_exporter > /var/log/node_exporter.log 2>&1 &)

# ===== Grafana =====
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/grafana.gpg
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | \
  sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update
sudo apt install -y grafana
sudo systemctl enable --now grafana-server

# ===== cAdvisor (Docker) =====
sudo systemctl enable --now docker
sudo docker run \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --publish=8081:8080 \
  --detach=true \
  --name=cadvisor \
  gcr.io/cadvisor/cadvisor:latest

echo "============================================="
echo " Installation Completed!"
echo " Prometheus:  http://localhost:9090"
echo " NodeExporter: http://localhost:9100/metrics"
echo " Grafana:     http://localhost:3000 (admin / admin)"
echo " cAdvisor:    http://localhost:8081"
echo "============================================="
````

Make it executable and run:

```bash
chmod +x install-monitoring.sh
./install-monitoring.sh
```

---

## Step 2: Access Applications

* **Prometheus** → [http://localhost:9090](http://localhost:9090)
* **Node Exporter** → [http://localhost:9100/metrics](http://localhost:9100/metrics)
* **cAdvisor** → [http://localhost:8081](http://localhost:8081)
* **Grafana** → [http://localhost:3000](http://localhost:3000)  (default login: `admin / admin`)

---

## Step 3: Add Prometheus as a Data Source in Grafana

1. Go to **Grafana → Connections → Data sources**.
2. Choose **Prometheus**.
3. Set URL: `http://localhost:9090` → **Save & Test**.

---

## Step 4: Create Alert Rules in Grafana

1. Go to **Alerting → Alert rules** and create rules, e.g.:

   * **High CPU Usage (>20% for 3 mins)**

     ```
     100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[3m])) * 100) > 20

     ```

<img width="1366" height="654" alt="alert1" src="https://github.com/user-attachments/assets/d9aece66-cb63-411e-9a5c-885fff809a55" />
<img width="984" height="645" alt="alert2" src="https://github.com/user-attachments/assets/270c9e73-c14d-4701-ae66-dd3181c79c68" />
<img width="1128" height="651" alt="alert3" src="https://github.com/user-attachments/assets/84f48505-c7fb-41a0-b65a-463e38bc6ab0" />



   * **Low Memory (<20%)**

     ```
     (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 < 20
     ```

   * **Container Not Running (1 min)**

     ```
     sum(container_last_seen{container_label_com_docker_swarm_service_name!=""}) < 1
     ```

---

## Step 5: Integrate Gmail as Contact Point

1. Go to **Alerting → Contact points** → **New contact point → Email**.
   Enter your Gmail address (e.g., `your_gmail@gmail.com`).

2. Edit Grafana SMTP settings:

   ```ini
   [smtp]
   enabled = true
   host = smtp.gmail.com:587
   user = your_gmail@gmail.com
   password = your_app_password
   from_address = your_gmail@gmail.com
   from_name = Grafana
   ```

   ⚠️ For Gmail you must generate an **App Password**
   (Google Account → Security → App Passwords).

3. Restart Grafana:

   ```bash
   sudo systemctl restart grafana-server
   ```

4. Test email → Alerts should now be sent.

5. 

---<img width="1090" height="646" alt="grafana laert" src="https://github.com/user-attachments/assets/fe62128b-8174-4c5e-80e1-ccfbfab289d4" />


## 🧪 Step 6: Testing Alerts

```bash
# High CPU
sudo apt install stress-ng -y
stress-ng --cpu 4 --timeout 300

# Low Memory
stress-ng --vm 1 --vm-bytes 80% --timeout 300

# Container Not Running
sudo docker stop cadvisor
# Restart later:
sudo docker start cadvisor
```

---

purpose for using following tools:



---

### 🔹 1. Kibana (ELK Stack)

* **Use**: Logs ko **search**, **analyze** aur **visualize** karne ke liye.
* **Stack**: Elasticsearch + Logstash + Kibana (kabhi Beats bhi).
* **Example**: Nginx, Apache, ya app logs ko collect karke Elasticsearch me store karte ho, phir Kibana se search aur graphs banate ho.
* **Commands jo tumne likhe**:

  ```bash
  sudo apt install kibana -y
  sudo systemctl enable kibana
  sudo systemctl start kibana
  ```

  Ye Kibana ko install aur service start karne ke liye hain.

---

### 🔹 2. Prometheus Ecosystem

Metrics based monitoring ke liye use hota hai (CPU, RAM, disk, container stats).

**a) Node Exporter**

* Server ke hardware/OS level metrics deta hai: CPU load, memory usage, disk I/O, network stats, etc.
* Prometheus is data ko scrape karta hai.

**b) cAdvisor (Container Advisor)**

* Docker containers ka CPU, memory, network, filesystem stats deta hai.
* Real-time container monitoring.

**c) Grafana**

* Visualization tool hai.
* Prometheus ya dusre data sources se data lekar dashboards aur graphs banata hai.
* Alerts bhi configure kar sakte ho.

---

### 📊 Quick Table

| Tool              | Type              | Main Use                        |
| ----------------- | ----------------- | ------------------------------- |
| **Kibana**        | Log visualization | Logs search & analysis (ELK)    |
| **Prometheus**    | Metrics DB        | Time-series metrics collection  |
| **Node Exporter** | Exporter          | Server hardware/OS metrics      |
| **cAdvisor**      | Exporter          | Docker container metrics        |
| **Grafana**       | Visualization     | Dashboards & alerts for metrics |

---

👉 **Simple Guide**

* **Logs check** karne hain → **ELK stack (Kibana)**
* **Real-time CPU/RAM/Container performance** chahiye → **Prometheus + Grafana + Node Exporter + cAdvisor**

Dono ko saath bhi use kar sakte ho:
Logs ke liye ELK aur performance metrics ke liye Prometheus + Grafana.


✅ At this point you will have:

* **Prometheus, Grafana, Node Exporter & cAdvisor installed**
* **Prometheus integrated with Grafana**
* **Alert rules for CPU, memory, and container state**
* **Email alerts sent to Gmail**

---

```


GitHub will render the headings, lists, and code blocks properly.
```
