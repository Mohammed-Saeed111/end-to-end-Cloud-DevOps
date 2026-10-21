# 📊 Lab 17: Deploying Kube-Prometheus-Stack on AWS EKS

## 🧠 Concept & Architecture ("What & Why")

### What is Kube-Prometheus-Stack?
**Kube-Prometheus-Stack** is a comprehensive, production-ready Kubernetes observability suite. It packages:
- **Prometheus Operator**: Manages Prometheus and Alertmanager instances using custom Kubernetes CRDs (`ServiceMonitor`, `PrometheusRule`).
- **Prometheus**: High-performance time-series database scraping metrics from nodes, pods, and services via Kubernetes API discovery.
- **Alertmanager**: Deduplicates, groups, and routes alert notifications.
- **Grafana**: Real-time metrics visualization platform.
- **kube-state-metrics & node-exporter**: Generates host-level and cluster object metrics.

```text
                                OBSERVALITY & MONITORING STACK
                                
  +-----------------------------------------------------------------------------------+
  |  AWS EKS WORKER NODES & PODS                                                      |
  |  [ Pod 1 Metrics ]        [ Pod 2 Metrics ]        [ Kubelet Node Exporter ]      |
  +-----------------------------------+-----------------------------------------------+
                                      |
                                      | Scrapes Metrics via ServiceMonitor (30s)
                                      v
  +-----------------------------------------------------------------------------------+
  |  PROMETHEUS TIME-SERIES DATABASE (prometheus-0)                                  |
  |  Stores CPU, Memory, Network, Disk & Custom App Metrics                           |
  +-----------------------------------+-----------------------------------------------+
                                      |
                                      | PromQL Queries
                                      v
  +-----------------------------------------------------------------------------------+
  |  GRAFANA DASHBOARDS (Port 3000)                                                   |
  |  Real-time RED Metrics & Infrastructure Dashboards                                |
  +-----------------------------------------------------------------------------------+
```

---

## 🎯 Objectives

- Deploy `kube-prometheus-stack` on EKS using Helm and custom values (`kube-prometheus-stack-values.yaml`).
- Configure automated monitoring installation script (`install/install.sh`).
- Access Grafana visualization web UI.

---

## 📂 Project Structure

```text
07-Monitoring-Observability/Lab17-Prometheus-Grafana-Monitoring-Stack/
├── install/
│   ├── dashboards-configmap.yaml               # Auto-provisioned Grafana dashboard ConfigMap
│   └── install.sh                             # Automated monitoring stack installer script
├── kube-prometheus-stack-values.yaml          # Helm values override specification
└── README.md                                   # Exhaustive step-by-step lab documentation
```

---

## 🛠 Comprehensive Code & Configuration Breakdown

### 1. Automated Monitoring Installer Script (`install/install.sh`)

```bash
#!/bin/bash
set -e

echo "🚀 Deploying Kube-Prometheus-Stack on EKS..."

kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install prometheus prometheus-community/kube-prometheus-stack   -n monitoring   -f kube-prometheus-stack-values.yaml

kubectl apply -f install/dashboards-configmap.yaml -n monitoring

echo "✅ Kube-Prometheus-Stack deployed successfully!"
```

---

## 📋 Step-by-Step Execution Guide

### Step 1: Execute Automated Monitoring Deployment Script
```bash
cd 07-Monitoring-Observability/Lab17-Prometheus-Grafana-Monitoring-Stack
chmod +x install/install.sh
./install/install.sh
```

### Step 2: Access Grafana Web UI
```bash
kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring
```
Open web browser to `http://localhost:3000` (Default Credentials: `admin` / `prom-operator`).

---

## 🧪 Verification & Expected Terminal Output

```bash
kubectl get pods -n monitoring
```

**Expected Output:**
```text
NAME                                                        READY   STATUS    RESTARTS   AGE
prometheus-kube-prometheus-prometheus-0                     2/2     Running   0          2m
prometheus-grafana-7d8b9f7c4d-x9y8z                        1/1     Running   0          2m
prometheus-kube-state-metrics-6d8b9f7c4d-v1w2u             1/1     Running   0          2m
```

---

## 🔗 Connection to Next Lab
In **Lab 18**, we configure 10 custom Prometheus alert rules (>80% CPU/RAM alarms) and import custom Grafana dashboards!
