# Virtualisation & Cloud Computing | GCP Lab Assignments

![GCP](https://img.shields.io/badge/Google%20Cloud-Compute%20Engine%20%7C%20MIG%20%7C%20IAM-4285F4?logo=googlecloud&logoColor=white)
![VirtualBox](https://img.shields.io/badge/VirtualBox-Ubuntu%2022.04-183A61?logo=virtualbox&logoColor=white)
![Prometheus](https://img.shields.io/badge/Monitoring-Prometheus%20%7C%20Grafana-E6522C?logo=prometheus&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-containerised%20monitoring-2496ED?logo=docker&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-lightgrey)

Three hands-on cloud infrastructure assignments from **CSL7510 — Virtualisation and Cloud Computing** (IIT Jodhpur, Sem II 2024–25, Dr Sumit Kalra). Covers VM orchestration, GCP auto-scaling, IAM security, and CPU-triggered cloud bursting with Prometheus + Grafana monitoring.

---

## Assignment Overview

| # | Title | Platform | Key Concepts |
|---|---|---|---|
| 1 | VirtualBox Microservice Deployment | VirtualBox + Flask | Multi-VM networking, SSH, microservices |
| 2 | GCP VM + Auto-Scaling + Security | GCP Console | Compute Engine, MIG, IAM, VPC Firewall |
| 3 | Auto-Scaling Local VM to GCP | GCP CLI + Prometheus + Grafana | Cloud bursting, monitoring, autoscaler |

---

## Assignment 1 — VirtualBox Multi-VM Microservice

**Task:** Create multiple VMs in VirtualBox, configure inter-VM networking, and deploy a Flask microservice across them.

### Setup

- VM Master (Ubuntu 22.04, 2.5 GB RAM, 2 CPUs, 30 GB) — runs Flask app
- VM Node1 (Ubuntu 22.04, 1.7 GB RAM, 2 CPUs, 30 GB) — SSH client

### Steps

```bash
# Change network adapter: NAT to Bridged Adapter (VirtualBox Settings)
# Assign unique IPs to each VM
dhclient -r && dhclient

# Enable SSH on both VMs
sudo apt install openssh-server -y
sudo systemctl enable ssh && sudo systemctl start ssh

# Verify connectivity
ping 172.20.10.10   # VM Node1 IP
```

### Flask Microservice (News Aggregator)

Deployed on VM Master (IP: 172.20.10.9), accessible from VM Node1 via SSH:

```python
from flask import Flask, jsonify
import requests, os

app = Flask(__name__)

@app.route('/')
def get_news():
    api_key = os.getenv("NEWS_API_KEY")
    url = f'https://newsapi.org/v2/top-headlines?country=us&apiKey={api_key}'
    return jsonify(requests.get(url).json())

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

```bash
# Run on VM Master
python3 fetch.py

# Access from VM Node1
ssh vboxuser@172.20.10.9
curl http://localhost:5000/news
```

### Architecture

```
+----------------------+          +----------------------+
|   VM Master          |          |   VM Node1           |
|   Ubuntu 22.04       | <--SSH-- |   Ubuntu 22.04       |
|   Flask App          |          |   SSH Client         |
|   IP: 172.20.10.9    |          |   IP: 172.20.10.10   |
+----------------------+          +----------------------+
           |                                  |
           +--------- Bridged Network --------+
                   Oracle VirtualBox Host
```

Result: Live news headlines successfully fetched from VM Node1 via Flask running on VM Master.

---

## Assignment 2 — GCP VM + Auto-Scaling + Security

**Task:** Deploy a VM on GCP, configure CPU-based auto-scaling with Managed Instance Groups, and secure with IAM and VPC Firewall rules.

### VM Configuration

| Setting | Value |
|---|---|
| Machine type | E2 General Purpose (4 GB RAM) |
| OS | Windows Server |
| Region | asia-south1-c |
| Access | Remote Desktop (RDP via external IP) |

### Auto-Scaling (Managed Instance Group)

```
Compute Engine → Instance Templates → Create Template
→ Instance Groups → Create Managed Instance Group (MIG)
→ Autoscaling: CPU utilisation > 60% triggers scale-up
→ Min instances: 2 · Max instances: 5
```

### IAM Roles

| Role | Permission |
|---|---|
| Compute Viewer | Read-only access to all Compute resources |
| Compute Admin | Full administrative control |
| Custom Role | Scoped per-resource permissions |

### VPC Firewall Rules

| Rule | Type | Ports | Purpose |
|---|---|---|---|
| allow-ssh | Ingress | TCP: 22 | SSH access |
| allow-http | Ingress | TCP: 80 | HTTP traffic |
| deny-all-egress | Egress | All | Restrict outbound |

Result: VM deployed, MIG auto-scaling verified, IAM roles and firewall rules applied successfully.

---

## Assignment 3 — Auto-Scaling Local VM to GCP (Cloud Bursting)

**Task:** Monitor a local VM with Prometheus + Grafana. When CPU exceeds 75%, automatically trigger scaling to GCP using Managed Instance Groups.

### Architecture

```
Local VM (Ubuntu 22.04, VirtualBox)
        |
        +-- Prometheus (Docker, :9090) -- scrapes CPU metrics
        +-- Grafana    (Docker, :3000) -- visualises metrics
        |
        CPU load exceeds 75%
                |
                v
        GCP Auto-Scaling Trigger
                |
        Managed Instance Group (monitoring-mig)
        Region: asia-south1 · Max replicas: 5
```

### Step 1 — Monitoring Stack (Docker)

```bash
# Create monitoring network
docker network create monitoring

# Run Prometheus
docker run -d --name prometheus \
  --network monitoring \
  -p 9090:9090 \
  -v /etc/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml \
  prom/prometheus

# Run Grafana
docker run -d --name=grafana \
  --network monitoring \
  -p 3000:3000 grafana/grafana
```

Add Prometheus as Grafana data source: `Configuration → Data Sources → Prometheus → http://prometheus:9090`

### Step 2 — GCP Infrastructure (gcloud CLI)

```bash
# Create instance template
gcloud compute instance-templates create monitoring-template \
  --machine-type=e2-medium \
  --image-family=debian-11 \
  --image-project=debian-cloud \
  --boot-disk-size=20GB \
  --tags=monitoring-vm

# Create Managed Instance Group
gcloud compute instance-groups managed create monitoring-mig \
  --base-instance-name monitoring-instance \
  --template monitoring-template \
  --size 1 \
  --region asia-south1

# Define auto-scaling policy (CPU > 75%)
gcloud compute instance-groups managed set-autoscaling monitoring-mig \
  --region asia-south1 \
  --max-num-replicas 5 \
  --target-cpu-utilization 0.75
```

### Step 3 — Deploy Flask App + Expose via Firewall

```bash
pip install flask
python app.py

# Expose port 80 to MIG instances
gcloud compute firewall-rules create allow-http \
  --allow tcp:80 \
  --target-tags monitoring-vm
```

### Step 4 — Load Test + Verification

```bash
# Simulate high CPU load
stress --cpu 4 --timeout 300

# Verify autoscaler
gcloud compute instance-groups managed describe monitoring-mig \
  --region asia-south1
# autoscaler: enabled · target utilization: 75%
```

After 2-3 minutes: GCP Console → Compute Engine → Instance Groups → instance count increases automatically.

### Monitoring Stack

| Tool | Role | Port |
|---|---|---|
| Prometheus | Metrics scraping + storage | 9090 |
| Grafana | Dashboard + visualisation | 3000 |
| Node Exporter | CPU/memory/disk metrics | 9100 |

---

## Tech Stack

| Component | Technology |
|---|---|
| Cloud Platform | Google Cloud Platform (GCP) |
| VM Orchestration | Oracle VirtualBox, GCP Compute Engine |
| Auto-Scaling | GCP Managed Instance Groups (MIG) |
| Monitoring | Prometheus + Grafana (Docker) |
| Security | GCP IAM, VPC Firewall Rules |
| CLI | gcloud SDK |
| Backend | Flask (Python) |
| OS | Ubuntu 22.04, Windows Server |

---

## Repository Structure

```
VCC_Assignments/
+-- Assignment1_VirtualBox_Microservice.pdf       # Multi-VM setup + Flask microservice
+-- Assignment2_GCP_AutoScaling_Security.pdf      # GCP Compute Engine + MIG + IAM + Firewall
+-- Assignment3_AutoScaling_LocalVM_to_GCP.pdf    # Prometheus + Grafana + GCP cloud bursting
+-- README.md
```

---

## Course Details

**Course:** CSL7510 — Virtualisation and Cloud Computing
**Instructor:** Dr Sumit Kalra, IIT Jodhpur
**Semester:** II (2024-25)
**Topics covered:** Hypervisors · Full/Para-Virtualisation · GCP Compute · MIG · IAM · VPC · Docker · Prometheus · Grafana · Cloud Bursting

---

## Author

**Kakarla Sai Swaroop** M25DE1023, IIT Jodhpur M.Tech Data Engineering
