# 🔒 Lab 13: Kubernetes Security Hardening (NetworkPolicies) & HPA

## 🧠 Concept & Architecture ("What & Why")

### What is a Kubernetes NetworkPolicy?
By default in Kubernetes, pod networking is an open flat network model: **every pod can communicate with every other pod across all namespaces**. In production enterprise environments, this presents a severe security vulnerability. If an attacker compromises a single public-facing pod, they can pivot laterally across the cluster network to access database pods or internal APIs.

A **NetworkPolicy** acts as an in-cluster zero-trust firewall operating at Layers 3 and 4 (IP address and TCP/UDP port). It enforces packet filtering directly on pod virtual network interfaces (veth) using the underlying Container Network Interface (CNI) plugin (e.g. AWS VPC CNI / Calico).

### What is HorizontalPodAutoscaler (HPA)?
The **HorizontalPodAutoscaler (HPA)** automatically scales the number of running pod replicas in a `Deployment`, `StatefulSet`, or `ReplicaSet` up or down based on real-time resource metrics (CPU and Memory utilization) collected from worker node Kubelets by the **Metrics Server**.

```text
                                ZERO-TRUST POD SECURITY & AUTOSCALING
                                
  [ Public Internet ]
           |
           | HTTP / Port 80
           v
  +-----------------------------------------------------------------------------------+
  |  AWS Application Load Balancer (ALB Ingress)                                     |
  +----------------------------------------+------------------------------------------+
                                           |
                                           | ALLOW Ingress Port 8000
                                           v
  +-----------------------------------------------------------------------------------+
  |  KUBERNETES NAMESPACE: nti-devops                                                 |
  |                                                                                   |
  |  +-----------------------------------------------------------------------------+  |
  |  |  DJANGO APPLICATION PODS (Deployment: 2 to 6 Replicas via HPA)              |  |
  |  |  +------------------------+             +------------------------+          |  |
  |  |  | Pod 1 (IP: 10.0.1.42)   |             | Pod 2 (IP: 10.0.2.89)   |          |  |
  |  |  +------------------------+             +------------------------+          |  |
  |  +-------------------------------------+---------------------------------------+  |
  |                                        |                                          |
  |                                        | ALLOW Egress Port 5432                   |
  |                                        v                                          |
  |  +-----------------------------------------------------------------------------+  |
  |  |  DATABASE POD / AWS RDS POSTGRESQL (Port 5432)                              |  |
  |  +-----------------------------------------------------------------------------+  |
  |                                        ^                                          |
  |                                        | BLOCKED by NetworkPolicy                 |
  |  +-------------------------------------+---------------------------------------+  |
  |  |  UNAUTHORIZED / COMPROMISED POD (In any Namespace)                           |  |
  |  +-----------------------------------------------------------------------------+  |
  +-----------------------------------------------------------------------------------+
```

---

## 🎯 Objectives

- Implement zero-trust network isolation using Kubernetes `NetworkPolicy` (`networkpolicy.yaml`).
- Enforce strict default-deny rules while selectively allowing ingress traffic on Port 8000 and egress database connections on Port 5432.
- Configure `HorizontalPodAutoscaler` (`horizontalpodautoscaler.yaml`) to automatically scale pod replicas between **2 and 6** when CPU or Memory utilization exceeds **80%**.
- Perform a simulated load test to verify real-time HPA pod scaling.

---

## 📂 Project Structure

```text
04-Kubernetes-Orchestration/Lab13-K8s-Security-Autoscaling/
├── horizontalpodautoscaler.yaml # Dynamic HPA scaling rules (2 to 6 pods @ 80% CPU/RAM)
├── networkpolicy.yaml           # Zero-trust network security firewall policy
└── README.md                    # Exhaustive step-by-step lab documentation
```

---

## 🛠 Comprehensive Code & Configuration Breakdown

### 1. Zero-Trust Network Security Policy (`networkpolicy.yaml`)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: nti-django-network-policy
  namespace: nti-devops
  labels:
    app.kubernetes.io/name: django-app
    app.kubernetes.io/part-of: nti-devops-platform
spec:
  podSelector:
    matchLabels:
      app: nti-django-app
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: nti-devops
        - podSelector: {}
      ports:
        - protocol: TCP
          port: 8000
  egress:
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
      ports:
        - protocol: TCP
          port: 5432
        - protocol: TCP
          port: 6379
        - protocol: UDP
          port: 53
