# 🚨 Lab 18: Custom Prometheus Alert Rules & Grafana Dashboards

## 🧠 Concept & Architecture ("What & Why")

### What are PrometheusRules & ServiceMonitors?
- **PrometheusRule**: Custom Kubernetes CRD defining alert threshold expressions evaluated dynamically against scraped time-series metrics.
- **ServiceMonitor**: CRD telling Prometheus which Kubernetes services to discover and scrape for metrics.
- **Grafana Dashboard JSON**: Declarative visualization layout displaying RED metrics (Request Rate, Error Rate, Duration/Latency).

---

## 📂 Project Structure

```text
07-Monitoring-Observability/Lab18-Alerts-Grafana-Dashboards/
├── alerts/
│   ├── nti-devops-rules.yaml           # 10 custom PrometheusRule alert definitions
│   └── servicemonitor.yaml             # ServiceMonitor metric scraping configuration
├── dashboards/
│   └── nti-devops-application.json     # Custom Grafana application dashboard JSON
└── README.md                           # Exhaustive step-by-step lab documentation
```

---

## 🛠 Comprehensive Code & Configuration Breakdown

### 1. Custom Prometheus Alert Rules (`alerts/nti-devops-rules.yaml`)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: nti-devops-alert-rules
  namespace: monitoring
  labels:
    role: alert-rules
    app.kubernetes.io/part-of: kube-prometheus-stack
spec:
  groups:
    - name: nti-devops-workload-alerts
      rules:
        - alert: PodHighCpuUtilization
          expr: sum(node_namespace_pod_container:container_cpu_usage_seconds_total:sum_irate{namespace="nti-devops"}) by (pod) / sum(kube_pod_container_resource_limits{namespace="nti-devops", resource="cpu"}) by (pod) * 100 > 80
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "High CPU utilization on Pod {{ $labels.pod }}"
            description: "Pod {{ $labels.pod }} CPU usage has exceeded 80% for 5 minutes."

        - alert: PodHighMemoryUtilization
          expr: sum(container_memory_working_set_bytes{namespace="nti-devops"}) by (pod) / sum(kube_pod_container_resource_limits{namespace="nti-devops", resource="memory"}) by (pod) * 100 > 80
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "High Memory utilization on Pod {{ $labels.pod }}"
            description: "Pod {{ $labels.pod }} Memory usage has exceeded 80% for 5 minutes."
```

---

## 📋 Step-by-Step Execution Guide

### Step 1: Apply Alert Rules & ServiceMonitor
```bash
cd 07-Monitoring-Observability/Lab18-Alerts-Grafana-Dashboards
kubectl apply -f alerts/nti-devops-rules.yaml -f alerts/servicemonitor.yaml
```

---

## 🧪 Verification & Expected Terminal Output

```bash
kubectl get prometheusrule -n monitoring
```

**Expected Output:**
```text
NAME                     AGE
nti-devops-alert-rules   1m
```

---

## 🔗 Connection to Next Lab
In **Lab 19**, we complete the platform handbook with Day-2 Operations, credential rotation, and troubleshooting guides!