```

#### Line-by-Line Breakdown:
- **`apiVersion: networking.k8s.io/v1`**: Specifies the standard Kubernetes Networking API group.
- **`kind: NetworkPolicy`**: Defines object type as a NetworkPolicy firewall rule.
- **`metadata.namespace: nti-devops`**: Restricts policy scope to the `nti-devops` workload namespace.
- **`podSelector.matchLabels.app: nti-django-app`**: Selects target pods bearing label `app: nti-django-app`. Any pod matching this selector will have network filtering applied.
- **`policyTypes: [Ingress, Egress]`**: Applies isolation rules for BOTH incoming (Ingress) and outgoing (Egress) network traffic.
- **`ingress.from`**: Restricts incoming traffic. Traffic is accepted ONLY from pods in the `nti-devops` namespace on TCP Port 8000.
- **`egress.to`**: Restricts outgoing traffic. Allows Django pods to make outbound TCP connections on Port 5432 (PostgreSQL RDS), Port 6379 (Redis Cache), and UDP Port 53 (CoreDNS name resolution). All other outbound traffic (such as malicious reverse shells) is blocked.

---

### 2. HorizontalPodAutoscaler Configuration (`horizontalpodautoscaler.yaml`)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nti-django-hpa
  namespace: nti-devops
  labels:
    app.kubernetes.io/name: django-app
    app.kubernetes.io/part-of: nti-devops-platform
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nti-django-app
  minReplicas: 2
  maxReplicas: 6
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 80
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
```

#### Line-by-Line Breakdown:
- **`apiVersion: autoscaling/v2`**: Uses Autoscaling API v2 supporting multi-metric scaling (CPU and Memory).
- **`scaleTargetRef`**: Points HPA directly to target Deployment `nti-django-app`.
- **`minReplicas: 2`**: Guarantees a minimum baseline of 2 running pods for high availability across multi-AZ EKS worker nodes.
- **`maxReplicas: 6`**: Sets maximum burst limit to 6 pods during heavy traffic spikes.
- **`averageUtilization: 80`**: Triggers scaling when average CPU or Memory consumption across all pods exceeds 80% of requested limits.
- **`behavior.scaleUp`**: Immediate scale-up policy (0s stabilization window) allowing 100% capacity doubling every 15 seconds.
- **`behavior.scaleDown`**: Conservative 300s (5-minute) cooldown window to prevent rapid "flapping" during fluctuating traffic.

---

## 📋 Step-by-Step Execution Guide

### Step 1: Connect to EKS Cluster & Change Directory
```bash
aws eks update-kubeconfig --name nti-devops-eks --region us-east-1
cd 04-Kubernetes-Orchestration/Lab13-K8s-Security-Autoscaling
```

### Step 2: Apply NetworkPolicy & HPA Manifests
```bash
kubectl apply -f networkpolicy.yaml
kubectl apply -f horizontalpodautoscaler.yaml
```

### Step 3: Monitor Live HPA Metrics
```bash
kubectl get hpa -n nti-devops -w
```

### Step 4: Simulate Heavy CPU Load to Test HPA Scaling
In a separate terminal, launch a temporary generator pod to send concurrent HTTP requests:
```bash
kubectl run -i --tty load-generator --rm --image=busybox --namespace=nti-devops -- /bin/sh -c "while true; do wget -q -O- http://nti-django-svc:8000/; done"
```

---

## 🧪 Verification & Expected Terminal Output

### 1. Verify NetworkPolicy Status
```bash
kubectl get networkpolicy -n nti-devops
```
**Expected Output:**
```text
NAME                        POD-SELECTOR          AGE
nti-django-network-policy   app=nti-django-app    1m
```

### 2. Verify HPA Auto-Scaling Response Under Load
```bash
kubectl get hpa -n nti-devops
```
**Expected Output:**
```text
NAME             REFERENCE                   TARGETS           MINPODS   MAXPODS   REPLICAS   AGE
nti-django-hpa   Deployment/nti-django-app   145%/80%, 60%/80%   2         6         5          3m
```

### 3. Verify Deployment Pod Count Scaling
```bash
kubectl get pods -n nti-devops -l app=nti-django-app
```
**Expected Output:**
```text
NAME                              READY   STATUS    RESTARTS   AGE
nti-django-app-6d8b9f7c4d-a1b2c   1/1     Running   0          4m
nti-django-app-6d8b9f7c4d-d5e6f   1/1     Running   0          4m
nti-django-app-6d8b9f7c4d-g7h8i   1/1     Running   0          90s
nti-django-app-6d8b9f7c4d-j9k0l   1/1     Running   0          90s
nti-django-app-6d8b9f7c4d-m1n2o   1/1     Running   0          45s
```

---

## 💡 Best Practices & Common Troubleshooting

### Enterprise Best Practices:
1. **Always Install Metrics Server First**: HPA requires Metrics Server installed in `kube-system` namespace.
2. **Explicit Resource Requests**: HPA cannot calculate utilization percentage unless `resources.requests.cpu` and `resources.requests.memory` are defined on container templates.

### Troubleshooting Matrix:

| Error Symptom | Root Cause | Remediation Command |
|---------------|------------|---------------------|
| `HPA TARGETS shows <unknown>/80%` | Metrics Server is missing or pods lack resource requests | Deploy Metrics Server (`helm install metrics-server`) and define `resources.requests` in `deployment.yaml`. |
| `NetworkPolicy not blocking unauthorized pods` | CNI plugin does not support NetworkPolicies | Ensure AWS VPC CNI or Calico policy engine is enabled on the EKS cluster. |

---

## 🔗 Connection to Next Lab
In **Lab 14**, we parameterize and package all raw Kubernetes manifests into a clean, reusable Helm Chart (`myapp`)!
